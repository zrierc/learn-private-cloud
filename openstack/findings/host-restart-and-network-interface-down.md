# What happened — Root Cause

Three separate issues combined to break SSH after the VM reboot:

## Issue 1 — br-ex state resets after reboot

```
VM reboots
      ↓
br-ex comes back as state DOWN, no IP
      ↓
No path from controller main namespace
into qrouter namespace
      ↓
Ping and SSH to floating IP fail
```

## Issue 2 — qrouter default route resets after reboot

```
VM reboots
      ↓
qrouter default route resets to:
  "via 20.168.168.1" (fictitious gateway)
      ↓
Packets from qrouter namespace can't
exit to internet or reach controller
      ↓
Even if br-ex is fixed, internet still broken
```

## Issue 3 — OpenSSH 9.x vs CirrOS dropbear compatibility

```
Modern OpenSSH 9.x deprioritizes ssh-rsa key type
      ↓
CirrOS dropbear only supports ssh-rsa
      ↓
Key negotiation silently falls back to password auth
      ↓
Looks like keypair auth is broken when it's actually
just a client-side algorithm negotiation issue
```

---

## How to detect this issue

| Symptom                                        | Likely cause                          |
| ---------------------------------------------- | ------------------------------------- |
| SSH hangs or immediately asks password         | br-ex DOWN or no IP                   |
| `gocubsgo` password also rejected over SSH     | br-ex/qrouter routing broken          |
| VNC console works fine with `gocubsgo`         | VM is healthy, network is the problem |
| `ip addr show br-ex` shows `state DOWN`        | Issue 1 confirmed                     |
| qrouter default route shows `via 20.168.168.1` | Issue 2 confirmed                     |
| SSH works via VNC but not via keypair          | Issue 3 confirmed                     |

---

## Step-by-step fix for next time

**Step 1 — Confirm br-ex is DOWN:**

```bash
ip addr show br-ex
# Look for: state DOWN and no inet address
```

**Step 2 — Check qrouter default route:**

```bash
sudo ip netns exec qrouter-<router-uuid> ip route show
# Look for: default via 20.168.168.1 (fictitious = broken)
# Should be: default via 20.168.168.2 (br-ex IP = correct)
```

**Step 3 — Fix br-ex:**

```bash
sudo ip link set br-ex up
sudo ip addr add <external-prefix>.2/24 dev br-ex
sudo ip route add <external-prefix>.0/24 dev br-ex
# "RTNETLINK answers: File exists" on route = already exists, ignore it
```

**Step 4 — Restore NAT rules:**

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -s <internal-prefix>.0/24 -o <mgmt-nic> -j MASQUERADE
sudo iptables -t nat -A POSTROUTING -s <external-prefix>.0/24 -o <mgmt-nic> -j MASQUERADE
sudo iptables -A FORWARD -i br-ex -o <mgmt-nic> -j ACCEPT
sudo iptables -A FORWARD -i <mgmt-nic> -o br-ex -j ACCEPT
```

**Step 5 — Fix qrouter default route:**

```bash
# Get current qrouter UUID and qg interface
sudo ip netns list                                         # get qrouter-<uuid>
sudo ip netns exec qrouter-<uuid> ip addr show            # get qg-<interface-id>

# Replace fictitious gateway with br-ex IP
sudo ip netns exec qrouter-<uuid> \
  ip route replace default via <external-prefix>.2 dev qg-<interface-id>
```

**Step 6 — Clear stale known_hosts entry:**

```bash
ssh-keygen -f '/home/<user>/.ssh/known_hosts' -R '<floating-ip>'
```

**Step 7 — SSH with correct flags for CirrOS:**

```bash
# Always use this flag when SSHing into CirrOS
ssh -i ~/.ssh/id_rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa cirros@<floating-ip>
```

---

## Permanent fix — add to rc.local

Since Steps 3, 4, and 5 reset on every reboot, add them to `/etc/rc.local` as documented in Section 6 of your journal. That way you only need to worry about Steps 6 and 7 after a reboot.

For Step 5 specifically, use the dynamic script from your journal that auto-detects the qrouter UUID:

```bash
QROUTER=$(sudo ip netns list | grep qrouter | awk '{print $1}')
QG=$(sudo ip netns exec $QROUTER ip addr show | grep qg- | awk '{print $NF}' | head -1)
sudo ip netns exec $QROUTER ip route replace default via <external-prefix>.2 dev $QG
```

Add this to `rc.local` and you'll never have to manually fix the qrouter route again after reboot.
