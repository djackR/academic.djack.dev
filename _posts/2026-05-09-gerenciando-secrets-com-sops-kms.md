---
title: 'Gerenciando Secrets com SOPS: KMS, GCP e GPG'
date: 2026-05-09
permalink: /posts/2026/05/gerenciando-secrets-sops/
lang: pt
tags:
  - devops
  - seguranca
  - aws
  - gcp
  - sops
  - secrets
  - gpg
excerpt: 'Aprenda a criptografar e gerenciar secrets de projetos usando SOPS. Suporta AWS KMS, GCP Cloud KMS, Azure Key Vault, age e GPG — escolha o backend que faz sentido para seu time.'
---
# Gerenciando Secrets com SOPS: KMS, GCP e GPG

Sexta-feira, 17h. Você termina aquela feature, roda um `git add .` satisfeito, commit, push. Vai pegar um cafe. No caminho de volta pro computador, aquele frio na barriga: "espera... o `.env` estava no staged?". Abre o GitHub, confere o commit e la esta — `AWS_SECRET_ACCESS_KEY` em texto plano, publicado para o mundo. O resto da sexta vira um incidente de seguranca: revogar chaves, rotacionar credenciais, avisar o time, rezar para ninguem ter visto.

Se voce ja viveu isso (ou vive com medo de viver), o **SOPS** e para voce. Ele permite commitar seus arquivos de secrets **criptografados** direto no repositorio. Sem medo. Sem incidente. Sem sexta-feira arruinada.

---

## O Problema

Toda aplicacao tem secrets: tokens de API, credenciais de banco, chaves privadas. O desafio e como compartilha-las com o time sem:

- Commitar `.env` em texto plano no repositorio
- Depender de alguem mandar credenciais por Slack/email
- Perder acesso quando alguem sai do time
- Manter um vault complexo para projetos pequenos/medios

O **SOPS** (Secrets OPerationS) resolve isso: ele criptografa apenas os **valores** dos seus arquivos de secrets, mantendo as chaves legiveis. Combinado com um KMS (AWS, GCP, Azure) ou GPG, qualquer pessoa autorizada pode decriptar — sem compartilhar senha.

## TL;DR

```bash
# Instalar SOPS
# https://github.com/getsops/sops/releases

# Configurar a chave KMS
export SOPS_KMS_ARN="arn:aws:kms:us-east-1:xxxxxxxxxxxx:alias/sua-chave"

# Criptografar um arquivo existente
sops encrypt .env > .env.enc

# Editar secrets (decripta, abre editor, re-criptografa ao salvar)
sops edit .env

# Usar secrets como variáveis de ambiente (sem arquivo decriptado em disco)
sops exec-env .env "helm install ..."
```

---

## Indice

- [O que e SOPS](#o-que-é-sops)
- [Instalacao](#instalação)
- [Configuracao Inicial](#configuração-inicial)
- [Workflow no Dia a Dia](#workflow-no-dia-a-dia)
- [Formatos Suportados](#formatos-suportados)
- [Backends de Criptografia](#backends-de-criptografia)
- [Usando com Helm e Kubernetes](#usando-com-helm-e-kubernetes)
- [Boas Praticas](#boas-práticas)
- [Conclusao](#conclusão)

---

## O que é SOPS

[SOPS](https://github.com/getsops/sops) é uma ferramenta da Mozilla (agora mantida pela CNCF) que criptografa arquivos de configuração. O diferencial:

```bash
# Arquivo original (.env)
GITHUB_PAT=ghp_abc123xyz789
AWS_SECRET_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCY
DB_PASSWORD=super-secret-123
```

```bash
# Após criptografar com SOPS
GITHUB_PAT=ENC[AES256_GCM,data:k8vM2n...,type:str]
AWS_SECRET_KEY=ENC[AES256_GCM,data:pQ7xR...,type:str]
DB_PASSWORD=ENC[AES256_GCM,data:mN3kL...,type:str]
```

As **chaves** (`GITHUB_PAT`, `AWS_SECRET_KEY`) continuam legíveis — você sabe o que o arquivo contém sem decriptar. Apenas os **valores** são criptografados.

Isso significa que:
- O arquivo pode ser commitado no repositório com segurança
- Code review consegue ver quais secrets foram adicionadas/removidas
- `git diff` mostra mudanças estruturais sem expor valores

---

## Por que usar um KMS (Key Management Service)

O SOPS suporta vários backends. Os cloud KMS (AWS, GCP, Azure) compartilham vantagens sobre GPG/age:

- **Controle via IAM** — quem pode decriptar é controlado por políticas do cloud provider
- **Auditoria** — cada uso da chave é registrado (CloudTrail, Cloud Audit Logs)
- **Rotação automática** — o provider rotaciona a chave sem quebrar arquivos existentes
- **Sem segredo compartilhado** — não precisa distribuir uma chave privada entre o time

Se você não usa nenhum cloud provider, GPG e age funcionam perfeitamente — só exigem mais gestão manual de chaves.

---

## Instalação

### SOPS

Baixe o binário da [página de releases](https://github.com/getsops/sops/releases):

```bash
# Linux (amd64)
curl -LO https://github.com/getsops/sops/releases/download/v3.9.4/sops-v3.9.4.linux.amd64
mv sops-v3.9.4.linux.amd64 /usr/local/bin/sops
chmod +x /usr/local/bin/sops

# macOS (via Homebrew)
brew install sops

# Verificar
sops --version
```

### AWS CLI

Você precisa do AWS CLI configurado com credenciais que tenham acesso à chave KMS:

```bash
aws sts get-caller-identity  # Verifica se está autenticado
```

---

## Configuração Inicial

### 1. Criar (ou identificar) a chave KMS

Se você já tem uma chave KMS, pegue o ARN ou alias. Se não:

```bash
aws kms create-key --description "SOPS encryption key"

# Criar um alias para facilitar
aws kms create-alias \
    --alias-name alias/sops-key \
    --target-key-id <key-id-retornado>
```

### 2. Exportar o ARN

```bash
export SOPS_KMS_ARN="arn:aws:kms:us-east-1:xxxxxxxxxxxx:alias/sops-key"
```

### 3. Criar o arquivo `.sops.yaml` (opcional, recomendado)

Na raiz do projeto, crie um `.sops.yaml` para não precisar passar o ARN toda vez:

```yaml
creation_rules:
  - path_regex: \.env$
    kms: "arn:aws:kms:us-east-1:xxxxxxxxxxxx:alias/sops-key"
  - path_regex: secrets\.ya?ml$
    kms: "arn:aws:kms:us-east-1:xxxxxxxxxxxx:alias/sops-key"
```

Com isso, qualquer arquivo `.env` ou `secrets.yml` será automaticamente criptografado com a chave correta.

---

## Workflow no Dia a Dia

### Criptografar um arquivo pela primeira vez

```bash
# Se tem .sops.yaml configurado:
sops encrypt .env > .env.enc
mv .env.enc .env

# Ou especificando a chave manualmente:
sops encrypt --kms "arn:aws:kms:..." .env > .env.enc
```

### Editar secrets

O comando mais usado. Decripta, abre no editor, re-criptografa ao salvar:

```bash
sops edit .env
```

Usa o editor definido em `$EDITOR` (vim, nano, code, etc).

### Usar secrets sem arquivo em disco

O `exec-env` injeta as secrets como variáveis de ambiente em um subprocesso — nada fica em texto plano no disco:

```bash
# Abre um shell com todas as variáveis disponíveis
sops exec-env .env "bash"

# Executa um comando específico
sops exec-env .env "helm upgrade --set token=\$GITHUB_PAT ..."
```

### Decriptar para stdout (debug)

```bash
sops decrypt .env
```

### Ver diff entre versões

Como as chaves são legíveis, `git diff` funciona normalmente para mostrar quais secrets foram adicionadas ou removidas.

---

## Formatos Suportados

O SOPS funciona com múltiplos formatos:

| Formato | Extensão | Uso |
|---------|----------|-----|
| dotenv | `.env` | Variáveis de ambiente |
| YAML | `.yml`, `.yaml` | Helm values, configs K8s |
| JSON | `.json` | Configs de app |
| INI | `.ini` | Configs legadas |
| Binary | qualquer | Arquivos inteiros (criptografa tudo) |

### Exemplo com YAML

```yaml
# secrets.yml (após SOPS encrypt)
database:
    host: ENC[AES256_GCM,data:mQ2x...,type:str]
    password: ENC[AES256_GCM,data:k9Lp...,type:str]
    port: 5432  # valores não-sensíveis podem ser excluídos da criptografia
sops:
    kms:
        - arn: arn:aws:kms:us-east-1:xxxxxxxxxxxx:alias/sops-key
    version: 3.9.4
```

### Criptografar apenas alguns campos

Use `--encrypted-regex` para criptografar apenas campos que contenham "password", "secret", "token", etc:

```bash
sops encrypt --encrypted-regex '^(password|secret|token)$' config.yml
```

---

## Usando com Helm e Kubernetes

O caso de uso mais comum: passar secrets para `helm install` sem expô-las.

### Padrão com `exec-env`

```bash
# .env criptografado contém GITHUB_PAT=ghp_xxx
sops exec-env .env "helm install runner \
    --set githubConfigSecret.github_token=\$GITHUB_PAT \
    --namespace arc-runners \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set"
```

### Padrão com script

```bash
#!/bin/bash
# update-runners.sh — executar via: sops exec-env .env "./update-runners.sh"

helm upgrade --install runner \
    --namespace arc-runners \
    --set githubConfigSecret.github_token="${GITHUB_PAT}" \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

```bash
sops exec-env .env "./update-runners.sh"
```

### Valores sensíveis em um values file

Se preferir manter tudo em YAML:

```bash
# Criar values-secrets.yml com tokens
sops edit values-secrets.yml

# Usar decriptado inline com helm
sops decrypt values-secrets.yml | helm install runner \
    --namespace arc-runners \
    -f - \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

---

## Boas Práticas

### 1. Commite o arquivo criptografado

```gitignore
# .gitignore
# NÃO ignore o .env se ele está criptografado com SOPS
# Ignore apenas se for texto plano:
# .env
```

O ponto central do SOPS é poder versionar secrets com segurança. Se o arquivo está criptografado, ele **deve** estar no repositório.

### 2. Use `.sops.yaml` no projeto

Evita que alguém esqueça de especificar a chave e criptografe com o backend errado.

### 3. Controle acesso via IAM

```json
{
  "Effect": "Allow",
  "Action": ["kms:Decrypt", "kms:DescribeKey"],
  "Resource": "arn:aws:kms:us-east-1:xxxxxxxxxxxx:key/key-id"
}
```

Apenas quem precisa decriptar recebe `kms:Decrypt`. Outros podem ver a estrutura do arquivo sem acessar os valores.

### 4. Nunca decripte para um arquivo

```bash
# Ruim — cria arquivo em texto plano no disco
sops decrypt .env > .env.plain

# Bom — usa em memória apenas
sops exec-env .env "comando"
```

### 5. Adicione múltiplas chaves KMS

Para redundância ou acesso cross-account:

```yaml
# .sops.yaml
creation_rules:
  - path_regex: \.env$
    kms: "arn:aws:kms:us-east-1:xxxxxxxxxxxx:alias/sops-key,arn:aws:kms:us-east-1:yyyyyyyyyyyy:alias/sops-backup"
```

Qualquer uma das chaves pode decriptar o arquivo.

---

## Backends de Criptografia

O SOPS não é exclusivo da AWS. Ele suporta múltiplos backends — escolha o que faz sentido para sua infraestrutura:

### AWS KMS

Ideal se seu time já está na AWS. Controle de acesso via IAM, auditoria via CloudTrail.

```bash
export SOPS_KMS_ARN="arn:aws:kms:us-east-1:xxxxxxxxxxxx:alias/sops-key"
sops encrypt .env
```

```yaml
# .sops.yaml
creation_rules:
  - path_regex: \.env$
    kms: "arn:aws:kms:us-east-1:xxxxxxxxxxxx:alias/sops-key"
```

### GCP Cloud KMS

Mesmo conceito, para times no Google Cloud. Controle via IAM do GCP.

```bash
# Criar keyring e chave (uma vez)
gcloud kms keyrings create sops --location global
gcloud kms keys create sops-key \
    --location global \
    --keyring sops \
    --purpose encryption

# Criptografar
sops encrypt \
    --gcp-kms "projects/meu-projeto/locations/global/keyRings/sops/cryptoKeys/sops-key" \
    .env
```

```yaml
# .sops.yaml
creation_rules:
  - path_regex: \.env$
    gcp_kms: "projects/meu-projeto/locations/global/keyRings/sops/cryptoKeys/sops-key"
```

Para decriptar, basta ter `gcloud auth application-default login` configurado com permissão `cloudkms.cryptoKeyVersions.useToDecrypt`.

### Azure Key Vault

Para times no Azure:

```yaml
# .sops.yaml
creation_rules:
  - path_regex: \.env$
    azure_keyvault: "https://meu-vault.vault.azure.net/keys/sops-key/version-id"
```

### GPG/PGP

Funciona sem nenhum cloud provider. Cada membro do time tem sua chave GPG, e você criptografa para múltiplos destinatários:

```bash
# Listar chaves disponíveis
gpg --list-keys

# Criptografar para um ou mais fingerprints
sops encrypt \
    --pgp "FP_PESSOA_1,FP_PESSOA_2,FP_PESSOA_3" \
    .env
```

```yaml
# .sops.yaml
creation_rules:
  - path_regex: \.env$
    pgp: "FP_PESSOA_1,FP_PESSOA_2,FP_PESSOA_3"
```

Quando alguém entra ou sai do time, você adiciona/remove o fingerprint e re-criptografa:

```bash
# Adicionar novo membro
sops updatekeys .env
```

Vantagem: zero dependência de cloud. Desvantagem: gestão manual de chaves e distribuição.

### age (moderno, simples)

Substituto moderno do GPG, mais simples de usar:

```bash
# Gerar chave
age-keygen -o key.txt
# Output: public key: age1xxxxxxx...

# Criptografar
sops encrypt --age "age1xxxxxxx..." .env

# Decriptar
export SOPS_AGE_KEY_FILE=key.txt
sops decrypt .env
```

Recomendado para projetos pessoais ou times pequenos que não querem a complexidade do GPG.

### Múltiplos backends simultaneamente

Você pode combinar backends — útil para times distribuídos entre clouds ou para redundância:

```yaml
# .sops.yaml — qualquer uma das chaves pode decriptar
creation_rules:
  - path_regex: \.env$
    kms: "arn:aws:kms:us-east-1:xxxxxxxxxxxx:alias/sops-key"
    gcp_kms: "projects/meu-projeto/locations/global/keyRings/sops/cryptoKeys/sops-key"
    pgp: "FP_DO_ADMIN"
```

Isso garante que se um provider ficar indisponível, você ainda tem acesso via outro backend.

### Qual escolher?

| Backend | Quando usar |
|---------|------------|
| AWS KMS | Time na AWS, controle de acesso via IAM |
| GCP Cloud KMS | Time no GCP, mesmo modelo de IAM |
| Azure Key Vault | Time no Azure |
| GPG/PGP | Sem cloud, time com chaves GPG existentes |
| age | Projetos pessoais, simplicidade máxima |
| Múltiplos | Times multi-cloud ou para redundância |

---

## Conclusão

O SOPS com KMS resolve o problema de secrets com um workflow simples:

1. **Criptografe** — `sops encrypt .env`
2. **Commite** — o arquivo criptografado vai para o repositório
3. **Edite** — `sops edit .env` quando precisar alterar
4. **Use** — `sops exec-env .env "comando"` para injetar sem expor

Sem servidores extras, sem vault para manter, sem senhas compartilhadas. Quem tem acesso IAM à chave KMS consegue trabalhar. Quem não tem, vê apenas valores criptografados.

Para projetos que já estão na AWS, é a forma mais simples de sair do `.env` em texto plano para algo seguro e auditável.

---

## Referências

- [SOPS - GitHub](https://github.com/getsops/sops)
- [SOPS Documentation](https://getsops.io/)
- [AWS KMS Developer Guide](https://docs.aws.amazon.com/kms/latest/developerguide/)
- [age encryption](https://github.com/FiloSottile/age)
