---
name: deploy-union-aws
description: Deploy a Union self-managed dataplane onto AWS (EKS) that registers to a managed Union control plane, without any Terraform. Provisions the infrastructure the Union AWS dataplane needs — an EKS Auto Mode cluster with IRSA, two S3 buckets (metadata + fast-registration), and backend/worker/fluentbit IAM roles — using the aws CLI, eksctl, kubectl and helm, then installs the Union dataplane chart. Use when the user wants a real Union dataplane on their own AWS account and has a control-plane host + operator OAuth client from Union (BYOC/self-managed). For a hosted-control-plane-free OSS Flyte on AWS use flyte-deploy-aws; for local/hermetic use deploy-flyte-k3d.
---

# Deploy a Union dataplane to AWS (EKS), no Terraform

Stand up a **Union self-managed dataplane** on AWS and register it to a **managed
Union control plane**. A dataplane is **not** flyte-binary: there is no in-cluster
control plane and no database — the cluster runs the union-operator, proxy, and
propeller, and connects out to the control plane over the operator tunnel.

This recreates, **with cloud CLIs instead of Terraform**, exactly what the
`selfmanaged/aws/dataplane` module provisions:

- an **EKS Auto Mode** cluster with an OIDC provider (IRSA),
- two **S3 buckets** — metadata (30-day lifecycle) + fast-registration,
- **IRSA roles** — backend (operator/propeller/proxy), worker (task pods), and
  fluentbit (logs) — each granting S3 read/write (+ ECR auth for the worker),
- optionally an **ECR** repository for the image builder,

then installs the `dataplane-crds` + `dataplane` helm charts pointed at the control
plane. Everything a user can run with `aws`, `eksctl`, `kubectl`, and `helm` — no
Terraform state, no module.

## Step 0: Inputs and prerequisites

Tools on PATH: `aws` (configured creds), `eksctl`, `kubectl`, `helm` (**3.18+**), `jq`.

**Ask the user for these inputs** (they come from Union / their control plane, not from
AWS) and keep them in shell variables:

```bash
export ORG_NAME=<union-org>                 # must match the control plane
export CLUSTER_NAME=<unique-cluster-id>     # e.g. my-company-aws-1
export CONTROLPLANE_HOST=<cp-hostname>      # bare host, no scheme/port (BYOC: given by Union)
export AUTH_CLIENT_ID=<operator-oauth-client-id>       # machine identity minted in the Union org
export AUTH_CLIENT_SECRET=<operator-oauth-client-secret>
export OIDC_S2S_SCOPE=""                    # Okta: empty. Entra ID: api://<app>/.default

export AWS_REGION=<region>                  # e.g. us-east-2
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export EKS_CLUSTER=$CLUSTER_NAME            # the EKS cluster name (reuse the cluster id)
export UNION_NS=union                       # namespace for Union components
```

The `AUTH_CLIENT_ID` / `AUTH_CLIENT_SECRET` are a **client-credentials OAuth app** the
user creates in their Union organization for this cluster's operator — they are inputs,
never generated here, and never committed.

> [!NOTE] Running under an isolated CI identity?
> If a functional-test agent runs this under a scoped role with a **permissions
> boundary**, every IAM role created below must carry `--permissions-boundary <arn>`, and
> the eksctl cluster config must set `iam.permissionsBoundary`. A boundary that denies
> creating unbounded roles will otherwise fail cluster/IRSA creation. Omit for an
> ordinary admin.

## Step 1: Create the EKS Auto Mode cluster (with IRSA)

Auto Mode manages compute, so there are no node groups to size. Enable the OIDC
provider so pods can assume IAM roles (IRSA).

```bash
cat > /tmp/eks-cluster.yaml <<EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ${EKS_CLUSTER}
  region: ${AWS_REGION}
  version: "1.32"
iam:
  withOIDC: true
autoModeConfig:
  enabled: true
EOF

eksctl create cluster -f /tmp/eks-cluster.yaml
aws eks update-kubeconfig --region "$AWS_REGION" --name "$EKS_CLUSTER"
kubectl get nodes

# The OIDC issuer host is needed for the IRSA trust policies below.
export OIDC_ISSUER=$(aws eks describe-cluster --name "$EKS_CLUSTER" \
  --query 'cluster.identity.oidc.issuer' --output text)
export OIDC_HOST=${OIDC_ISSUER#https://}
```

## Step 2: Create the two S3 buckets

Globally-unique names; block all public access; 30-day lifecycle on metadata (mirrors
the module).

```bash
export METADATA_BUCKET=${CLUSTER_NAME}-metadata-${AWS_ACCOUNT_ID}
export FAST_REGISTRATION_BUCKET=${CLUSTER_NAME}-fastreg-${AWS_ACCOUNT_ID}

for b in "$METADATA_BUCKET" "$FAST_REGISTRATION_BUCKET"; do
  if [ "$AWS_REGION" = "us-east-1" ]; then
    aws s3api create-bucket --bucket "$b" --region us-east-1
  else
    aws s3api create-bucket --bucket "$b" --region "$AWS_REGION" \
      --create-bucket-configuration LocationConstraint="$AWS_REGION"
  fi
  aws s3api put-public-access-block --bucket "$b" --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
done

aws s3api put-bucket-lifecycle-configuration --bucket "$METADATA_BUCKET" \
  --lifecycle-configuration '{"Rules":[{"ID":"expire-metadata","Status":"Enabled","Filter":{"Prefix":""},"Expiration":{"Days":30}}]}'
```

## Step 3: Create the IRSA roles

Each role trusts the cluster OIDC provider for service accounts in the Union namespace.
The subject is a **wildcard** (`system:serviceaccount:<ns>:*`) so it covers every chart
SA (and task pods in per-project namespaces) without coupling the trust to one SA name —
matching the module's low-privilege wildcard.

```bash
trust_policy() {   # $1 = namespace subject pattern
  cat <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Federated": "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/${OIDC_HOST}"},
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {"${OIDC_HOST}:aud": "sts.amazonaws.com"},
      "StringLike":   {"${OIDC_HOST}:sub": "system:serviceaccount:${UNION_NS}:$1"}
    }
  }]
}
EOF
}

s3_policy=$(cat <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject*","s3:PutObject*","s3:DeleteObject*","s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::${METADATA_BUCKET}","arn:aws:s3:::${METADATA_BUCKET}/*",
      "arn:aws:s3:::${FAST_REGISTRATION_BUCKET}","arn:aws:s3:::${FAST_REGISTRATION_BUCKET}/*"
    ]
  }]
}
EOF
)

# Backend + worker roles (fluentbit optional — same pattern, logs prefix only).
for r in backend worker; do
  trust_policy '*' > "/tmp/trust-$r.json"
  aws iam create-role --role-name "${CLUSTER_NAME}-${r}" \
    --assume-role-policy-document "file:///tmp/trust-$r.json"
  aws iam put-role-policy --role-name "${CLUSTER_NAME}-${r}" \
    --policy-name s3-access --policy-document "$s3_policy"
done

# The worker also needs ECR auth (image pulls + image-builder push).
aws iam put-role-policy --role-name "${CLUSTER_NAME}-worker" --policy-name ecr-auth \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":["ecr:GetAuthorizationToken","ecr:BatchGetImage","ecr:GetDownloadUrlForLayer","ecr:BatchCheckLayerAvailability","ecr:PutImage","ecr:InitiateLayerUpload","ecr:UploadLayerPart","ecr:CompleteLayerUpload"],"Resource":"*"}]}'

export BACKEND_IAM_ROLE_ARN=arn:aws:iam::${AWS_ACCOUNT_ID}:role/${CLUSTER_NAME}-backend
export WORKER_IAM_ROLE_ARN=arn:aws:iam::${AWS_ACCOUNT_ID}:role/${CLUSTER_NAME}-worker
```

Optional image-builder repository:

```bash
aws ecr create-repository --repository-name union --region "$AWS_REGION" || true
```

## Step 4: Install the Union dataplane

Assemble a values file from the collected inputs. **Write it 0600 and pass it with `-f`
— never `helm --set`**: numeric OAuth client IDs coerce to numbers under `--set`, secrets
containing `, . =` break it, and the secret would land on the command line.

```yaml
# values-aws.yaml — Union dataplane on AWS (0600; contains the operator secret)
provider: aws

global:
  CONTROLPLANE_HOST: <CONTROLPLANE_HOST>
  ORG_NAME: <ORG_NAME>
  CLUSTER_NAME: <CLUSTER_NAME>
  METADATA_BUCKET: <METADATA_BUCKET>
  FAST_REGISTRATION_BUCKET: <FAST_REGISTRATION_BUCKET>
  AWS_ACCOUNT_ID: "<AWS_ACCOUNT_ID>"
  AWS_REGION: <AWS_REGION>
  AWS_POD_IDENTITY_ANNOTATION_PREFIX: eks.amazonaws.com
  BACKEND_IAM_ROLE_ARN: <BACKEND_IAM_ROLE_ARN>
  WORKER_IAM_ROLE_ARN: <WORKER_IAM_ROLE_ARN>
  AUTH_CLIENT_ID: "<AUTH_CLIENT_ID>"
  OIDC_S2S_SCOPE: ""

storage:
  provider: aws
  authType: iam
  region: <AWS_REGION>

additionalServiceAccountAnnotations:
  eks.amazonaws.com/role-arn: <BACKEND_IAM_ROLE_ARN>
userRoleAnnotationKey: eks.amazonaws.com/role-arn
userRoleAnnotationValue: <WORKER_IAM_ROLE_ARN>

# Keep the single-replica operator-proxy (the DP<->CP tunnel) from being evicted by
# EKS Auto Mode / Karpenter consolidation, which would drop the tunnel.
proxy:
  podAnnotations:
    karpenter.sh/do-not-disrupt: "true"

secrets:
  admin:
    enable: true
    create: true
    clientId: "<AUTH_CLIENT_ID>"
    clientSecret: "<AUTH_CLIENT_SECRET>"

config:
  operator:
    apiKey:
      enabled: true
```

Render it from the environment (keeps the secret off your shell history and out of git),
then install:

```bash
umask 077
envsubst < values-aws.template.yaml > values-aws.yaml   # or hand-fill the placeholders above

helm repo add unionai https://unionai.github.io/helm-charts/
helm repo update

helm upgrade --install unionai-dataplane-crds unionai/dataplane-crds \
  --namespace "$UNION_NS" --create-namespace --wait

helm upgrade --install unionai-dataplane unionai/dataplane \
  --namespace "$UNION_NS" -f values-aws.yaml --wait --timeout 10m

kubectl -n "$UNION_NS" get pods
```

Expected pods: `operator-system-*`, `flytepropeller-system-*`, `proxy-system-*`,
`fluentbit-system-*`, `metrics-server-*`. On failure, dump diagnostics:
`kubectl -n "$UNION_NS" describe pod <pod>` and
`kubectl -n "$UNION_NS" logs <pod> --all-containers --tail=80`.

## Step 5: Verify the cluster registered with the control plane

The dataplane is healthy once the operator's tunnel is up and the control plane lists the
cluster ONLINE. Check the operator-proxy pod is running and the operator logs show a
successful control-plane connection:

```bash
kubectl -n "$UNION_NS" get pods -l app.kubernetes.io/name=union-operator-proxy
kubectl -n "$UNION_NS" logs deploy/operator-system --tail=50 | grep -i -E 'connected|registered|control.?plane' || true
```

Then confirm on the control plane (in the Union console or via the SDK/CLI against
`$CONTROLPLANE_HOST`) that `$CLUSTER_NAME` appears healthy. To run the shared functional
suite against this dataplane:

```bash
export FLYTE_FUNCTIONAL_ENDPOINT=dns:///${CONTROLPLANE_HOST}
export FLYTE_FUNCTIONAL_ORG=${ORG_NAME}
export FLYTE_FUNCTIONAL_PROJECT=flytesnacks
pytest --pyargs flyte_functional_tests -m integration
```

Unlike OSS flyte-binary, the full suite runs here (this is a managed Union backend), so
`trigger`, `reusable`, and `app` scenarios execute rather than skip.

## Teardown

Delete in reverse: helm releases, the buckets, the IAM roles, then the cluster (eksctl
also removes the VPC/OIDC it created).

```bash
helm -n "$UNION_NS" uninstall unionai-dataplane || true
helm -n "$UNION_NS" uninstall unionai-dataplane-crds || true
for r in backend worker; do
  aws iam delete-role-policy --role-name "${CLUSTER_NAME}-${r}" --policy-name s3-access || true
  aws iam delete-role-policy --role-name "${CLUSTER_NAME}-worker" --policy-name ecr-auth 2>/dev/null || true
  aws iam delete-role --role-name "${CLUSTER_NAME}-${r}" || true
done
for b in "$METADATA_BUCKET" "$FAST_REGISTRATION_BUCKET"; do
  aws s3 rm "s3://$b" --recursive || true; aws s3api delete-bucket --bucket "$b" || true
done
eksctl delete cluster --name "$EKS_CLUSTER" --region "$AWS_REGION"
```
