# K3s Setup in `pi-control-1` and `pi-control-2`

## Objective
Set up a multi-control-plane K3s cluster on Raspberry Pi and document the errors encountered and how they were fixed.

## Nodes Used
- `pi-control-1` -> `192.168.1.169`
- `pi-control-2` -> `192.168.1.216`

Laptop IPs allowed for SSH:
- `192.168.1.170`
- `192.168.1.198`

---

## 1. Access `pi-control-1`

```bash
ssh node1@192.168.1.169
```

---

## 2. Prepare `pi-control-1`

Update packages:

```bash
sudo apt update -y
sudo apt install -y curl wget
```

Enable cgroups:

```bash
sudo nano /boot/firmware/cmdline.txt
```

At the end of the existing single line, add:

```text
cgroup_memory=1 cgroup_enable=memory
```

Reboot:

```bash
sudo reboot
```

---

## 3. Initialize K3s on `pi-control-1`

```bash
curl -sfL https://get.k3s.io | K3S_TOKEN=<YOUR_CLUSTER_TOKEN> sh -s - server --cluster-init
```

This initializes `pi-control-1` as the first control-plane node.

---

## 4. Firewall Rules Needed for K3s

To allow `pi-control-2` to join the cluster, `pi-control-1` needed to allow K3s traffic.

Allow K3s API access:

```bash
sudo ufw allow from 192.168.1.216 to any port 6443 proto tcp
```

Official K3s HA requirements:
- `6443/TCP` must be reachable from nodes to servers
- For embedded etcd HA, server nodes must reach each other on:
  - `2379/TCP`
  - `2380/TCP`

Allow embedded etcd peer traffic:

```bash
sudo ufw allow from 192.168.1.216 to any port 2379 proto tcp
sudo ufw allow from 192.168.1.216 to any port 2380 proto tcp
```

---

## 5. Check etcd Members on `pi-control-1`

To list all members registered in the embedded etcd cluster:

```bash
sudo etcdctl \
  --cacert=/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/k3s/server/tls/etcd/client.crt \
  --key=/var/lib/rancher/k3s/server/tls/etcd/client.key \
  --endpoints=https://127.0.0.1:2379 \
  member list
```

---

## 6. Remove a Stale etcd Member

A failed join attempt left a stale etcd member for `pi-control-2`, which caused duplicate node name errors.

```bash
sudo etcdctl \
  --cacert=/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/k3s/server/tls/etcd/client.crt \
  --key=/var/lib/rancher/k3s/server/tls/etcd/client.key \
  --endpoints=https://127.0.0.1:2379 \
  member remove e4f05284fcddf8f7
```

---

## 7. Join `pi-control-2` to the Cluster

Run on `pi-control-2`:

```bash
curl -sfL https://get.k3s.io | K3S_TOKEN='<FULL_SERVER_TOKEN_FROM_PI_CONTROL_1>' sh -s - server --server https://192.168.1.169:6443
```

This joins `pi-control-2` to the existing cluster created by `pi-control-1`.

---

## 8. View K3s Logs

To follow K3s logs in real time:

```bash
sudo journalctl -u k3s -f
```

This was useful for troubleshooting:
- connection issues
- duplicate node name errors
- etcd join failures
- TLS and handshake timeouts

---

## 9. Final Firewall Tightening

After debugging, the firewall was tightened again.

Goal:
- allow all traffic between cluster Pis
- allow SSH only from laptop IPs
- deny all other inbound traffic

Run on each Pi:

```bash
sudo ufw --force reset

sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow all traffic from cluster Pis
sudo ufw allow from 192.168.1.169
sudo ufw allow from 192.168.1.216

# Allow SSH only from laptops
sudo ufw allow from 192.168.1.170 to any port 22 proto tcp
sudo ufw allow from 192.168.1.198 to any port 22 proto tcp

sudo ufw enable
sudo ufw status numbered
```

---

## 10. Final Verification

Verify both control-plane nodes successfully joined:

```bash
sudo kubectl get nodes -o wide
```

Expected result:

```text
NAME           STATUS   ROLES                AGE   VERSION        INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION         CONTAINER-RUNTIME
pi-control-1   Ready    control-plane,etcd   ...   v1.34.6+k3s1   192.168.1.169   <none>        Debian GNU/Linux 13 (trixie)     6.12.75+rpt-rpi-2712   containerd://2.2.2-bd1.34
pi-control-2   Ready    control-plane,etcd   ...   v1.34.6+k3s1   192.168.1.216   <none>        Debian GNU/Linux 12 (bookworm)   6.6.51+rpt-rpi-2712    containerd://2.2.2-bd1.34
```

---

## 11. Errors Faced and Fixes Applied

### Error 1: Wrong IP address used for `pi-control-1`
**Issue:** `pi-control-2` was initially trying to join the wrong IP.

**Fix:** Checked the actual IP using:

```bash
hostname -I
```

Updated the join command to use `192.168.1.169`.

### Error 2: Port `6443` blocked by firewall
**Issue:** `pi-control-2` could ping `pi-control-1` but could not connect to the K3s API.

**Fix:** Opened `6443/tcp` on `pi-control-1` for `pi-control-2`.

### Error 3: Ports `2379` and `2380` blocked
**Issue:** etcd peer communication timed out during cluster join.

**Fix:** Opened embedded etcd ports `2379/tcp` and `2380/tcp` between the control nodes.

### Error 4: Duplicate node name error
**Issue:** A partial failed join created a stale etcd member for `pi-control-2`, causing duplicate node name errors.

**Fix:** Listed etcd members and removed the stale member with `etcdctl member remove`.

### Error 5: Broad debugging firewall rule
**Issue:** A temporary rule allowed all traffic from `192.168.1.0/24` during debugging.

**Fix:** Replaced it with tighter rules that allow:
- all traffic only between cluster Pis
- SSH only from laptop IPs

---

## 12. Current Status

Completed:
- `pi-control-1` is running as the first control-plane node
- `pi-control-2` successfully joined as the second control-plane node
- both nodes are `Ready`
- firewall rules have been tightened again

Pending:
- `pi-control-3` setup
- worker node setup
- final HA cluster expansion
- application deployment and monitoring

---

## 13. Notes

- Embedded etcd HA works best with **3 control-plane nodes**
- Current setup with 2 control-plane nodes works, but the final design should include `pi-control-3`
- Static IPs or DHCP reservations are recommended to avoid future join issues
- Avoid storing live cluster tokens directly in documentation; use placeholders instead
