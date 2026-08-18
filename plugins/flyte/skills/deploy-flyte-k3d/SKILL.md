---
name: deploy-flyte-k3d
description: Deploy a complete, self-contained Flyte OSS stack (flyte-binary + in-cluster PostgreSQL + in-cluster RustFS object store) onto a local k3d cluster, with nothing hosted and no cloud credentials. Mirrors the helm-charts integration-checks k3d infrastructure (single-node k3d with an embedded registry + RustFS). Use when the user wants a hermetic, throwaway Flyte on k3d for evaluation, CI, or functional testing — everything runs inside the cluster; for a kind cluster or a hosted-PostgreSQL/S3 topology use deploy-flyte-kind instead, and for a real cloud deployment use the AWS/GCP skills.
---

# Deploy Flyte OSS to a k3d cluster (hermetic)

Stand up Flyte on a [k3d](https://k3d.io) cluster with **everything in-cluster**:
the [flyte-binary](https://github.com/flyteorg/flyte) control plane, a **PostgreSQL**
database, and a **RustFS** object store (S3-API compatible). No hosted database, no
cloud bucket, no credentials to collect — the whole stack is a throwaway suitable for
evaluation, CI, and functional testing. For **evaluation only**: no TLS, no auth,
static in-cluster credentials.

This mirrors the **helm-charts integration-checks** k3d leg
(`tools/dataplane/k3d/up.sh` + `rustfs.yaml`): a single-node k3d cluster with an
embedded registry and an in-cluster RustFS object store. The difference is topology —
that leg installs the **Union dataplane** (which registers to a control plane and uses
the CP's database), whereas this skill installs **OSS flyte-binary**, which is its own
control plane and therefore **also needs an in-cluster PostgreSQL**.

Everything below is a **phase** (idempotent, re-runnable), matching the integration-checks
structure: `cluster → storage → database → install → verify → teardown`.

## Step 0: Prerequisites

Required on PATH: `k3d`, `kubectl`, `helm`, `docker`.

- **helm 3.17.x** is what integration-checks pins; newer majors can hit helm's 1 MB
  release-secret limit on large charts. flyte-binary is small, so any helm 3 works, but
  prefer 3.17.x for parity. **helm 4** is a new major and untested against this chart —
  if the Step 4 install errors, fall back to helm 3.17.x before debugging further.
- Check for an existing cluster before creating one:

```bash
k3d cluster list
kubectl config current-context   # k3d-<name> if a cluster is already selected
```

If `k3d-flyte-oss` already exists and you want a clean slate, tear it down first
(`k3d cluster delete flyte-oss`). Otherwise the phases below are safe to re-run.

Fixed names used throughout (override via the environment if you must):

| Thing | Value |
| --- | --- |
| k3d cluster | `flyte-oss` |
| namespace (all components) | `flyte` |
| object-store bucket | `union-data` |
| RustFS access / secret key | `rustfsadmin` / `rustfsadmin` |
| PostgreSQL db / user / password | `flyte` / `flyte` / `flyte` |

## Step 1: Create the k3d cluster (phase `cluster`)

Single-node cluster with an embedded HTTP registry — the same shape as the CI leg. The
retry guards a transient blip during node bring-up.

```bash
# Pin kubectl/helm to a DEDICATED, throwaway kubeconfig for this cluster. The
# skill never reads or mutates ~/.kube/config — so it can't hijack your current
# context and there is no multi-file KUBECONFIG ambiguity (the exact thing that
# breaks a plain `kubectl` on a machine with other clusters). The env var resets
# in a new shell, so EVERY code block below re-exports this same line; the file
# it points at persists, so kubectl stays pinned to k3d across phases.
export KUBECONFIG=/tmp/flyte-oss.kubeconfig

cat > /tmp/k3d-registry.yaml <<'EOF'
mirrors:
  "k3d-registry:5000":
    endpoint:
      - "http://k3d-registry:5000"
EOF

if ! k3d cluster list 2>/dev/null | grep -q '^flyte-oss '; then
  n=0
  until k3d cluster create flyte-oss \
      --registry-create k3d-registry:0.0.0.0:5001 \
      --registry-config /tmp/k3d-registry.yaml --wait --timeout 180s; do
    n=$((n+1)); [ "$n" -ge 3 ] && { echo "k3d create failed after $n attempts"; exit 1; }
    echo "cluster create failed (attempt $n) — cleaning up and retrying"
    k3d cluster delete flyte-oss || true; sleep 10
  done
fi

# Write THIS cluster's kubeconfig into the dedicated file. `k3d kubeconfig get`
# prints to stdout (no default-kubeconfig write, so no multi-file error), and
# this also covers the reuse path where the cluster already existed.
k3d kubeconfig get flyte-oss > "$KUBECONFIG"
[ "$(kubectl config current-context)" = "k3d-flyte-oss" ] || {
  echo "ERROR: could not select k3d-flyte-oss (got '$(kubectl config current-context)')"; exit 1; }

kubectl create namespace flyte --dry-run=client -o yaml | kubectl apply -f -
kubectl wait --for=condition=Ready nodes --all --timeout=120s
```

> [!IMPORTANT] Every phase re-exports `KUBECONFIG`
> `export KUBECONFIG=/tmp/flyte-oss.kubeconfig` is the first line of every code
> block below. The env var resets in a fresh shell but the file persists, so this
> keeps kubectl/helm pinned to the k3d cluster **without ever touching
> `~/.kube/config`**. Run the phases in one shell and the repeated export is a
> harmless no-op.

## Step 2: Deploy the object store — RustFS (phase `storage`)

RustFS is MinIO-API-compatible; flyte-binary talks to it over the S3 protocol. This is
the **same manifest** the integration-checks k3d leg uses (pinned image + digest for
reproducibility), applied into the `flyte` namespace.

```bash
export KUBECONFIG=/tmp/flyte-oss.kubeconfig   # pin to the k3d cluster (see Step 1)

cat > /tmp/rustfs.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rustfs
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rustfs
  template:
    metadata:
      labels:
        app: rustfs
    spec:
      containers:
        - name: rustfs
          image: rustfs/rustfs:1.0.0-beta.8@sha256:fa19210ac4697c79d7ccca1ec9b0eb91aebacc6691991ffb14014bb3c67e6cc3
          env:
            - name: RUSTFS_ACCESS_KEY
              value: rustfsadmin
            - name: RUSTFS_SECRET_KEY
              value: rustfsadmin
            - name: MINIO_ROOT_USER
              value: rustfsadmin
            - name: MINIO_ROOT_PASSWORD
              value: rustfsadmin
          args: ["server", "/data", "--address", ":9000", "--console-address", ":9001"]
          ports:
            - name: s3api
              containerPort: 9000
            - name: console
              containerPort: 9001
          readinessProbe:
            httpGet:
              path: /health/live
              port: 9000
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 24
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: rustfs
spec:
  selector:
    app: rustfs
  ports:
    - name: s3api
      port: 9000
      targetPort: 9000
    - name: console
      port: 9001
      targetPort: 9001
EOF

kubectl apply -n flyte -f /tmp/rustfs.yaml
kubectl wait --for=condition=Available deploy/rustfs -n flyte --timeout=180s
```

**Create the bucket.** flyte-binary does not create the bucket; make it with the MinIO
client over a short-lived port-forward (integration-checks does the same):

```bash
export KUBECONFIG=/tmp/flyte-oss.kubeconfig   # pin to the k3d cluster (see Step 1)

kubectl -n flyte port-forward svc/rustfs 9000:9000 >/tmp/pf-rustfs.log 2>&1 &
PF=$!; sleep 3
# `mc` (minio-client). If absent: curl -fsSL https://dl.min.io/client/mc/release/$(uname -s | tr A-Z a-z)-amd64/mc -o /usr/local/bin/mc && chmod +x /usr/local/bin/mc
mc alias set k3dstore http://localhost:9000 rustfsadmin rustfsadmin
mc mb --ignore-existing k3dstore/union-data
kill $PF 2>/dev/null || true
```

## Step 3: Deploy the database — PostgreSQL (phase `database`)

The OSS-only piece the dataplane leg lacks. A single-replica in-cluster PostgreSQL,
mirroring the RustFS manifest style (pinned image, readiness probe, `emptyDir` — this is
throwaway state). flyte-binary's `wait-for-db` init container blocks until this is up.

```bash
export KUBECONFIG=/tmp/flyte-oss.kubeconfig   # pin to the k3d cluster (see Step 1)

cat > /tmp/postgres.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flyte-postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flyte-postgres
  template:
    metadata:
      labels:
        app: flyte-postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16.4
          env:
            - name: POSTGRES_DB
              value: flyte
            - name: POSTGRES_USER
              value: flyte
            - name: POSTGRES_PASSWORD
              value: flyte
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          ports:
            - name: postgres
              containerPort: 5432
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "flyte", "-d", "flyte"]
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 24
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: data
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: flyte-postgres
spec:
  selector:
    app: flyte-postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
EOF

kubectl apply -n flyte -f /tmp/postgres.yaml
kubectl wait --for=condition=Available deploy/flyte-postgres -n flyte --timeout=180s
```

> Pinning a `postgres` digest (`postgres:16.4@sha256:…`) makes CI reproducible, but only
> add a digest you have verified resolves for your architecture — a wrong/stale digest
> leaves the pod `ImagePullBackOff` and flyte stuck in `Init`.

## Step 4: Write the values file and install flyte-binary (phase `install`)

Point flyte-binary at the two in-cluster services. **The endpoints below are
in-cluster DNS** (`flyte-postgres:5432`, `http://rustfs:9000`) — the control plane and
the task pods both run inside the cluster, so they resolve these directly. (The laptop
SDK is different — see the endpoint-asymmetry note in Step 5.)

```yaml
# values-k3d.yaml — hermetic OSS flyte-binary on k3d
fullnameOverride: flyte

configuration:
  database:
    postgres:
      host: flyte-postgres
      port: 5432
      dbname: flyte
      username: flyte
      password: flyte
      options: "sslmode=disable"

  storage:
    metadataContainer: union-data
    userDataContainer: union-data
    provider: s3
    providerConfig:
      s3:
        endpoint: http://rustfs:9000
        region: us-east-1
        authType: accesskey
        accessKey: rustfsadmin
        secretKey: rustfsadmin
        disableSSL: true
        v2Signing: false
        secure: false

  inline:
    runs:
      storagePrefix: s3://union-data
    plugins:
      k8s:
        default-env-vars:
          - _U_EP_OVERRIDE: "flyte-http.flyte:8090"
          - _U_INSECURE: "true"
          - _U_USE_ACTIONS: "1"
          - FLYTE_AWS_ENDPOINT: "http://rustfs.flyte:9000"
          - FLYTE_AWS_ACCESS_KEY_ID: "rustfsadmin"
          - FLYTE_AWS_SECRET_ACCESS_KEY: "rustfsadmin"
          - AWS_REGION: "us-east-1"

serviceAccount:
  create: true
  annotations: {}

ingress:
  create: false
```

**The three traps this file defuses** (the API comes up without them, but every *task*
fails):

1. `runs.storagePrefix` defaults to `s3://flyte-data`; task I/O and `error.pb` writes go
   to a nonexistent bucket → `403 AccessDenied`. Point it at `s3://union-data`.
2. The `storage.*` block configures **only the control plane**; task pods get **no**
   object-store credentials and fall back to the AWS metadata endpoint →
   `Generic S3 error … http://169.254.169.254/...`. Inject them via
   `plugins.k8s.default-env-vars` (the `FLYTE_AWS_*` vars).
3. `default-env-vars` **replaces** the chart default outright, so the three `_U_*`
   control-plane callback vars must be repeated — dropping them breaks
   task→control-plane callbacks. The task-pod `FLYTE_AWS_ENDPOINT` uses the **fully
   qualified** `rustfs.flyte:9000` because task pods may land in other namespaces.

Install:

```bash
export KUBECONFIG=/tmp/flyte-oss.kubeconfig   # pin to the k3d cluster (see Step 1)

helm repo add flyteorg https://flyteorg.github.io/flyte
helm repo update flyteorg   # scope to THIS repo — a bare `helm repo update` refreshes
                            # every repo on the machine and aborts if any unrelated one
                            # 404s (e.g. a stale kubernetes-dashboard repo)
helm upgrade --install flyte flyteorg/flyte-binary -n flyte -f values-k3d.yaml --wait --timeout 8m

kubectl -n flyte rollout status deploy/flyte
kubectl -n flyte get pods
```

If a pod is stuck in `Init`, the `wait-for-db` init container is blocking on PostgreSQL
— check `kubectl -n flyte logs <pod> -c wait-for-db`. On any install failure, dump
diagnostics: `kubectl -n flyte describe pod <pod>` and
`kubectl -n flyte logs <pod> --all-containers --tail=80`.

## Step 5: Verify access (phase `verify`)

Expose the API and the object store to the machine running the SDK/CLI:

```bash
export KUBECONFIG=/tmp/flyte-oss.kubeconfig   # pin to the k3d cluster (see Step 1)

# Start DURABLE port-forwards: nohup + disown so they survive this shell exiting
# and are still alive for Step 6 (which runs in a separate shell). Kill any stale
# ones first so re-running this phase doesn't stack duplicate forwards on the port.
pkill -f 'port-forward.*flyte-http' 2>/dev/null || true
pkill -f 'port-forward.*svc/rustfs' 2>/dev/null || true
pkill -f 'port-forward.*service/rustfs' 2>/dev/null || true
nohup kubectl -n flyte port-forward service/flyte-http 8090:8090 >/tmp/pf-flyte.log 2>&1 & disown
nohup kubectl -n flyte port-forward service/rustfs   9000:9000 >/tmp/pf-rustfs.log 2>&1 & disown
sleep 3

curl -s -X POST \
  http://localhost:8090/flyteidl2.project.ProjectService/ListProjects \
  -H 'Content-Type: application/json' -d '{}'
```

A JSON response (not a connection error) confirms Flyte is up and talking to its
database.

> [!IMPORTANT] Object-store endpoint asymmetry
> The control plane and task pods reach RustFS at the **in-cluster** DNS
> `http://rustfs:9000`, but the **laptop SDK** cannot resolve that — it uploads the code
> bundle through the **port-forward** at `http://localhost:9000`. Same bucket
> (`union-data`), same keys, two different hostnames. This is the one thing that differs
> from a hosted-bucket deployment (where a single public endpoint works everywhere).

**SDK config** to submit runs (the SDK reads a project-local `.flyte/config.yaml` before
`~/.flyte/config.yaml`):

```yaml
admin:
  endpoint: dns:///localhost:8090
  insecure: True
task:
  org: local
  domain: development
  project: flytesnacks
```

For the SDK's object-store client, set the S3 endpoint + credentials to the
port-forward so `flyte run` can upload the code bundle:

```bash
export FLYTE_AWS_ENDPOINT=http://localhost:9000
export FLYTE_AWS_ACCESS_KEY_ID=rustfsadmin
export FLYTE_AWS_SECRET_ACCESS_KEY=rustfsadmin
export AWS_REGION=us-east-1
```

> [!NOTE] `helm upgrade` drops the port-forwards
> Every `helm upgrade` rolls the flyte pod and kills the `flyte-http` port-forward
> ("Flyte system is currently unavailable"). Restart both port-forwards after an upgrade.

## Step 6: Run the functional tests (optional, phase `functional`)

Once the endpoint answers, the shared `flyte-functional-tests` suite is the deterministic
verdict. Point it at the port-forwarded API and object store; the OSS-only scenarios run
and the Union-platform scenarios (`trigger`, `reusable`, `app`) auto-skip on a flyte-binary
backend.

```bash
export FLYTE_FUNCTIONAL_ENDPOINT=dns:///localhost:8090
export FLYTE_FUNCTIONAL_PROJECT=flytesnacks
pytest --pyargs flyte_functional_tests -m integration
```

## Teardown (phase `teardown`)

Deleting the k3d cluster removes flyte, PostgreSQL, RustFS, and the embedded registry in
one shot — the state is all `emptyDir`, so nothing survives.

```bash
pkill -f 'port-forward.*flyte' || true
pkill -f 'port-forward.*rustfs' || true
k3d cluster delete flyte-oss
```
