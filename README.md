# Homelab

Shared Raspberry Pi home lab platform that will later host the Family Dashboard / Life Management System.

## Goals
- Build a real home lab platform on 4 Raspberry Pi 5 devices
- Learn infrastructure, deployment, monitoring, storage, and app hosting
- Deploy a useful day-to-day family dashboard on top of the platform

## Current Phase
Week 1 — Pi setup, network, storage

## Team
- Person 1: John
- Person 2: Friend

## Planned Phases
1. Raspberry Pi setup and network baseline
2. K3s cluster bring-up
3. Ingress and routing
4. Monitoring
5. GitOps and persistence
6. Family dashboard MVP

## Repo Structure
- `docs/` → project documentation
- `infra/` → infrastructure setup files
- `k8s/` → Kubernetes manifests
- `apps/` → application code
- `scripts/` → helper scripts

## Definition of Success
By the end of the first phase, all 4 Pis should be reachable over SSH, documented, and ready for cluster setup.
