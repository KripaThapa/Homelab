# Homelab Demo: Nginx on K3s

## Goal
Deploy a simple Nginx application into our K3s cluster using declarative Kubernetes YAML files.

## Architecture Flow
Browser -> Ingress -> Service -> Pods

Separately:
Deployment -> ReplicaSet -> Pods

## Cluster
- pi-control-1 (control-plane)
- pi-control-2 (control-plane)
- pi-control-3 (control-plane)
- pi-worker-1 (worker)

## Files
- k8s/namespaces/homelab-demo.yaml
- k8s/apps/nginx/deployment.yaml
- k8s/apps/nginx/service.yaml
- k8s/apps/nginx/ingress.yaml

## Apply Commands
Create namespace:
```bash
kubectl apply -f k8s/namespaces/homelab-demo.yaml
```

Deploy nginx:
```bash
kubectl apply -f k8s/apps/nginx/deployment.yaml
```

Create service:
```bash
kubectl apply -f k8s/apps/nginx/service.yaml
```

Create ingress:
```bash
kubectl apply -f k8s/apps/nginx/ingress.yaml
```

Apply everything:
```bash
kubectl apply -f k8s/
```

## Verify
```bash
kubectl get all -n homelab-demo
kubectl get ingress -n homelab-demo
kubectl get endpoints -n homelab-demo
```

## Debug Commands
Check nodes:
```bash
kubectl get nodes -o wide
```

Check k3s:
```bash
sudo systemctl status k3s --no-pager
sudo journalctl -u k3s -n 50 --no-pager -l
```

Check etcd leader:
```bash
sudo ETCDCTL_API=3 etcdctl --endpoints=https://192.168.1.169:2379,https://192.168.1.216:2379,https://192.168.1.218:2379 --cacert=/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt --cert=/var/lib/rancher/k3s/server/tls/etcd/client.crt --key=/var/lib/rancher/k3s/server/tls/etcd/client.key endpoint status -w table
```

## Concepts
- Namespace organizes resources.
- Deployment keeps desired pods running.
- Service finds pods by labels and load balances.
- Ingress routes external HTTP traffic to Services.

Label flow:
- Deployment creates pods with label app=nginx
- Service selector must match app=nginx
- If selector mismatches, Endpoints become empty.

## Access
Local access example:
- Add hosts entry mapping nginx.local to control-plane IP.
- Access http://nginx.local

## Next Milestones
- Monitoring (Prometheus + Grafana)
- GitOps (Argo CD or Flux)
- Secure remote access (Tailscale or Cloudflare Tunnel)
