# Market research: manual Kubernetes deployment

Runtime contract resolved. **Intended release: `sha-3875a67`.** No cluster actions, Secrets, migrations or baseline runs were
performed while preparing these files. Application source and research behavior belong
in the separate market-scanner repository.

## Layout and architecture

Namespace: `market-research`, in `../../namespaces/market-research.yaml`.

| Directory/file | Workload | Service / container port |
| --- | --- | --- |
| postgres/ | StatefulSet market-postgres | postgres / 5432 |
| backend/ | Deployment market-backend | backend / 8000 HTTP |
| internal-backend/ | Deployment market-internal-backend; uploads PVC | internal-backend / 8001 HTTP |
| scanner/ | Deployment market-scanner | None |
| research/ | Deployment market-research | None |
| frontend/ | Deployment market-frontend | market-frontend-service / 8080 HTTP |
| strategy-lab/ | Deployment market-strategy-lab | market-strategy-lab-service / 8080 HTTP |
| configmap.yaml | Explicit non-secret runtime settings | None |
| jobs/migration.yaml.template | Suspended Job market-migration | None |
| jobs/historical-baseline.yaml.template | Suspended Job market-historical-baseline | None |

All Services are ClusterIP; postgres is headless. The public frontend is routed by `ingress.yaml`; there is no NodePort, LoadBalancer, host
networking, or node pinning. The intended route is:

```text
scanner.clusterberry.net
→ Cloudflare Tunnel
→ pi-control-1 port 80
→ Traefik
→ market-frontend-service:8080
```

Cloudflare Tunnel supplies the external HTTPS edge; this Ingress has no Kubernetes TLS
termination. The route targets the frontend only; its Nginx proxies `/api/` internally
to the public backend. No backend port is directly exposed.

A separate `strategy-lab-ingress.yaml` prepares the private Strategy Lab Kubernetes origin:

```text
lab.clusterberry.net
→ Cloudflare Access
→ Cloudflare Tunnel
→ pi-control-1 port 80
→ Traefik
→ market-strategy-lab-service:8080
```

This Ingress only prepares the Kubernetes origin. **Do not add `lab.clusterberry.net` to
Cloudflare Tunnel ingress or public DNS until Cloudflare Access protection has been
configured and verified.** External HTTPS terminates at Cloudflare; Kubernetes TLS is not
configured here. Strategy Lab is an owner/private research tool. Its frontend proxies
`/api/internal/` to `internal-backend`; that backend remains ClusterIP-only and is not
routed directly by either Ingress. PostgreSQL, scanner and research remain private.

The internal API has no authentication/authorization. ClusterIP does not isolate it from
other cluster workloads; no NetworkPolicy isolation is claimed. Keep Cloudflare Access
in front of any external Strategy Lab route and do not expose the internal backend.

All Deployments have one replica and Recreate updates: this avoids overlapping worker
instances and temporary resource doubling, at the cost of update downtime. Research
continues its existing nightly scheduling; there is no CronJob.

## Images and exact commands

Use the same immutable release in all eight application workloads/Jobs:

```text
ghcr.io/kripathapa/market-scanner/application:sha-3875a67
ghcr.io/kripathapa/market-scanner/frontend:sha-3875a67
ghcr.io/kripathapa/market-scanner/strategy-lab:sha-3875a67
```

PostgreSQL uses `postgres:16-bookworm`. Application Actions builds and checks manifests
for `linux/amd64` and `linux/arm64`; Raspberry Pi needs arm64. The intended release is now `sha-3875a67`.
Confirm its successful Actions architecture results before deployment.
No independent registry or Pi runtime check was performed here. No floating application tags are used.

The shared application image uses `/app`; Python manifests explicitly preserve that
working directory. Commands are:

```text
backend:          python -m uvicorn backend.api:create_app --factory --host 0.0.0.0 --port 8000
internal-backend: python -m uvicorn backend.internal_api:create_internal_app --factory --host 0.0.0.0 --port 8001
scanner:          python -m scanner.worker
research:         python -m research.job
migration:        python -m alembic upgrade head
baseline:         python -m research.baseline --last-trading-days 20
```

Frontends preserve their image ENTRYPOINT/CMD. Baked Nginx routes `/api/` to
`http://backend:8000` with Host `backend`, and `/api/internal/` to
`http://internal-backend:8001` with Host `internal-backend`. Paths are preserved. These
exact Service names work in the namespace without replacement Nginx configuration.
Strategy Lab does not proxy `/internal/watchlist/upload` or `/internal/health`.

The public and internal API factories are separate. Public API serves display-only
routes; internal routes are not mounted by its command. Never substitute one factory
for the other.

## Configuration and manually managed Secrets

No Secret manifests are included. Application apply never manages Secret values.
Create Secrets manually in `market-research`; do not reuse another namespace's Secrets.

| Secret | Exact keys | Consumers |
| --- | --- | --- |
| market-scanner-db | POSTGRES_DB, POSTGRES_USER, POSTGRES_PASSWORD | PostgreSQL, both APIs, scanner, research, migration, baseline |
| market-scanner-alpaca | ALPACA_API_KEY, ALPACA_SECRET_KEY | Internal-backend, scanner, research, baseline only |

Python uses SQLAlchemy/psycopg and constructs its database connection safely from the
three POSTGRES variables, using `postgres:5432`. No DATABASE_URL is configured or needed.
Changing initialization environment values does not rotate an existing database password.

`market-scanner-config` contains the verified defaults, with explicit configMapKeyRef
selection by component (no blanket envFrom):

| Group | Values | Consumers |
| --- | --- | --- |
| Provider | MARKET_DATA_PROVIDER=alpaca_iex; ALPACA_PAPER=true | internal-backend, scanner, research, baseline |
| FORMING | FORMING_LOOKBACK_BARS=6; FORMING_MIN_RETRACE_PCT=0.003; FORMING_CLOUD_PROXIMITY_PCT=0.004; FORMING_SLOW_TOLERANCE_PCT=0.002 | internal-backend, scanner, research, baseline |
| Trading window | TRADING_WINDOW_START=08:00; TRADING_WINDOW_END=10:00; TRADING_TIMEZONE=America/Chicago | internal-backend, scanner, research |
| Scanner/discovery | SCANNER_INTERVAL_SECONDS=60; SCANNER_JSON_FALLBACK=false; DISCOVERY_ENABLED=true; DISCOVERY_INTERVAL_SECONDS=300; DISCOVERY_TOP_N=10 | scanner |
| Nightly | NIGHTLY_ANALYSIS_TIME=18:00; NIGHTLY_ANALYSIS_TIMEZONE=America/Chicago; BASELINE_NIGHTLY_ENABLED=true; BASELINE_MAX_CATCHUP_TRADING_DAYS=5 | research |
| Baseline retries | BASELINE_PROVIDER_RETRIES=3; BASELINE_RETRY_DELAY_SECONDS=2 | internal-backend, research, baseline |
| Replay | REPLAY_TIMEZONE=America/Chicago; REPLAY_START=08:00; REPLAY_EQUITY_START=08:30; REPLAY_END=10:00; REPLAY_WARMUP_CALENDAR_DAYS=7 | internal-backend, research, baseline |
| Uploads | RIPSTER_DATA_DIR=/app/data | internal-backend |
| Public controls | PUBLIC_RATE_LIMIT_PER_MINUTE=120; PUBLIC_EXPENSIVE_RATE_LIMIT_PER_MINUTE=30; TRUSTED_HOSTS=localhost,127.0.0.1,backend | backend |

CORS_ALLOWED_ORIGINS is deliberately unset, preserving application defaults. The initial
frontends proxy same-origin requests. Revisit CORS/access origins, trusted hosts and
internal upload Origin restrictions when LAN/public access is designed. Do not add
wildcards. Scanner reads bundled `/app/config/sectors.json`; JSON watchlist fallback
remains disabled. ConfigMap environment changes require Pod recreation to take effect.

### Manual Secret creation command templates — never executed here

Run only after namespace creation. Placeholder strings below are not usable credentials.
Replace them privately; avoid shell history, tracing, recordings and committing commands
with real values. `create` refuses to overwrite an existing Secret. Do not run these
commands with literal placeholders. Never print Secret contents for verification.

```bash
kubectl create secret generic market-scanner-db \
  -n market-research \
  --from-literal=POSTGRES_DB='<DATABASE_NAME>' \
  --from-literal=POSTGRES_USER='<DATABASE_USER>' \
  --from-literal=POSTGRES_PASSWORD='<PRIVATE_DATABASE_PASSWORD>'

kubectl create secret generic market-scanner-alpaca \
  -n market-research \
  --from-literal=ALPACA_API_KEY='<PRIVATE_ALPACA_API_KEY>' \
  --from-literal=ALPACA_SECRET_KEY='<PRIVATE_ALPACA_SECRET_KEY>'
```

The current GHCR packages for release `sha-3875a67` are publicly readable; no image
pull Secret or registry authentication is required for this deployment. During the first
migration deployment, the operator confirmed that the application image pulled without
authentication despite a warning about an unavailable previously referenced pull Secret.
The stale Pod-template references have been removed. No cluster verification was performed
as part of this repository update.

If these packages become private in the future, manually create a registry image pull
Secret in `market-research` and add its reference to the affected Pod templates before
pulling private images. Keep registry credentials out of Git.

## Persistent storage and permissions

PostgreSQL: 5Gi ReadWriteOnce claim generated as `postgres-storage-market-postgres-0`,
mounted at `/var/lib/postgresql/data`. Uploads: separate **1Gi ReadWriteOnce** PVC
`market-scanner-uploads`, mounted only on internal-backend at `/app/data/uploads`.
The conservative upload size needs monitoring as retained screenshots accumulate.
No MinIO, shared worker filesystem, or other application PVC is needed.

Both claims omit storageClassName, matching repository default-storage conventions.
Verify the default StorageClass before deployment. Default k3s local-path storage is
node-local: persistence survives Pod replacement but not disk loss; moving to another
node may require recovery. Volume placement constrains scheduling without a hard-coded
node selector. Do not delete PVCs or namespace during updates/rollback. Back up PostgreSQL
and retained uploads outside their PVCs.

Internal-backend runs non-root as UID/GID 10001, with fsGroup 10001 and
fsGroupChangePolicy OnRootMismatch. Privilege escalation is disabled and capabilities
are dropped. This requests writable volume group ownership without a privileged
container. Whether group ownership is applied depends on the installed volume driver
and provisioning behavior, which has not been checked against the cluster. Verify write
access at first deployment; if the provisioner does not honor fsGroup, an operator must
prepare ownership/group permissions on the backing volume. Do not solve this by making
the application privileged or globally writable. Image-layer ownership alone is not
sufficient for a newly mounted PVC.

## Health and initial resources

Backend readiness: GET `/health`, port 8000, with `Host: backend` to satisfy trusted-host
middleware. Internal readiness: GET `/internal/health`, port 8001. Both check database
connectivity and are **not liveness probes**. PostgreSQL readiness uses pg_isready with
POSTGRES_USER and POSTGRES_DB. No liveness or fabricated worker HTTP probes are added.
API readiness alone does not prove provider access, schema currency, or baseline coverage.

| Container | CPU request / limit | Memory request / limit |
| --- | --- | --- |
| PostgreSQL | 100m / 500m | 256Mi / 512Mi |
| Each API, scanner, migration | 100m / 500m | 128Mi / 512Mi |
| Research, baseline | 100m / 500m | 256Mi / 768Mi |
| Each frontend | 50m / 250m | 64Mi / 256Mi |

Steady-state requests are 600m CPU and 1024Mi memory, excluding Jobs. These are initial
budgets, not measured Pi requirements. Watch OOM/restarts, throttling and PVC capacity.

## First deployment on pi-control-1 — commands for the operator only

Start in the checkout directory containing `k3s/`. All six Deployments and both Job
templates now select the intended release `sha-3875a67`.
Review/commit the desired-state diff. No automatic GitHub/SSH deployment is installed.
Do not proceed with placeholder image tags.

1. Create namespace and inspect storage; manually create the two application Secrets using the
   procedures above. Stop until all prerequisites are satisfied.

   ```bash
   kubectl apply -f k3s/namespaces/market-research.yaml
   kubectl get storageclass
   ```

2. Apply configuration, then PostgreSQL and upload storage. Verify PostgreSQL readiness.
   Upload PVC may remain Pending until its consumer schedules with WaitForFirstConsumer.

   ```bash
   kubectl apply -f k3s/apps/market-research/configmap.yaml
   kubectl apply -f k3s/apps/market-research/postgres/
   kubectl apply -f k3s/apps/market-research/internal-backend/pvc.yaml
   kubectl -n market-research rollout status statefulset/market-postgres --timeout=300s
   kubectl -n market-research get pvc
   ```

3. Deliberate migration: explicitly create the suspended template, then authorize its run.
   Exact-file creation reads YAML regardless of the .template suffix. Directory traversal
   does not select that suffix. An existing named Job causes create to fail: stop and
   inspect it instead of blindly proceeding to the patch command.

   ```bash
   kubectl create -f k3s/apps/market-research/jobs/migration.yaml.template && \
     kubectl -n market-research patch job market-migration --type=merge -p '{"spec":{"suspend":false}}'
   kubectl -n market-research wait --for=condition=complete job/market-migration --timeout=600s
   kubectl -n market-research logs job/market-migration
   ```

   **Stop on creation failure, migration failure or timeout.** Only verified migration
   success permits the next step. No per-Pod migration, automatic retry or downgrade exists.

4. Apply application components only after migrations succeed:

   ```bash
   for component in backend internal-backend scanner research frontend strategy-lab; do
     kubectl apply -f "k3s/apps/market-research/$component/" || break
     kubectl -n market-research rollout status "deployment/market-$component" --timeout=300s || break
   done
   kubectl -n market-research get pods,services,pvc,jobs
   kubectl -n market-research get endpointslices
   ```

   Stop and inspect failures before continuing. Verify upload write access and application
   behavior privately. Normal directory apply cannot start baseline: both Jobs remain
   .yaml.template and suspended. Nevertheless avoid blanket recursive deployment because
   it would bypass PostgreSQL/migration/application ordering.

5. Access Strategy Lab through loopback only:

   ```bash
   kubectl -n market-research port-forward --address=127.0.0.1 service/market-strategy-lab-service 8081:8080
   ```

   Open http://127.0.0.1:8081 on that host. For access from another computer, use your
   established authorized SSH local forwarding to pi-control-1 loopback; do not bind
   port-forward to all interfaces. Public frontend can similarly be forwarded:

   ```bash
   kubectl -n market-research port-forward --address=127.0.0.1 service/market-frontend-service 3000:8080
   ```

## Verification and logs

Operator commands, not executed during preparation:

```bash
kubectl -n market-research logs deployment/market-backend --tail=100
kubectl -n market-research logs deployment/market-internal-backend --tail=100
kubectl -n market-research logs deployment/market-scanner --tail=100
kubectl -n market-research logs deployment/market-research --tail=100
kubectl -n market-research logs deployment/market-frontend --tail=100
kubectl -n market-research logs deployment/market-strategy-lab --tail=100
kubectl -n market-research logs statefulset/market-postgres --tail=100
kubectl -n market-research get events --sort-by=.metadata.creationTimestamp
```

Check image pulls, readiness, Service endpoints, PVC binding/permissions, provider access,
and scanner progress. Inspect logs locally; never publish sensitive contents. Worker
rollout success means the process is running, not that data processing succeeded.

## Historical baseline — explicit separate operation

Do not execute this as part of normal deployment. The research worker already performs
nightly baseline catch-up and has no cross-process baseline lock. **The manual 20-day
baseline must not overlap nightly catch-up.** Coordinate a maintenance window; if stopping
research, allow any active catch-up to finish first and record/restore its prior replica
count after the manual run. Do not change its nightly defaults or add a CronJob.

Prerequisites: migrated database, DB/Alpaca Secrets, configmap, provider access, sufficient
capacity and recorded exact-date historical universe. An empty database cannot reconstruct
unrecorded prior universes. The Job uses no upload volume.

Only when explicitly requested and concurrency has been prevented:

```bash
kubectl create -f k3s/apps/market-research/jobs/historical-baseline.yaml.template && \
  kubectl -n market-research patch job market-historical-baseline --type=merge -p '{"spec":{"suspend":false}}'
kubectl -n market-research logs -f job/market-historical-baseline
kubectl -n market-research get job market-historical-baseline
```

Stop if creation fails. Inspect Complete/Failed conditions. **Job completion alone does
not prove requested coverage succeeded:** per-symbol failures can be persisted while the
process exits successfully. Inspect the private Strategy Lab historical-baseline report,
run status, completed-day coverage and failure counts before declaring success.
The report is also available at `/api/internal/strategy-lab/baseline` through its private
frontend proxy. Restore any deliberately paused research worker afterward.

Both Jobs use suspend=true, restartPolicy=Never and backoffLimit=0. For a later run, assign
a new unique metadata.name in the reviewed template and use that name in commands. Keep
the template suffix and suspension. Preserve prior results; do not blindly delete/retry
failed Jobs. The baseline has checkpoints but concurrent execution is not guaranteed safe.

## Updates and rollback

Record the previous known-good release. Replace every application image tag, including
both Job templates, with one successful immutable SHA release. Review and commit. Apply
configmap changes and recreate affected Pods deliberately; environment references do not
update running processes. Image changes naturally trigger Deployment replacement.

Back up PostgreSQL before migration. Follow release compatibility requirements, quiescing
old workloads if necessary. For a new migration execution use a unique Job name, explicitly
create/unsuspend it, and wait for success before applying the six application components.
Never include the baseline in this sequence.

Rollback: restore previous known-good image references in desired state and reapply the
six components, retaining compatible configuration and PVCs. **Database migrations are
NOT automatically rolled back. The previous application image must be compatible with
the current database schema.** Do not run migration/baseline Jobs as part of application
rollback or automatically downgrade the database.

## Remaining deployment checks

The intended release is `sha-3875a67`; confirm its successful publication and architecture
results before deployment. Create manual Secrets; verify default storage, upload ownership, backups and
Pi capacity. Plan baseline concurrency and review origins before any future LAN/public
access. Runtime commands, ports, Secret keys and configuration have no unresolved TODOs.
Local validation cannot establish registry availability, live admission behavior, PVC
permissions or application performance; no cluster was contacted.
