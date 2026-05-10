---
title: 'GitHub Actions Self-Hosted Runners on EKS with ARC'
lang: en
uid: github-actions-arc-eks
date: 2026-05-09
permalink: /en/posts/2026/05/github-actions-self-hosted-runners-eks/
tags:
  - kubernetes
  - github-actions
  - devops
  - aws
  - helm
  - ci-cd
excerpt: 'Complete guide to set up GitHub Actions self-hosted runners on Amazon EKS using Actions Runner Controller (ARC), with custom ECR image, autoscaling and automatic credential renewal.'
---
# GitHub Actions Self-Hosted Runners on EKS with ARC

## The Problem

GitHub Actions managed runners work fine for small projects, but in production scenarios you hit limitations:

- **Fixed hardware** — standard Linux runners offer only 4 vCPUs and 16GB RAM. If your workflow builds heavy images, runs integration tests in parallel or needs more memory for compilation, you're stuck with what GitHub offers
- **High cost** with execution minutes on private repositories
- **Lack of control** over the environment (tool versions, internal dependencies)
- **Latency** when pulling heavy images from private registries
- **Security** — jobs running on shared infrastructure without access to your VPC

With self-hosted runners on EKS, **you choose the machine**. Need CPU for compilation? Use `c6i.2xlarge`. Memory-heavy workflow? Use `r6i.xlarge`. And with node groups separated by workload type, each pipeline runs on ideal hardware without paying for idle resources.

The solution: run your own runners inside the Kubernetes cluster on AWS, with autoscaling based on real job demand.

## TL;DR (Architecture Summary)

```
GitHub Actions (webhook) → ARC Controller → Scale Set → Runner Pods (EKS)
                                                              ↓
                                                    Custom image (ECR)
                                                              ↓
                                                    CronJob renews ECR credentials every 5h
```

**Stack:**
- **EKS** — Kubernetes cluster on AWS
- **ARC** — Actions Runner Controller (native autoscaling)
- **ECR** — Private registry for custom runner image
- **Helm** — release management
- **SOPS** — secrets encryption with KMS

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Architecture](#architecture)
- [Step 1: Custom Runner Image](#step-1-custom-runner-image)
- [Step 2: Push to ECR](#step-2-push-to-ecr)
- [Step 3: Install the ARC Controller](#step-3-install-the-arc-controller)
- [Step 4: Configure the Runner Scale Set](#step-4-configure-the-runner-scale-set)
- [Step 5: Automatic ECR Credential Renewal](#step-5-automatic-ecr-credential-renewal)
- [Step 6: Automation with Script](#step-6-automation-with-script)
- [Using the Runners in Workflows](#using-the-runners-in-workflows)
- [Troubleshooting](#troubleshooting)
- [Conclusion](#conclusion)

---

## Prerequisites

Before starting, you need:

- **EKS** cluster running with `kubectl` configured
- **Helm 3** installed
- **AWS CLI** authenticated with ECR permissions
- **GitHub PAT** (Personal Access Token) with `admin:org` or `repo` scope
- **SOPS** configured with KMS to manage secrets (optional, but recommended)

```bash
# Verify cluster connection
kubectl get nodes

# If connection error, update kubeconfig
aws eks update-kubeconfig --region us-east-1 --name your-eks-cluster
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                        AWS (EKS)                        │
│                                                         │
│  ┌──────────────────┐    ┌───────────────────────────┐  │
│  │  arc-systems ns  │    │     arc-runners ns        │  │
│  │                  │    │                           │  │
│  │  ARC Controller  │───▶│  Runner Scale Set        │  │
│  │  (manages pods)  │    │  ├─ runner-repo-1        │  │
│  │                  │    │  ├─ runner-repo-2        │  │
│  └──────────────────┘    │  └─ runner-org           │  │
│                          │                           │  │
│                          │  CronJob ECR (5h)        │  │
│                          │  (renews docker secret)   │  │
│                          └───────────────────────────┘  │
│                                                         │
│  ┌──────────────────┐                                   │
│  │       ECR        │                                   │
│  │ my-app-github-     │◀── Custom image                   │
│  │ action:latest    │    (tools + dependencies)         │
│  └──────────────────┘                                   │
└─────────────────────────────────────────────────────────┘
         ▲
         │ webhooks (job queued/completed)
         │
┌────────┴────────┐
│  GitHub Actions  │
│  (your repos)    │
└─────────────────┘
```

The flow works like this:

1. A workflow is triggered on GitHub
2. GitHub sends a webhook to the ARC Controller
3. The Controller scales the Runner Scale Set (creates pods)
4. The pod runs the job using the custom ECR image
5. When finished, the pod is destroyed (scale to zero)

---

## Step 1: Custom Runner Image

The base GitHub Actions Runner image is minimal. To run your pipelines, you probably need additional tools.

### Dockerfile

```dockerfile
FROM ghcr.io/actions/actions-runner:latest
USER root

RUN apt-get update && apt-get install -y \
    git gcc make wget curl jq netcat-openbsd

RUN chown root:runner -R /opt/ && chmod g+w /opt

# Install the same toolset used in GitHub hosted runners (Ubuntu 24.04)
RUN wget https://raw.githubusercontent.com/actions/runner-images/main/images/ubuntu/toolsets/toolset-2404.json
RUN APT_PACKAGES=$(cat toolset-2404.json | jq -r \
    '.apt | [.vital_packages[], .common_packages[], .cmd_packages[]] | del(.[] | select(. == "lib32z1" or . == "netcat")) | join(" ")') \
    && apt-get update && apt-get install -y --no-install-recommends ${APT_PACKAGES}

USER runner
```

The strategy here is to reuse the **official toolset** from GitHub for Ubuntu 24.04 runners. This ensures compatibility with most Actions that expect pre-installed tools (like `zip`, `unzip`, `python3`, etc).

### Local build to test

```bash
docker build -t my-github-runner --pull --no-cache .
```

---

## Step 2: Push to ECR

Publish the image to your private registry:

```bash
# Variables
ECR_REGISTRY="xxxxxxxxxxxx.dkr.ecr.us-east-1.amazonaws.com"
ECR_REPOSITORY="my-github-runner"

# Authenticate with ECR
aws ecr get-login-password --region us-east-1 | \
    docker login --username AWS --password-stdin $ECR_REGISTRY

# Tag and push
docker tag my-github-runner:latest $ECR_REGISTRY/$ECR_REPOSITORY:latest
docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
```

If the ECR repository doesn't exist yet:

```bash
aws ecr create-repository \
    --repository-name my-github-runner \
    --region us-east-1
```

---

## Step 3: Install the ARC Controller

The ARC Controller is the central component that receives webhooks from GitHub and manages the runner pod lifecycle.

```bash
NAMESPACE="arc-systems"
INSTALLATION_NAME="arc"

helm install $INSTALLATION_NAME \
    --namespace $NAMESPACE \
    --create-namespace \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

Verify the controller is running:

```bash
kubectl get pods -n arc-systems
```

Expected output:

```
NAME                                     READY   STATUS    RESTARTS   AGE
arc-gha-runner-scale-set-controller-xxx  1/1     Running   0          30s
```

---

## Step 4: Configure the Runner Scale Set

This is where we configure the runners that will execute jobs. Each Scale Set can be associated with a repository or organization.

### Values file (`values.yml`) — relevant parts

The complete file with all available options is in the [official chart documentation](https://github.com/actions/actions-runner-controller/blob/master/charts/gha-runner-scale-set/values.yaml). Here I highlight the essentials:

```yaml
githubConfigUrl: "https://github.com/your-org/your-repo"
githubConfigSecret:
  github_token: ""

maxRunners: 10
minRunners: 0

containerMode:
  type: "dind"
```

The most important part is the pod `template`, where you define image, node placement and ECR access:

```yaml
template:
  spec:
    activeDeadlineSeconds: 3000
    nodeSelector:
      intent: "ci-jobs"
    tolerations:
      - key: "ci-xlarge"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
    containers:
      - name: runner
        image: xxxxxxxxxxxx.dkr.ecr.us-east-1.amazonaws.com/my-github-runner:latest
        command: ["/home/runner/run.sh"]
        env:
          - name: DOCKER_HOST
            value: unix:///var/run/docker.sock
    imagePullSecrets:
      - name: ecr-registry-credentials
```

### Key configuration points

| Field | Description |
|-------|-------------|
| `containerMode: dind` | Docker-in-Docker — allows jobs to run `docker build` |
| `activeDeadlineSeconds: 3000` | Auto-kill stuck pods (50 min) |
| `nodeSelector: ci-jobs` | Runs only on dedicated CI nodes |
| `tolerations: ci-xlarge` | Allows using nodes with specific taint |
| `imagePullSecrets` | Uses ECR secret for image pull |
| `minRunners: 0` | Scale to zero when there are no jobs |

### Install the Runner Scale Set

```bash
INSTALLATION_NAME="runner-your-repo"
NAMESPACE="arc-runners"
GITHUB_CONFIG_URL="https://github.com/your-org/your-repo"

helm install "$INSTALLATION_NAME" \
    --namespace "$NAMESPACE" \
    --create-namespace \
    --values values.yml \
    --set githubConfigSecret.github_token="${GITHUB_PAT}" \
    --set githubConfigUrl="${GITHUB_CONFIG_URL}" \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

> **Tip:** To avoid leaving `GITHUB_PAT` in plain text, use [SOPS](/en/posts/2026/05/managing-secrets-sops/) with AWS KMS to encrypt your secret files.

---

## Step 5: Automatic ECR Credential Renewal

ECR tokens expire every **12 hours**. Without automatic renewal, your runners will fail when trying to pull the image.

The solution is a CronJob that runs every 5 hours and recreates the `docker-registry` secret:

### The CronJob — the central piece

The job uses `alpine/k8s` (which already has `aws` CLI and `kubectl`) to obtain a new token and recreate the secret:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: ecr-registry-helper
  namespace: arc-runners
spec:
  schedule: "0 */5 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: sa-health-check
          containers:
          - name: ecr-registry-helper
            image: alpine/k8s:1.27.15
            envFrom:
              - secretRef:
                  name: ecr-registry-helper-secrets
              - configMapRef:
                  name: ecr-registry-helper-cm
            command:
              - /bin/bash
              - -c
              - |-
                ECR_TOKEN=$(aws ecr get-login-password --region ${AWS_REGION})
                kubectl delete secret --ignore-not-found $DOCKER_SECRET_NAME -n arc-runners
                kubectl create secret docker-registry $DOCKER_SECRET_NAME \
                  --docker-server=https://${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com \
                  --docker-username=AWS \
                  --docker-password="${ECR_TOKEN}" \
                  --namespace=arc-runners
          restartPolicy: Never
```

The CronJob needs a `ServiceAccount` with minimal permission to delete and create the specific secret:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: arc-runners
  name: role-ecr-secret-renewal
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["ecr-registry-credentials"]
  verbs: ["delete"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["create"]
```

AWS credentials are stored in a separate `Secret` (`ecr-registry-helper-secrets`) with `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` and `AWS_ACCOUNT`.

### Apply and verify

```bash
# Apply all resources
kubectl apply -f cronjob.yaml

# Verify the CronJob
kubectl get cronjob -n arc-runners

# Test manually (without waiting for the schedule)
kubectl create job --from=cronjob/ecr-registry-helper ecr-test -n arc-runners

# View logs
kubectl logs -n arc-runners -l job-name=ecr-test -f
```

### Why the RBAC is minimal

The Role grants only `delete` on the specific secret `ecr-registry-credentials` and generic `create` — the bare minimum needed for the delete/create cycle. No extra permissions.

---

## Step 6: Automation with Script

When you have multiple runners (one per repository), manually updating each one is impractical. This script automates the entire flow:

```bash
#!/bin/bash
set -e

ECR_REGISTRY="xxxxxxxxxxxx.dkr.ecr.us-east-1.amazonaws.com"
ECR_REPOSITORY="my-github-runner"
IMAGE_TAG="latest"
AWS_REGION="us-east-1"
NAMESPACE="arc-runners"

# 1. Authenticate with ECR
echo "Authenticating with ECR..."
aws ecr get-login-password --region $AWS_REGION | \
    docker login --username AWS --password-stdin $ECR_REGISTRY

# 2. Build and push image
echo "Building Docker image..."
docker build -t my-github-runner --pull --no-cache .
docker tag my-github-runner:latest $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

# 3. Update ARC controller
echo "Updating ARC controller..."
helm upgrade --install arc \
    --namespace arc-systems \
    --create-namespace \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller

# 4. Update all runners
RELEASES=$(helm list -n "$NAMESPACE" --short)

for RELEASE in $RELEASES; do
    echo "  → Updating release: $RELEASE"
    helm upgrade --install "$RELEASE" \
        --namespace "$NAMESPACE" \
        --reuse-values \
        --set githubConfigSecret.github_token="${GITHUB_PAT}" \
        oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
done

# 5. Summary
echo ""
echo "Summary:"
helm list -n arc-runners -o json | \
    jq -r '["NAME","REVISION","APP_VERSION"], (.[] | [.name, (.revision|tostring), .app_version]) | @tsv' | \
    column -t
```

Execute with secrets via SOPS:

```bash
sops exec-env .env "./update-all-runners.sh"
```

---

## Using the Runners in Workflows

After everything is configured, usage is simple. In your workflow, reference the runner scale set name:

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  build:
    runs-on: runner-your-repo  # helm release name
    steps:
      - uses: actions/checkout@v4

      - name: Build
        run: |
          docker build -t app .
          docker run app npm test
```

The `runs-on` must match the Helm installation name (the `INSTALLATION_NAME` used in `helm install`).

### Runner per organization vs per repository

| Scope | `githubConfigUrl` | Use |
|-------|-------------------|-----|
| Repository | `https://github.com/org/repo` | Jobs only from this repo |
| Organization | `https://github.com/org` | Any repo in the org can use it |

For organizations, the PAT needs the `admin:org` scope.

---

## Troubleshooting

### Error: `kubernetes cluster unreachable`

```
Error: kubernetes cluster unreachable: Get "http://localhost:8080/version": dial tcp 127.0.0.1:8080: connect: connection refused
```

**Solution:** Update kubeconfig:

```bash
aws eks update-kubeconfig --region us-east-1 --name your-eks-cluster
```

### Runners not showing up on GitHub

```bash
# Verify controller is running
kubectl get pods -n arc-systems

# Check controller logs
kubectl logs -n arc-systems -l app.kubernetes.io/name=gha-runner-scale-set-controller

# Verify listener is active
kubectl get pods -n arc-runners
```

### Pods with `ImagePullBackOff`

The ECR secret probably expired:

```bash
# Force manual renewal
kubectl create job --from=cronjob/ecr-registry-helper ecr-renew-now -n arc-runners

# Verify secret exists
kubectl get secret ecr-registry-credentials -n arc-runners
```

### Stuck jobs

The `activeDeadlineSeconds: 3000` in the template kills pods after 50 minutes. To clean up manually:

```bash
# List old pods
kubectl get pods -n arc-runners --sort-by=.metadata.creationTimestamp

# Delete stuck pods
kubectl delete pod <pod-name> -n arc-runners
```

### Check Helm releases

```bash
helm list -n arc-systems   # controller
helm list -n arc-runners   # runners
```

---

## Costs: Self-Hosted vs Managed

| | GitHub Hosted | Self-Hosted (EKS) |
|---|---|---|
| Hardware | Fixed: 4 vCPU / 16GB RAM | You choose (c6i, r6i, m6i...) |
| Cost per minute | $0.008 (Linux) | EC2 instance cost |
| Free minutes | 2000/month (private) | Unlimited |
| Scale to zero | N/A | Yes (pay only when running) |
| VPC access | No | Yes |
| Custom image | Limited | Full control |
| Pull latency | High (public registry) | Low (ECR in same region) |
| GPU available | No | Yes (p3, g5, etc) |

For teams with high CI/CD volume (>5000 min/month) or workflows requiring specific hardware, self-hosted on EKS is generally cheaper and faster.

### Choosing instance type by workload

The big advantage is being able to direct each job type to appropriate hardware using `nodeSelector` and `tolerations`:

| Workload | Recommended instance | Why |
|----------|---------------------|-----|
| Docker build / compilation | `c6i.2xlarge` (8 vCPU) | CPU-intensive, parallel build |
| Integration tests | `m6i.xlarge` (4 vCPU / 16GB) | Balanced |
| Heavy database tests | `r6i.xlarge` (4 vCPU / 32GB) | Memory-intensive |
| ML / image processing | `g5.xlarge` (GPU) | GPU workloads |

On EKS, you create **separate node groups** with labels and taints, and each runner scale set points to the ideal node group:

```yaml
# Runner for heavy builds (CPU)
template:
  spec:
    nodeSelector:
      intent: "ci-cpu-heavy"
    tolerations:
      - key: "ci-cpu-heavy"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
```

```yaml
# Runner for database tests (memory)
template:
  spec:
    nodeSelector:
      intent: "ci-memory"
    tolerations:
      - key: "ci-memory"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
```

This way, a heavy `docker build` doesn't compete for resources with integration tests, and you don't pay for 32GB of RAM on jobs that only need CPU.

---

## Conclusion

With this architecture you have:

- **Scale to zero** — no costs when no jobs are running
- **Autoscaling** — ARC creates pods on demand based on job queue
- **Custom image** — all tools your pipelines need, pre-installed
- **Security** — runners inside the VPC, with access to internal resources
- **Automation** — ECR credentials automatically renewed, batch updates via script

The initial setup has moderate complexity, but once running, maintenance is minimal. The update script and credentials CronJob cover the two points that cause the most day-to-day issues.

**Next step:** Clone this setup, adapt the variables for your environment and start with a runner for a test repository. Then just replicate for the rest.

---

## References

- [Actions Runner Controller (ARC)](https://github.com/actions/actions-runner-controller)
- [ARC Helm Charts](https://github.com/actions/actions-runner-controller/pkgs/container/actions-runner-controller-charts%2Fgha-runner-scale-set)
- [GitHub Actions Self-Hosted Runners](https://docs.github.com/en/actions/hosting-your-own-runners)
- [Amazon EKS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/)
- [SOPS - Secrets OPerationS](https://github.com/getsops/sops)
