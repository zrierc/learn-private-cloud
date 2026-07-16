# Kube API Server Crash Unexpectedly

- **Date:** July 16th, 2026

## Overview

> [!IMPORTANT]
> `k` abbreviation of `kubectl`. It's just alias.

This morning I got issue where kube cannot communicate across nodes. Network was
fine tho. Ping success. SSH success. But I can't do something like:

```sh
k get nodes
# or
k get pods
```

it shows:

```log
E0716 10:04:08.420706  644425 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"https://10.1.1.10:6443/api?timeout=32s\": dial tcp 10.1.1.10:6443: connect: connection refused"
E0716 10:04:08.426509  644425 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"https://10.1.1.10:6443/api?timeout=32s\": dial tcp 10.1.1.10:6443: connect: connection refused"
E0716 10:04:08.430237  644425 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"https://10.1.1.10:6443/api?timeout=32s\": dial tcp 10.1.1.10:6443: connect: connection refused"
```

Check the listening port with `sudo lsof -i -P -n` none of port 6443 appear (port
that being used by kube API Server). Kube proxy was fine, scheduler, kubelet, are
also fine. Everything else was fine.

### Assumption

I thought it was network issue at first, then I think about the cert expiration.
But **it's NOT**. I just deploy it like 1 week ago. Cert active like 1 years.

---

## Debugging Steps

### Step 1 — Confirm it's really the apiserver

```bash
ss -tlnp | grep 6443
```

- **Port exists** → apiserver is up, problem is elsewhere (kubeconfig, network, auth)
- **Port missing** → apiserver is down, continue to Step 2

### Step 2 — Check if the container exists and its state

```bash
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps -a | grep apiserver
```

- **Running** but port missing → it's starting up, wait or check logs
- **Exited** → it crashed, check logs
- **Not listed at all** → kubelet isn't even trying, check manifest and kubelet

### Step 3 — Check why it's crashing

```bash
# Get the log directory
APISERVERDIR=$(sudo ls /var/log/pods/ | grep apiserver | head -1)
LOGFILE=$(sudo ls -t /var/log/pods/$APISERVERDIR/kube-apiserver/ | head -1)
sudo tail -50 /var/log/pods/$APISERVERDIR/kube-apiserver/$LOGFILE
```

The **last few lines before it dies**, it tell the real error.

### Step 4 — Check its dependencies (etcd)

```bash
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock exec -it \
  $(sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps | grep etcd | awk '{print $1}') \
  etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health
```

- **healthy** → etcd is fine, problem is in apiserver itself
- **unhealthy/timeout** → etcd is the root cause, fix etcd first

### Step 5 — Check system resources

```bash
# CPU and memory pressure
top -bn1 | head -20

# Disk space (etcd dies if / fills up)
df -h

# Processes stuck in uninterruptible sleep (hung I/O)
ps aux | grep " D "

# Kernel-level warnings
sudo dmesg | tail -20
```

### The Decision Tree

```txt
kubectl fails
    │
    ├─ ss -tlnp | grep 6443
    │       │
    │   not listening ──→ crictl ps -a | grep apiserver
    │                           │
    │                       Exited/CrashLoop ──→ check pod logs (Step 3)
    │                           │                      │
    │                       Not there ──→ check        ├─ etcd error ──→ Step 4
    │                       /etc/kubernetes/           │
    │                       manifests/ + kubelet       ├─ cert error ──→ kubeadm certs check-expiration
    │                       logs                       │
    │                                                  ├─ flag/config error ──→ check manifest yaml
    │                                                  │
    │                                                  └─ timeout/slow ──→ Step 5 (resources)
    │
    └─ listening ──→ kubectl get nodes (check kubeconfig/network/auth)
```

This case the **correct short path** was:

1. `ss -tlnp | grep 6443` → port missing ✓
2. `crictl ps -a | grep apiserver` → CrashLoopBackOff ✓
3. Pod logs → etcd timeouts (misleading but pointed us toward etcd)
4. etcd health check → healthy → ruled out etcd
5. `dmesg` + `iostat` → CPU starvation pattern ✓

Everything else was us ruling out red herrings along the way.

---

## The Actual Root Cause

There wasn't one single root cause — it was a **cascade**. Think of it like dominoes:

```txt
Something stressed the master node
        ↓
apiserver became slow to respond
        ↓
kubelet's startup probe timed out → killed apiserver
        ↓
apiserver restarted → but now competing for CPU with everything else
        ↓
startup took too long again → probe failed again → killed again
        ↓
CrashLoopBackOff back-off timer kicked in (5 minutes)
        ↓
During back-off, controller-manager & scheduler also lost apiserver
→ they started crashing too (111 and 105 restarts)
        ↓
More crashes = more CPU consumed by restart cycles
        ↓
Even less CPU for the next apiserver startup attempt
        ↓
Death spiral — couldn't recover on its own
```

The **triggering event** was probably one of these (we can't know for sure without
earlier logs):

- A noisy neighbour on the OpenStack hypervisor stealing vCPU time (`%steal` was
  0.74% in the iostat — small but real)
- metrics-server going into its own crash loop (101 restarts over 2 days) and
  consuming CPU
- Normal overnight Kubernetes housekeeping (GC, lease renewals) hitting all at
  once on an already tight 2-vCPU machine

---

## Why It Looked So Confusing

Each time we looked at logs we were seeing a **different stage of the same failed
startup**:

- Sometimes etcd write timeouts → because apiserver was dying _after_ connecting
  to etcd but _before_ binding port 6443
- Sometimes TLS handshake timeouts → because kubelet was trying to reach an apiserver
  that was mid-crash
- Sometimes `find /var/log/pods` froze → because the kernel workqueue was CPU-starved

None of those were the _cause_ — they were all _symptoms_ of the same underlying
CPU starvation spiral.

---

## How to Reproduce This Problem

```md
1. Run a full Kubernetes control plane on a 2-vCPU node
2. Let something trigger a crash loop (metrics-server, noisy neighbour, etc.)
3. Wait — the system cannot recover on its own because:
   - Each restart attempt consumes CPU
   - Less CPU → slower startup → probe timeout → kill → repeat
   - Back-off timer reaches 5 minutes → cluster stays down
```

---

## How to Prevent It

**Short term — fix the probe tolerances** so the apiserver gets more time to start
on a slow/loaded system:

```bash
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

Change `startupProbe`:

```yaml
startupProbe:
  failureThreshold: 24 # change to 40
  initialDelaySeconds: 10
  periodSeconds: 10 # 40 × 10s = 400s max startup time instead of 240s
  timeoutSeconds: 15
```

**Medium term — stop metrics-server from thrashing**, since it was the likely trigger.
Check if it needs `--kubelet-insecure-tls`:

```bash
kubectl logs -n kube-system deployment/metrics-server --tail=20
```

**Long term — yes, upgrade to 4 vCPU.** The current numbers show why:

```txt
master node: 529m CPU (26%) at idle
```

That's 26% just sitting there doing nothing special. One crash loop spike and the
server will spike at 80-100%, which is when the death spiral starts. 4 vCPU gives
the real headroom.

---

## TL;DR in one sentence

The apiserver crashed once for an unknown reason, and the 2-vCPU machine didn't
have enough headroom to restart it fast enough before the probe killed it again —
so it got stuck in a loop it couldn't escape from until the system happened to
get a quiet enough moment.
