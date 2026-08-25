# Exposing K3s Applications with Cloudflare Tunnel

This document describes the setup used to expose applications running in a Raspberry Pi K3s cluster to the public Internet using a custom domain, Cloudflare Tunnel, and the K3s Traefik ingress controller.

The example application in this guide is the Trading Journal application, exposed as:

```text
https://tradingjournal.clusterberry.net
```

The same pattern can be reused for additional applications and subdomains later.

---

## 1. Architecture

The final request path is:

```text
Internet
   |
   v
tradingjournal.clusterberry.net
   |
   v
Cloudflare DNS
   |
   v
Cloudflare Tunnel
   |
   v
cloudflared on pi-control-1
   |
   v
K3s Traefik :80
   |
   v
Kubernetes Ingress
   |------------------------------|
   v                              v
/api/*                         /*
   |                              |
   v                              v
trading-backend-service       trading-frontend-service
   |                              |
   v                              v
backend pod                   frontend pod
```

The important advantage of this design is that the home router does **not** need ports 80 or 443 forwarded from the Internet. `cloudflared` creates an outbound encrypted connection to Cloudflare.

---

## 2. Prerequisites

The following were already present before setting up the public domain:

- Raspberry Pi K3s cluster
- K3s Traefik ingress controller
- Trading Journal namespace
- Frontend and backend Kubernetes Services
- Existing local ingress using `trading-journal.local`
- `cloudflared` installed on `pi-control-1`
- A Cloudflare-managed domain: `clusterberry.net`

Useful checks:

```bash
kubectl get nodes -o wide
kubectl get ingress -A
kubectl get svc -n kube-system | grep traefik
which cloudflared
```

Example cluster:

```text
pi-control-1   control-plane,etcd
pi-control-2   control-plane,etcd
pi-control-3   control-plane,etcd
pi-worker-1    worker
```

---

## 3. Existing Trading Journal Ingress

The application already had a Traefik ingress similar to this:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: trading-journal-ingress
  namespace: trading-journal
spec:
  ingressClassName: traefik
  rules:
    - host: trading-journal.local
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: trading-backend-service
                port:
                  number: 8080
          - path: /
            pathType: Prefix
            backend:
              service:
                name: trading-frontend-service
                port:
                  number: 80
```

This means Traefik already knew how to route:

```text
trading-journal.local/api/* -> trading-backend-service:8080
trading-journal.local/*     -> trading-frontend-service:80
```

---

## 4. Add the Public Hostname to the Ingress

Edit the ingress:

```bash
kubectl edit ingress trading-journal-ingress -n trading-journal
```

Add a second host while keeping the local hostname:

```yaml
spec:
  ingressClassName: traefik
  rules:
    - host: trading-journal.local
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: trading-backend-service
                port:
                  number: 8080
          - path: /
            pathType: Prefix
            backend:
              service:
                name: trading-frontend-service
                port:
                  number: 80

    - host: tradingjournal.clusterberry.net
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: trading-backend-service
                port:
                  number: 8080
          - path: /
            pathType: Prefix
            backend:
              service:
                name: trading-frontend-service
                port:
                  number: 80
```

Do **not** manually edit the `status.loadBalancer` section. Kubernetes/K3s manages it automatically.

Verify:

```bash
kubectl describe ingress trading-journal-ingress -n trading-journal
```

Expected rules:

```text
trading-journal.local
  /api -> trading-backend-service:8080
  /    -> trading-frontend-service:80

tradingjournal.clusterberry.net
  /api -> trading-backend-service:8080
  /    -> trading-frontend-service:80
```

Test Traefik locally before involving Cloudflare:

```bash
curl -I -H "Host: tradingjournal.clusterberry.net" http://192.168.1.169
```

If this returns a valid HTTP response, Kubernetes ingress routing is working.

---

## 5. Authenticate cloudflared with Cloudflare

Run on `pi-control-1`:

```bash
cloudflared tunnel login
```

Because the Pi may be headless, copy the URL printed by the command and open it in a browser on another machine.

In Cloudflare:

1. Sign in.
2. Select the `clusterberry.net` zone.
3. Authorize `cloudflared`.

After successful authorization, verify that Cloudflare created:

```bash
ls -la ~/.cloudflared
```

You should see:

```text
cert.pem
```

Do not commit `cert.pem` to Git.

---

## 6. Create a Named Cloudflare Tunnel

Create a permanent tunnel:

```bash
cloudflared tunnel create clusterberry-k3s
```

This creates:

- A tunnel UUID
- A tunnel credentials JSON file under `~/.cloudflared/`

Verify:

```bash
cloudflared tunnel list
```

Example:

```text
NAME                ID
clusterberry-k3s    xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

Do not commit the tunnel credentials JSON file to Git.

---

## 7. Configure the Tunnel

Create the Cloudflare config file:

```bash
nano ~/.cloudflared/config.yml
```

Example:

```yaml
tunnel: YOUR_TUNNEL_UUID
credentials-file: /home/homelab/.cloudflared/YOUR_TUNNEL_UUID.json

ingress:
  - hostname: tradingjournal.clusterberry.net
    service: http://192.168.1.169:80

  - service: http_status:404
```

Replace `YOUR_TUNNEL_UUID` with the actual tunnel ID.

The final catch-all 404 rule is required so unmatched requests do not get routed accidentally.

Validate the ingress rules:

```bash
cloudflared tunnel ingress validate
```

---

## 8. Create the Public DNS Route

Create the DNS record for the public hostname:

```bash
cloudflared tunnel route dns clusterberry-k3s tradingjournal.clusterberry.net
```

Cloudflare creates the necessary DNS record pointing the hostname at the tunnel.

Verify the route/tunnel:

```bash
cloudflared tunnel list
cloudflared tunnel info clusterberry-k3s
```

---

## 9. Test the Tunnel Manually

Before installing it as a permanent service, test it interactively:

```bash
cloudflared tunnel run clusterberry-k3s
```

Leave the process running and open:

```text
https://tradingjournal.clusterberry.net
```

If the application loads, the following path is working:

```text
Cloudflare -> Tunnel -> Traefik -> Ingress -> Kubernetes Service -> Pod
```

Stop the manual tunnel with:

```text
Ctrl+C
```

---

## 10. Run cloudflared Permanently with systemd

If the manual tunnel stops, Cloudflare will return:

```text
Error 1033
Cloudflare Tunnel error
```

This means Cloudflare knows about the tunnel, but no active `cloudflared` connector is connected.

Install `cloudflared` as a Linux systemd service:

```bash
sudo cloudflared --config /home/homelab/.cloudflared/config.yml service install
```

Using the explicit config path is important because `sudo` changes the effective home directory to `/root`.

Enable and start the service:

```bash
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```

Or:

```bash
sudo systemctl enable --now cloudflared
```

Verify:

```bash
sudo systemctl status cloudflared --no-pager
```

Expected:

```text
Active: active (running)
```

Useful logs:

```bash
sudo journalctl -u cloudflared -f
```

Now the tunnel should start automatically whenever `pi-control-1` reboots.

---

## 11. Update Backend CORS for the Public Domain

After the public site loaded, registration requests returned HTTP `403`.

The backend deployment still contained the old CORS allowlist:

```text
http://trading-journal.local,
https://<old-random-name>.trycloudflare.com
```

The new public origin must be included:

```text
https://tradingjournal.clusterberry.net
```

Update the backend environment variable:

```bash
kubectl set env deployment/trading-backend \
  -n trading-journal \
  APP_CORS_ALLOWED_ORIGINS="http://trading-journal.local,https://tradingjournal.clusterberry.net"
```

Restart the deployment:

```bash
kubectl rollout restart deployment trading-backend -n trading-journal
```

Wait for rollout completion:

```bash
kubectl rollout status deployment trading-backend -n trading-journal
```

Verify the configured value:

```bash
kubectl get deployment trading-backend \
  -n trading-journal \
  -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="APP_CORS_ALLOWED_ORIGINS")].value}'
```

Expected:

```text
http://trading-journal.local,https://tradingjournal.clusterberry.net
```

After this change, registration through the public domain worked.

---

## 12. Verify the Application

Check application pods:

```bash
kubectl get pods -n trading-journal -o wide
```

Example healthy state:

```text
minio-0              1/1 Running
postgres-0           1/1 Running
trading-backend-*    1/1 Running
trading-frontend-*   1/1 Running
```

Check backend logs:

```bash
kubectl logs -n trading-journal deployment/trading-backend --tail=100
```

Check ingress:

```bash
kubectl describe ingress trading-journal-ingress -n trading-journal
```

Check tunnel service:

```bash
sudo systemctl status cloudflared --no-pager
```

Finally test:

```text
https://tradingjournal.clusterberry.net
```

---

## 13. Troubleshooting

### Cloudflare Error 1033

Symptom:

```text
Error 1033
Cloudflare Tunnel error
```

Cause:

Cloudflare has a route for the hostname, but no healthy tunnel connector is running.

Checks:

```bash
sudo systemctl status cloudflared --no-pager
cloudflared tunnel info clusterberry-k3s
sudo journalctl -u cloudflared -n 100 --no-pager
```

Start/restart:

```bash
sudo systemctl restart cloudflared
```

---

### HTTP 403 from API Requests

If the frontend loads but API calls return `403`, inspect the browser Network tab.

In this setup, the cause was the backend CORS configuration not allowing:

```text
https://tradingjournal.clusterberry.net
```

Check:

```bash
kubectl get deployment trading-backend -n trading-journal -o yaml | \
  grep -A5 -B3 APP_CORS_ALLOWED_ORIGINS
```

Update with `kubectl set env` as described above.

---

### Frontend Works but Backend Does Not

Check the ingress routing:

```bash
kubectl describe ingress trading-journal-ingress -n trading-journal
```

Expected:

```text
/api -> trading-backend-service:8080
/    -> trading-frontend-service:80
```

Check services:

```bash
kubectl get svc -n trading-journal
kubectl get endpoints -n trading-journal
```

---

### Test Traefik Without Cloudflare

This is useful for separating Kubernetes problems from Cloudflare problems:

```bash
curl -H "Host: tradingjournal.clusterberry.net" http://192.168.1.169
```

If this works but the public URL fails, investigate Cloudflare Tunnel.

If this fails, investigate Traefik/Ingress/Kubernetes routing first.

---

## 14. Security Notes

### Never commit Cloudflare credentials

Do not commit any of the following:

```text
~/.cloudflared/cert.pem
~/.cloudflared/*.json
Cloudflare tunnel tokens
```

Recommended `.gitignore` entries if Cloudflare files ever exist inside the repository:

```gitignore
.cloudflared/
cert.pem
*.json
```

Be careful with a broad `*.json` rule if your project intentionally tracks JSON files.

### No router port forwarding required

Cloudflare Tunnel uses an outbound connection, so ports 80/443 do not need to be exposed publicly on the home router.

### Keep internal K3s addresses private

Addresses such as these remain private LAN addresses and should not be directly exposed to the Internet:

```text
192.168.1.x
10.42.x.x
10.43.x.x
```

---

## 15. Adding More Applications Later

The same tunnel can serve multiple subdomains.

For example:

```text
tradingjournal.clusterberry.net
grafana.clusterberry.net
nexus.clusterberry.net
sonarqube.clusterberry.net
```

Add another Kubernetes ingress host for the application, then add another hostname to `~/.cloudflared/config.yml`.

Example:

```yaml
ingress:
  - hostname: tradingjournal.clusterberry.net
    service: http://192.168.1.169:80

  - hostname: grafana.clusterberry.net
    service: http://192.168.1.169:80

  - service: http_status:404
```

Then create the DNS route:

```bash
cloudflared tunnel route dns clusterberry-k3s grafana.clusterberry.net
```

Because Traefik receives the original hostname, it can route each public hostname to a different Kubernetes Service.

After modifying the tunnel config, restart the service:

```bash
sudo systemctl restart cloudflared
```

---

## 16. Current Setup Summary

```text
Domain:
  clusterberry.net

Public application:
  https://tradingjournal.clusterberry.net

Tunnel:
  clusterberry-k3s

Tunnel host:
  pi-control-1

Origin:
  http://192.168.1.169:80

Ingress controller:
  Traefik (K3s)

Ingress:
  trading-journal/trading-journal-ingress

Routing:
  /api -> trading-backend-service:8080
  /    -> trading-frontend-service:80

Local hostname retained:
  http://trading-journal.local
```

---

## 17. Future Improvement: High Availability

The current tunnel depends on `pi-control-1` because `cloudflared` runs there and the origin in `config.yml` points at `192.168.1.169:80`.

A future improvement is to run `cloudflared` as replicated pods inside K3s and target the Traefik Kubernetes Service rather than a specific Raspberry Pi IP.

That architecture would be:

```text
Cloudflare
   |
   v
cloudflared Deployment (2+ replicas in K3s)
   |
   v
Traefik Kubernetes Service
   |
   v
Ingress rules
   |
   v
Application Services
```

This removes `pi-control-1` as a single point of failure for the public tunnel.

---

## Quick Command Reference

```bash
# Authenticate Cloudflare CLI
cloudflared tunnel login

# Create named tunnel
cloudflared tunnel create clusterberry-k3s

# List tunnels
cloudflared tunnel list

# Validate tunnel config
cloudflared tunnel ingress validate

# Create public DNS hostname
cloudflared tunnel route dns clusterberry-k3s tradingjournal.clusterberry.net

# Test tunnel manually
cloudflared tunnel run clusterberry-k3s

# Install persistent service
sudo cloudflared --config /home/homelab/.cloudflared/config.yml service install
sudo systemctl enable --now cloudflared

# Check service
sudo systemctl status cloudflared --no-pager

# Follow logs
sudo journalctl -u cloudflared -f

# Check ingress
kubectl describe ingress trading-journal-ingress -n trading-journal

# Check app pods
kubectl get pods -n trading-journal -o wide

# Update backend CORS
kubectl set env deployment/trading-backend \
  -n trading-journal \
  APP_CORS_ALLOWED_ORIGINS="http://trading-journal.local,https://tradingjournal.clusterberry.net"
```

