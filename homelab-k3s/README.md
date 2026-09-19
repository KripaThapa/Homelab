# Homelab Kubernetes desired state

Application manifests live in `k3s/apps/`; namespaces live in `k3s/namespaces/`.

- [Market research infrastructure and manual deployment](k3s/apps/market-research/README.md)
  — runtime contract resolved; awaiting a successful real SHA release and manual prerequisites.

Market research is initially cluster-only. Deployment, Secrets and release selection are
manual. Its migration and baseline Jobs remain suspended templates for explicit execution.
