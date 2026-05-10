---
title: 'GitHub Actions Self-Hosted Runners no EKS com ARC'
lang: pt
uid: github-actions-arc-eks
date: 2026-05-09
permalink: /posts/2026/05/github-actions-self-hosted-runners-eks/
tags:
  - kubernetes
  - github-actions
  - devops
  - aws
  - helm
  - ci-cd
excerpt: 'Guia completo para configurar GitHub Actions self-hosted runners no Amazon EKS usando Actions Runner Controller (ARC), com imagem customizada no ECR, autoscaling e renovação automática de credenciais.'
---
# GitHub Actions Self-Hosted Runners no EKS com ARC

## O Problema

Os runners managed do GitHub Actions funcionam bem para projetos pequenos, mas em cenários de produção você esbarra em limitações:

- **Hardware fixo** — runners Linux padrão oferecem apenas 4 vCPUs e 16GB RAM. Se seu workflow faz build de imagens pesadas, roda testes de integração em paralelo ou precisa de mais memória para compilação, você fica refém do que o GitHub oferece
- **Custo elevado** com minutos de execução em repositórios privados
- **Falta de controle** sobre o ambiente (versões de ferramentas, dependências internas)
- **Latência** ao baixar imagens pesadas de registries privados
- **Segurança** — jobs rodando em infraestrutura compartilhada sem acesso à sua VPC

Com self-hosted runners no EKS, **você escolhe a máquina**. Precisa de CPU para compilação? Use `c6i.2xlarge`. Workflow pesado em memória? Use `r6i.xlarge`. E com node groups separados por tipo de workload, cada pipeline roda no hardware ideal sem pagar por recursos ociosos.

A solução: rodar seus próprios runners dentro do cluster Kubernetes na AWS, com autoscaling baseado na demanda real de jobs.

## TL;DR (Arquitetura Resumida)

```
GitHub Actions (webhook) → ARC Controller → Scale Set → Runner Pods (EKS)
                                                              ↓
                                                    Imagem customizada (ECR)
                                                              ↓
                                                    CronJob renova credenciais ECR a cada 5h
```

**Stack:**
- **EKS** — cluster Kubernetes na AWS
- **ARC** — Actions Runner Controller (autoscaling nativo)
- **ECR** — Registry privado para imagem customizada dos runners
- **Helm** — gerenciamento de releases
- **SOPS** — criptografia de secrets com KMS

---

## Índice

- [Pré-requisitos](#pré-requisitos)
- [Arquitetura](#arquitetura)
- [Passo 1: Imagem Customizada do Runner](#passo-1-imagem-customizada-do-runner)
- [Passo 2: Push para o ECR](#passo-2-push-para-o-ecr)
- [Passo 3: Instalar o ARC Controller](#passo-3-instalar-o-arc-controller)
- [Passo 4: Configurar o Runner Scale Set](#passo-4-configurar-o-runner-scale-set)
- [Passo 5: Renovação Automática de Credenciais ECR](#passo-5-renovação-automática-de-credenciais-ecr)
- [Passo 6: Automação com Script](#passo-6-automação-com-script)
- [Usando os Runners nos Workflows](#usando-os-runners-nos-workflows)
- [Troubleshooting](#troubleshooting)
- [Conclusão](#conclusão)

---

## Pré-requisitos

Antes de começar, você precisa ter:

- Cluster **EKS** rodando com `kubectl` configurado
- **Helm 3** instalado
- **AWS CLI** autenticada com permissões para ECR
- **GitHub PAT** (Personal Access Token) com scope `admin:org` ou `repo`
- **SOPS** configurado com KMS para gerenciar secrets (opcional, mas recomendado)

```bash
# Verificar conexão com o cluster
kubectl get nodes

# Se der erro de conexão, atualize o kubeconfig
aws eks update-kubeconfig --region us-east-1 --name seu-cluster-eks
```

---

## Arquitetura

```
┌─────────────────────────────────────────────────────────┐
│                        AWS (EKS)                        │
│                                                         │
│  ┌──────────────────┐    ┌───────────────────────────┐  │
│  │  arc-systems ns  │    │     arc-runners ns        │  │
│  │                  │    │                           │  │
│  │  ARC Controller  │───▶│  Runner Scale Set        │  │
│  │  (gerencia pods) │    │  ├─ runner-repo-1        │  │
│  │                  │    │  ├─ runner-repo-2        │  │
│  └──────────────────┘    │  └─ runner-org           │  │
│                          │                           │  │
│                          │  CronJob ECR (5h)        │  │
│                          │  (renova docker secret)   │  │
│                          └───────────────────────────┘  │
│                                                         │
│  ┌──────────────────┐                                   │
│  │       ECR        │                                   │
│  │ my-app-github-     │◀── Imagem customizada             │
│  │ action:latest    │    (tools + dependências)         │
│  └──────────────────┘                                   │
└─────────────────────────────────────────────────────────┘
         ▲
         │ webhooks (job queued/completed)
         │
┌────────┴────────┐
│  GitHub Actions  │
│  (seus repos)    │
└─────────────────┘
```

O fluxo funciona assim:

1. Um workflow é disparado no GitHub
2. O GitHub envia um webhook para o ARC Controller
3. O Controller escala o Runner Scale Set (cria pods)
4. O pod roda o job usando a imagem customizada do ECR
5. Ao terminar, o pod é destruído (scale to zero)

---

## Passo 1: Imagem Customizada do Runner

A imagem base do GitHub Actions Runner é mínima. Para rodar seus pipelines, você provavelmente precisa de ferramentas adicionais.

### Dockerfile

```dockerfile
FROM ghcr.io/actions/actions-runner:latest
USER root

RUN apt-get update && apt-get install -y \
    git gcc make wget curl jq netcat-openbsd

RUN chown root:runner -R /opt/ && chmod g+w /opt

# Instala o mesmo toolset usado nos runners hosted do GitHub (Ubuntu 24.04)
RUN wget https://raw.githubusercontent.com/actions/runner-images/main/images/ubuntu/toolsets/toolset-2404.json
RUN APT_PACKAGES=$(cat toolset-2404.json | jq -r \
    '.apt | [.vital_packages[], .common_packages[], .cmd_packages[]] | del(.[] | select(. == "lib32z1" or . == "netcat")) | join(" ")') \
    && apt-get update && apt-get install -y --no-install-recommends ${APT_PACKAGES}

USER runner
```

A estratégia aqui é reutilizar o **toolset oficial** do GitHub para runners Ubuntu 24.04. Isso garante compatibilidade com a maioria dos Actions que esperam ferramentas pré-instaladas (como `zip`, `unzip`, `python3`, etc).

### Build local para testar

```bash
docker build -t my-github-runner --pull --no-cache .
```

---

## Passo 2: Push para o ECR

Publique a imagem no seu registry privado:

```bash
# Variáveis
ECR_REGISTRY="xxxxxxxxxxxx.dkr.ecr.us-east-1.amazonaws.com"
ECR_REPOSITORY="my-github-runner"

# Autenticar no ECR
aws ecr get-login-password --region us-east-1 | \
    docker login --username AWS --password-stdin $ECR_REGISTRY

# Tag e push
docker tag my-github-runner:latest $ECR_REGISTRY/$ECR_REPOSITORY:latest
docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
```

Se o repositório ECR ainda não existir:

```bash
aws ecr create-repository \
    --repository-name my-github-runner \
    --region us-east-1
```

---

## Passo 3: Instalar o ARC Controller

O ARC Controller é o componente central que recebe webhooks do GitHub e gerencia o ciclo de vida dos runner pods.

```bash
NAMESPACE="arc-systems"
INSTALLATION_NAME="arc"

helm install $INSTALLATION_NAME \
    --namespace $NAMESPACE \
    --create-namespace \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

Verifique se o controller está rodando:

```bash
kubectl get pods -n arc-systems
```

Saída esperada:

```
NAME                                     READY   STATUS    RESTARTS   AGE
arc-gha-runner-scale-set-controller-xxx  1/1     Running   0          30s
```

---

## Passo 4: Configurar o Runner Scale Set

Aqui é onde configuramos os runners que vão executar os jobs. Cada Scale Set pode ser associado a um repositório ou organização.

### Arquivo de valores (`values.yml`) — partes relevantes

O arquivo completo com todas as opções disponíveis está na [documentação oficial do chart](https://github.com/actions/actions-runner-controller/blob/master/charts/gha-runner-scale-set/values.yaml). Aqui destaco o essencial:

```yaml
githubConfigUrl: "https://github.com/sua-org/seu-repo"
githubConfigSecret:
  github_token: ""

maxRunners: 10
minRunners: 0

containerMode:
  type: "dind"
```

A parte mais importante é o `template` do pod, onde você define imagem, node placement e acesso ao ECR:

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

### Pontos importantes da configuração

| Campo | Descrição |
|-------|-----------|
| `containerMode: dind` | Docker-in-Docker — permite que jobs façam `docker build` |
| `activeDeadlineSeconds: 3000` | Kill automático de pods travados (50 min) |
| `nodeSelector: ci-jobs` | Roda apenas em nodes dedicados para CI |
| `tolerations: ci-xlarge` | Permite usar nodes com taint específico |
| `imagePullSecrets` | Usa secret do ECR para pull da imagem |
| `minRunners: 0` | Scale to zero quando não há jobs |

### Instalar o Runner Scale Set

```bash
INSTALLATION_NAME="runner-seu-repo"
NAMESPACE="arc-runners"
GITHUB_CONFIG_URL="https://github.com/sua-org/seu-repo"

helm install "$INSTALLATION_NAME" \
    --namespace "$NAMESPACE" \
    --create-namespace \
    --values values.yml \
    --set githubConfigSecret.github_token="${GITHUB_PAT}" \
    --set githubConfigUrl="${GITHUB_CONFIG_URL}" \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

> **Dica:** Para não deixar o `GITHUB_PAT` em texto plano, use [SOPS](/posts/2026/05/gerenciando-secrets-sops/) com AWS KMS para criptografar seus arquivos de secrets.

---

## Passo 5: Renovação Automática de Credenciais ECR

Os tokens do ECR expiram a cada **12 horas**. Sem renovação automática, seus runners vão falhar ao tentar fazer pull da imagem.

A solução é um CronJob que roda a cada 5 horas e recria o secret `docker-registry`:

### O CronJob — a parte central

O job usa `alpine/k8s` (que já tem `aws` CLI e `kubectl`) para obter um novo token e recriar o secret:

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

O CronJob precisa de um `ServiceAccount` com permissão mínima para deletar e criar o secret específico:

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

As credenciais AWS ficam em um `Secret` separado (`ecr-registry-helper-secrets`) com `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` e `AWS_ACCOUNT`.

### Aplicar e verificar

```bash
# Aplicar todos os recursos
kubectl apply -f cronjob.yaml

# Verificar o CronJob
kubectl get cronjob -n arc-runners

# Testar manualmente (sem esperar o schedule)
kubectl create job --from=cronjob/ecr-registry-helper ecr-test -n arc-runners

# Ver logs
kubectl logs -n arc-runners -l job-name=ecr-test -f
```

### Por que o RBAC é mínimo

O Role concede apenas `delete` no secret específico `ecr-registry-credentials` e `create` genérico — o mínimo necessário para o ciclo delete/create. Nenhuma permissão extra.

---

## Passo 6: Automação com Script

Quando você tem múltiplos runners (um por repositório), atualizar manualmente cada um é inviável. Este script automatiza todo o fluxo:

```bash
#!/bin/bash
set -e

ECR_REGISTRY="xxxxxxxxxxxx.dkr.ecr.us-east-1.amazonaws.com"
ECR_REPOSITORY="my-github-runner"
IMAGE_TAG="latest"
AWS_REGION="us-east-1"
NAMESPACE="arc-runners"

# 1. Autenticar no ECR
echo "Authenticating with ECR..."
aws ecr get-login-password --region $AWS_REGION | \
    docker login --username AWS --password-stdin $ECR_REGISTRY

# 2. Build e push da imagem
echo "Building Docker image..."
docker build -t my-github-runner --pull --no-cache .
docker tag my-github-runner:latest $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

# 3. Atualizar ARC controller
echo "Updating ARC controller..."
helm upgrade --install arc \
    --namespace arc-systems \
    --create-namespace \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller

# 4. Atualizar todos os runners
RELEASES=$(helm list -n "$NAMESPACE" --short)

for RELEASE in $RELEASES; do
    echo "  → Updating release: $RELEASE"
    helm upgrade --install "$RELEASE" \
        --namespace "$NAMESPACE" \
        --reuse-values \
        --set githubConfigSecret.github_token="${GITHUB_PAT}" \
        oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
done

# 5. Resumo
echo ""
echo "Summary:"
helm list -n arc-runners -o json | \
    jq -r '["NAME","REVISION","APP_VERSION"], (.[] | [.name, (.revision|tostring), .app_version]) | @tsv' | \
    column -t
```

Execute com secrets via SOPS:

```bash
sops exec-env .env "./update-all-runners.sh"
```

---

## Usando os Runners nos Workflows

Depois de tudo configurado, usar é simples. No seu workflow, referencie o nome do runner scale set:

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  build:
    runs-on: runner-seu-repo  # nome do helm release
    steps:
      - uses: actions/checkout@v4

      - name: Build
        run: |
          docker build -t app .
          docker run app npm test
```

O `runs-on` deve corresponder ao nome da instalação Helm (o `INSTALLATION_NAME` usado no `helm install`).

### Runner por organização vs por repositório

| Scope | `githubConfigUrl` | Uso |
|-------|-------------------|-----|
| Repositório | `https://github.com/org/repo` | Jobs apenas desse repo |
| Organização | `https://github.com/org` | Qualquer repo da org pode usar |

Para organizações, o PAT precisa do scope `admin:org`.

---

## Troubleshooting

### Erro: `kubernetes cluster unreachable`

```
Error: kubernetes cluster unreachable: Get "http://localhost:8080/version": dial tcp 127.0.0.1:8080: connect: connection refused
```

**Solução:** Atualize o kubeconfig:

```bash
aws eks update-kubeconfig --region us-east-1 --name seu-cluster-eks
```

### Runners não aparecem no GitHub

```bash
# Verificar se o controller está rodando
kubectl get pods -n arc-systems

# Verificar logs do controller
kubectl logs -n arc-systems -l app.kubernetes.io/name=gha-runner-scale-set-controller

# Verificar se o listener está ativo
kubectl get pods -n arc-runners
```

### Pods com `ImagePullBackOff`

O secret do ECR provavelmente expirou:

```bash
# Forçar renovação manual
kubectl create job --from=cronjob/ecr-registry-helper ecr-renew-now -n arc-runners

# Verificar se o secret existe
kubectl get secret ecr-registry-credentials -n arc-runners
```

### Jobs travados (stuck)

O `activeDeadlineSeconds: 3000` no template mata pods após 50 minutos. Para limpar manualmente:

```bash
# Listar pods antigos
kubectl get pods -n arc-runners --sort-by=.metadata.creationTimestamp

# Deletar pods travados
kubectl delete pod <pod-name> -n arc-runners
```

### Verificar releases Helm

```bash
helm list -n arc-systems   # controller
helm list -n arc-runners   # runners
```

---

## Custos: Self-Hosted vs Managed

| | GitHub Hosted | Self-Hosted (EKS) |
|---|---|---|
| Hardware | Fixo: 4 vCPU / 16GB RAM | Você escolhe (c6i, r6i, m6i...) |
| Custo por minuto | $0.008 (Linux) | Custo da instância EC2 |
| Minutos gratuitos | 2000/mês (private) | Ilimitado |
| Scale to zero | N/A | Sim (paga só quando roda) |
| Acesso à VPC | Não | Sim |
| Imagem customizada | Limitado | Total controle |
| Latência de pull | Alta (registry público) | Baixa (ECR na mesma região) |
| GPU disponível | Não | Sim (p3, g5, etc) |

Para times com alto volume de CI/CD (>5000 min/mês) ou workflows que exigem hardware específico, self-hosted no EKS geralmente sai mais barato e mais rápido.

### Escolhendo o tipo de instância por workload

A grande vantagem é poder direcionar cada tipo de job para o hardware adequado usando `nodeSelector` e `tolerations`:

| Workload | Instância recomendada | Por que |
|----------|----------------------|---------|
| Build Docker / compilação | `c6i.2xlarge` (8 vCPU) | CPU-intensive, build paralelo |
| Testes de integração | `m6i.xlarge` (4 vCPU / 16GB) | Balanceado |
| Testes com banco pesado | `r6i.xlarge` (4 vCPU / 32GB) | Memory-intensive |
| ML / processamento de imagem | `g5.xlarge` (GPU) | Workloads com GPU |

No EKS, você cria **node groups separados** com labels e taints, e cada runner scale set aponta para o node group ideal:

```yaml
# Runner para builds pesados (CPU)
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
# Runner para testes com banco (memória)
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

Assim, um `docker build` pesado não compete por recursos com testes de integração, e você não paga por 32GB de RAM em jobs que só precisam de CPU.

---

## Conclusão

Com essa arquitetura você tem:

- **Scale to zero** — sem custos quando não há jobs rodando
- **Autoscaling** — o ARC cria pods sob demanda baseado na fila de jobs
- **Imagem customizada** — todas as ferramentas que seus pipelines precisam, pré-instaladas
- **Segurança** — runners dentro da VPC, com acesso a recursos internos
- **Automação** — credenciais ECR renovadas automaticamente, updates em batch via script

O setup inicial tem complexidade moderada, mas uma vez rodando, a manutenção é mínima. O script de update e o CronJob de credenciais cobrem os dois pontos que mais causam problemas no dia a dia.

**Próximo passo:** Clone este setup, adapte as variáveis para seu ambiente e comece com um runner para um repositório de teste. Depois é só replicar para os demais.

---

## Referências

- [Actions Runner Controller (ARC)](https://github.com/actions/actions-runner-controller)
- [ARC Helm Charts](https://github.com/actions/actions-runner-controller/pkgs/container/actions-runner-controller-charts%2Fgha-runner-scale-set)
- [GitHub Actions Self-Hosted Runners](https://docs.github.com/en/actions/hosting-your-own-runners)
- [Amazon EKS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/)
- [SOPS - Secrets OPerationS](https://github.com/getsops/sops)
