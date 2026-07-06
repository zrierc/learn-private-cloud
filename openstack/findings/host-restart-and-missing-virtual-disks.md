# What happened — Root Cause

This is a **race condition on VM reboot**. Here's the sequence:

```
VMs shut down
      ↓
/dev/vdb detached or unavailable during boot
      ↓
VMs boot up again
      ↓
Docker starts cinder_volume container early
      ↓
cinder_volume LVM driver tries to initialize:
  "find cinder-volumes VG on /dev/vdb"
      ↓
/dev/vdb not ready yet OR not attached → LVM init fails
      ↓
Driver stays uninitialized indefinitely:
  "config name lvm-1 is uninitialized"
  "not sending heartbeat, service will appear down"
      ↓
cinder-volume service shows DOWN in OpenStack
      ↓
Any volume creation → error state immediately
(scheduler can't find a healthy backend to create on)
```

The container itself shows `healthy` (Docker health check passes) but the internal
LVM driver is stuck uninitialized — so Docker doesn't know it's broken.

---

## How to detect this issue

| Symptom                                        | What it means                              |
| ---------------------------------------------- | ------------------------------------------ |
| Volume status `error` immediately after create | No healthy cinder-volume backend available |
| `cinder-volume` shows `down` in service list   | Backend not sending heartbeat              |
| Container shows `healthy` in `docker ps`       | Misleading — Docker-level health is fine   |
| Log says `lvm-1 is uninitialized`              | LVM driver failed to initialize at startup |

---

## Step-by-step fix for next time

**Step 1 — Confirm the issue:**

```bash
openstack volume service list
# Look for cinder-volume showing "down"
```

**Step 2 — Check LVM is accessible on the host:**

```bash
sudo vgs
sudo lvs
# Should show cinder-volumes VG on /dev/vdb
# If empty → /dev/vdb not attached yet, attach it first before continuing
```

**Step 3 — Check the actual error inside cinder_volume:**

```bash
sudo docker exec cinder_volume cat /var/log/kolla/cinder/cinder-volume.log | tail -30
# Look for: "lvm-1 is uninitialized" or similar
```

**Step 4 — Restart cinder containers on all nodes:**

```bash
# Controller
sudo docker restart cinder_volume cinder_scheduler cinder_backup

# Compute nodes
ssh ubuntu@pod-username-compute1 "sudo docker restart cinder_volume cinder_backup"
ssh ubuntu@pod-username-compute2 "sudo docker restart cinder_volume cinder_backup"
```

**Step 5 — Wait 30-60 seconds then verify:**

```bash
openstack volume service list
# All cinder-volume should show "up"
```

**Step 6 — Fix any errored volumes:**

```bash
# Option A: Reset state (faster)
openstack volume set --state available <volume-name>

# Option B: Delete and recreate (cleaner)
openstack volume delete <volume-name>
openstack volume create --size <size-gb> <volume-name>
```

---

## How to prevent this next time

After starting your VMs, before doing anything in OpenStack, always run this
quick sanity check:

```bash
# Verify /dev/vdb is attached
lsblk | grep vdb

# Verify LVM VG is visible
sudo vgs

# Verify all cinder services are up
openstack volume service list

# Verify all OpenStack services are up generally
openstack compute service list
openstack network agent list
```

Only proceed with volume operations once all services show `up`. If `cinder-volume` is still `down` after a minute or two, run the restart commands in Step 4 above.
