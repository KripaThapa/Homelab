# Homelab + Family Dashboard Roadmap

## Goal
Build a shared Raspberry Pi home lab platform first, then deploy a family dashboard / life management system on top of it.

## Team Setup
- **Person A**: cluster setup, backend APIs, database, deployment manifests
- **Person B**: monitoring, GitOps, frontend, dashboards/UI
- Swap ownership on later modules so both people learn app and infra.

## Time Budget
- Saturdays and Sundays only
- 6 hours per day as a team
- 12 team-hours per weekend

---

## Week 1 — March 28–29, 2026
### Goal
Prepare all Raspberry Pis, network, storage, and repo basics.

### Saturday, March 28, 2026
#### Tasks
- Label all 4 Raspberry Pis physically
- Install OS on each Pi
- Update packages
- Set static IPs or DHCP reservations
- Enable SSH on all Pis
- Verify all nodes can communicate with each other
- Decide and document node names:
  - `pi-control-1`
  - `pi-control-2`
  - `pi-control-3`
  - `pi-worker-1`

#### Definition of done
- All 4 Pis reachable over SSH
- Hostnames and IPs documented

### Sunday, March 29, 2026
#### Tasks
- Attach and test 1 TB external storage
- Decide which node hosts external storage first
- Create initial repo structure
- Add setup documentation
- Document credentials, hostnames, setup steps, and ownership

#### Definition of done
- External drive mounted and tested
- Repo contains initial docs

---

## Week 2 — April 4–5, 2026
### Goal
Bring up the K3s cluster and deploy a first test app.

### Saturday, April 4, 2026
#### Tasks
- Install K3s server on `pi-control-1`
- Join `pi-control-2` and `pi-control-3` as K3s server nodes
- Join `pi-worker-1` as worker node
- Verify cluster node health
- Deploy a simple test app such as nginx or hello-world

#### Definition of done
- 4-node K3s cluster running
- Test app reachable inside cluster

### Sunday, April 5, 2026
#### Tasks
- Create namespaces:
  - `platform`
  - `monitoring`
  - `apps`
- Write first deployment YAML manually
- Deploy test frontend/backend pods
- Document cluster bring-up steps in repo

#### Definition of done
- Namespaces created
- One test app deployed from YAML
- Basic deployment doc committed

---

## Week 3 — April 11–12, 2026
### Goal
Add ingress and stable access patterns.

### Saturday, April 11, 2026
#### Tasks
- Use K3s default Traefik ingress
- Configure ingress for test app
- Expose app through local domain or local IP routing
- Standardize names:
  - `app.home.lab`
  - `grafana.home.lab`
  - `api.home.lab`

#### Definition of done
- Test app reachable through ingress

### Sunday, April 12, 2026
#### Tasks
- Set up local DNS or hosts-file based routing
- Standardize Kubernetes manifest layout
- Add configuration examples
- Decide temporary secrets handling strategy

#### Definition of done
- Ingress naming pattern documented
- Routing and config docs committed

---

## Week 4 — April 18–19, 2026
### Goal
Add monitoring and cluster visibility.

### Saturday, April 18, 2026
#### Tasks
- Deploy Prometheus
- Deploy Node Exporter
- Deploy Grafana
- Verify dashboards show:
  - CPU
  - memory
  - disk
  - node availability

#### Definition of done
- Prometheus scraping node metrics
- Grafana dashboard working

### Sunday, April 19, 2026
#### Tasks
- Add dashboards for:
  - pod restarts
  - cluster health
  - storage usage
- Simulate a pod failure and observe metrics
- Write debugging notes for node/app health

#### Definition of done
- Usable dashboards exist
- One failure test documented

---

## Week 5 — April 25–26, 2026
### Goal
Add GitOps and persistent storage foundation.

### Saturday, April 25, 2026
#### Tasks
- Install Argo CD
- Connect Argo CD to repo
- Move manifests into GitOps structure
- Deploy one test app only through Argo CD

#### Definition of done
- Argo CD syncing from repo
- Test app deploys through GitOps

### Sunday, April 26, 2026
#### Tasks
- Set up persistent volume approach for MVP
- Use external drive for stateful workloads
- Deploy Postgres with persistent storage
- Restart Postgres and verify data survives

#### Definition of done
- Postgres running with persistence
- Home lab platform MVP is ready

---

## Platform Checkpoint — April 26, 2026
By this date, the repo/platform should include:
- K3s cluster
- ingress
- monitoring
- Argo CD
- persistent Postgres
- basic ops docs

---

## Week 6 — May 2–3, 2026
### Goal
Design the family dashboard MVP and scaffold the codebase.

### Saturday, May 2, 2026
#### Tasks
- Freeze MVP scope:
  - login
  - chores/tasks
  - grocery list
  - reminders
  - notes
- Create system design doc
- Define DB schema
- Define API contracts

#### Definition of done
- MVP scope frozen
- DB schema draft complete
- API list documented

### Sunday, May 3, 2026
#### Tasks
- Scaffold frontend project
- Scaffold backend project
- Set up database migrations
- Create initial tables:
  - `users`
  - `households`
  - `tasks`
  - `groceries`
  - `reminders`
  - `notes`

#### Definition of done
- Frontend and backend bootstrapped
- DB schema applied successfully

---

## Week 7 — May 9–10, 2026
### Goal
Build authentication and the household model.

### Saturday, May 9, 2026
#### Tasks
- Implement login/auth flow
- Create user registration or invite-only flow
- Create household entity
- Associate users with household
- Protect endpoints

#### Definition of done
- Users can sign in
- Household-based data model works

### Sunday, May 10, 2026
#### Tasks
- Build frontend login page
- Build dashboard shell
- Show authenticated user info
- Add route/session guards

#### Definition of done
- Login works end to end

---

## Week 8 — May 16–17, 2026
### Goal
Build chores/tasks and grocery list.

### Saturday, May 16, 2026
#### Tasks
- Build backend endpoints for tasks/chores
- Add CRUD for task title, assignee, due date, and status
- Build grocery list backend APIs

#### Definition of done
- Task and grocery APIs complete

### Sunday, May 17, 2026
#### Tasks
- Build frontend pages for tasks and grocery list
- Add create/edit/complete task interactions
- Add grocery item add/remove/check-off flows
- Test multi-user household behavior

#### Definition of done
- First useful family dashboard version usable by both of you

---

## Week 9 — May 23–24, 2026
### Goal
Build reminders, notes, and scheduled jobs.

### Saturday, May 23, 2026
#### Tasks
- Create reminders table and APIs
- Add background scheduler/worker
- Support daily/weekly reminders
- Choose MVP notification approach (in-app first)

#### Definition of done
- Reminder engine works in backend

### Sunday, May 24, 2026
#### Tasks
- Add notes feature
- Add reminders UI
- Show upcoming reminders on home page
- Test scheduled jobs and reminder state changes

#### Definition of done
- Dashboard feels like a real life-management app

---

## Week 10 — May 30–31, 2026
### Goal
Deploy, harden, document, and prepare next phase.

### Saturday, May 30, 2026
#### Tasks
- Deploy family dashboard through Argo CD
- Add monitoring for app health, DB health, and pod restarts
- Create backup strategy for Postgres
- Test node restart and app recovery

#### Definition of done
- Family dashboard fully deployed on home lab

### Sunday, May 31, 2026
#### Tasks
- Fix priority bugs
- Cleanup UI
- Write project docs:
  - architecture
  - deployment flow
  - failover lessons
  - known limitations
- Record demo screenshots/video
- Create backlog for phase 2:
  - bills
  - pet records
  - plant reminders
  - document vault

#### Definition of done
- Version 1 complete and demo-ready

---

## Final Delivery Date
**May 31, 2026**

By this date, you should have:
- a functioning Raspberry Pi home lab platform
- GitOps deployment flow
- monitoring and persistence
- a deployed family dashboard MVP for daily use

---

## Working Rules
- Saturday = build the core foundation
- Sunday = test, integrate, document, and demo
- Every weekend must end with something visible:
  - deployed service
  - working UI
  - monitoring dashboard
  - documentation

## Scope Guardrails
Do **not** add these during this phase:
- Longhorn or advanced distributed storage
- mobile app
- AI features
- document OCR
- event bus unless truly needed
- Redis unless there is a clear performance or queue need

Keep the first release finishable.
