---
title: 'Managing Secrets with SOPS: KMS, GCP and GPG'
date: 2026-05-09
permalink: /en/posts/2026/05/managing-secrets-sops/
uid: sops-secrets
tags:
  - devops
  - security
  - aws
  - gcp
  - sops
  - secrets
  - gpg
excerpt: 'Learn how to encrypt and manage project secrets using SOPS. Supports AWS KMS, GCP Cloud KMS, Azure Key Vault, age and GPG — choose the backend that makes sense for your team.'
lang: en
---
# Managing Secrets with SOPS: KMS, GCP and GPG

Friday, 5 PM. You finish that feature, run a satisfying `git add .`, commit, push. Go grab a coffee. On the way back to your computer, that sinking feeling: "wait... was the `.env` in staged?". You open GitHub, check the commit and there it is — `AWS_SECRET_ACCESS_KEY` in plain text, published to the world. The rest of Friday becomes a security incident: revoke keys, rotate credentials, notify the team, pray no one saw it.

If you've lived through this (or live in fear of it), **SOPS** is for you. It lets you commit your secret files **encrypted** directly into the repository. No fear. No incident. No ruined Friday.

---

## The Problem

Every application has secrets: API tokens, database credentials, private keys. The challenge is how to share them with the team without:

- Committing `.env` in plain text to the repository
- Relying on someone sending credentials via Slack/email
- Losing access when someone leaves the team
- Maintaining a complex vault for small/medium projects

**SOPS** (Secrets OPerationS) solves this: it encrypts only the **values** in your secret files, keeping the keys readable. Combined with a KMS (AWS, GCP, Azure) or GPG, anyone authorized can decrypt — without sharing passwords.

## TL;DR

```bash
# Install SOPS
# https://github.com/getsops/sops/releases

# Configure the KMS key
export SOPS_KMS_ARN="arn:aws:kms:us-east-1:xxxxxxxxxxxx:alias/your-key"

# Encrypt an existing file
sops encrypt .env > .env.enc

# Edit secrets (decrypts, opens editor, re-encrypts on save)
sops edit .env

# Use secrets as environment variables (no decrypted file on disk)
sops exec-env .env "helm install ..."
```

---

## Table of Contents

- [What is SOPS](#what-is-sops)
- [Installation](#installation)
- [Initial Configuration](#initial-configuration)
- [Daily Workflow](#daily-workflow)
- [Supported Formats](#supported-formats)
- [Encryption Backends](#encryption-backends)
- [Using with Helm and Kubernetes](#using-with-helm-and-kubernetes)
- [Best Practices](#best-practices)
- [Conclusion](#conclusion)

---

## What is SOPS

[SOPS](https://github.com/getsops/sops) is a tool originally from Mozilla (now maintained by the CNCF) that encrypts configuration files. The differentiator:

```bash
# Original file (.env)
GITHUB_PAT=ghp_abc123xyz789
AWS_SECRET_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCY
DB_PASSWORD=super-secret-123
```

```bash
# After encrypting with SOPS
GITHUB_PAT=ENC[AES256_GCM,data:k8vM2n...,type:str]
AWS_SECRET_KEY=ENC[AES256_GCM,data:pQ7xR...,type:str]
DB_PASSWORD=ENC[AES256_GCM,data:mN3kL...,type:str]
```

The **keys** (`GITHUB_PAT`, `AWS_SECRET_KEY`) remain readable — you know what the file contains without decrypting. Only the **values** are encrypted.

This means:
- The file can be safely committed to the repository
- Code review can see which secrets were added/removed
- `git diff` shows structural changes without exposing values

---

## Why Use a KMS (Key Management Service)

SOPS supports several backends. Cloud KMS services (AWS, GCP, Azure) share advantages over GPG/age:

- **IAM-based access control** — who can decrypt is controlled by cloud provider policies
- **Auditing** — every key usage is logged (CloudTrail, Cloud Audit Logs)
- **Automatic rotation** — the provider rotates the key without breaking existing files
- **No shared secret** — no need to distribute a private key across the team

If you don't use any cloud provider, GPG and age work perfectly — they just require more manual key management.

---

## Installation

### SOPS

Download the binary from the [releases page](https://github.com/getsops/sops/releases):

```bash
# Linux (amd64)
curl -LO https://github.com/getsops/sops/releases/download/v3.9.4/sops-v3.9.4.linux.amd64
mv sops-v3.9.4.linux.amd64 /usr/local/bin/sops
chmod +x /usr/local/bin/sops

# macOS (via Homebrew)
brew install sops

# Verify
sops --version
```

### AWS CLI

You need the AWS CLI configured with credentials that have access to the KMS key:

```bash
aws sts get-caller-identity  # Verify authentication
```

---

## Initial Configuration

### 1. Create (or identify) the KMS key

If you already have a KMS key, get the ARN or alias. If not:

```bash
aws kms create-key --description "SOPS encryption key"

# Create an alias for convenience
aws kms create-alias \
    --alias-name alias/sops-key \
    --target-key-id <returned-key-id>
```

### 2. Export the ARN

```bash
export SOPS_KMS_ARN="arn:aws:kms:us-east-1:xxxxxxxxxxxx:alias/sops-key"
```

### 3. Create the `.sops.yaml` file (optional, recommended)

At the project root, create a `.sops.yaml` so you don't need to pass the ARN every time:

```yaml
creation_rules:
  - path_regex: \.env$
    kms: "arn:aws:kms:us-east-1:xxxxxxxxxxxx:alias/sops-key"
  - path_regex: secrets\.ya?ml$
    kms: "arn:aws:kms:us-east-1:xxxxxxxxxxxx:alias/sops-key"
```

With this, any `.env` or `secrets.yml` file will automatically be encrypted with the correct key.

---

## Daily Workflow

### Encrypt a file for the first time

```bash
# If .sops.yaml is configured:
sops encrypt .env > .env.enc
mv .env.enc .env

# Or specifying the key manually:
sops encrypt --kms "arn:aws:kms:..." .env > .env.enc
```

### Edit secrets

The most used command. Decrypts, opens in editor, re-encrypts on save:

```bash
sops edit .env
```

Uses the editor defined in `$EDITOR` (vim, nano, code, etc).

### Use secrets without a file on disk

`exec-env` injects secrets as environment variables in a subprocess — nothing stays in plain text on disk:

```bash
# Open a shell with all variables available
sops exec-env .env "bash"

# Execute a specific command
sops exec-env .env "helm upgrade --set token=\$GITHUB_PAT ..."
```

### Decrypt to stdout (debug)

```bash
sops decrypt .env
```

### View diff between versions

Since keys are readable, `git diff` works normally to show which secrets were added or removed.

---

## Supported Formats

SOPS works with multiple formats:

| Format | Extension | Use |
|--------|-----------|-----|
| dotenv | `.env` | Environment variables |
| YAML | `.yml`, `.yaml` | Helm values, K8s configs |
| JSON | `.json` | App configs |
| INI | `.ini` | Legacy configs |
| Binary | any | Entire files (encrypts everything) |

### YAML Example

```yaml
# secrets.yml (after SOPS encrypt)
database:
    host: ENC[AES256_GCM,data:mQ2x...,type:str]
    password: ENC[AES256_GCM,data:k9Lp...,type:str]
    port: 5432  # non-sensitive values can be excluded from encryption
sops:
    kms:
        - arn: arn:aws:kms:us-east-1:xxxxxxxxxxxx:alias/sops-key
    version: 3.9.4
```

### Encrypt only specific fields

Use `--encrypted-regex` to encrypt only fields containing "password", "secret", "token", etc:

```bash
sops encrypt --encrypted-regex '^(password|secret|token)$' config.yml
```

---

## Using with Helm and Kubernetes

The most common use case: pass secrets to `helm install` without exposing them.

### Pattern with `exec-env`

```bash
# Encrypted .env contains GITHUB_PAT=ghp_xxx
sops exec-env .env "helm install runner \
    --set githubConfigSecret.github_token=\$GITHUB_PAT \
    --namespace arc-runners \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set"
```

### Pattern with script

```bash
#!/bin/bash
# update-runners.sh — run via: sops exec-env .env "./update-runners.sh"

helm upgrade --install runner \
    --namespace arc-runners \
    --set githubConfigSecret.github_token="${GITHUB_PAT}" \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

```bash
sops exec-env .env "./update-runners.sh"
```

### Sensitive values in a values file

If you prefer keeping everything in YAML:

```bash
# Create values-secrets.yml with tokens
sops edit values-secrets.yml

# Use decrypted inline with helm
sops decrypt values-secrets.yml | helm install runner \
    --namespace arc-runners \
    -f - \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

---

## Best Practices

### 1. Commit the encrypted file

```gitignore
# .gitignore
# DON'T ignore .env if it's encrypted with SOPS
# Only ignore if it's plain text:
# .env
```

The whole point of SOPS is being able to version secrets safely. If the file is encrypted, it **should** be in the repository.

### 2. Use `.sops.yaml` in the project

Prevents someone from forgetting to specify the key and encrypting with the wrong backend.

### 3. Control access via IAM

```json
{
  "Effect": "Allow",
  "Action": ["kms:Decrypt", "kms:DescribeKey"],
  "Resource": "arn:aws:kms:us-east-1:xxxxxxxxxxxx:key/key-id"
}
```

Only those who need to decrypt receive `kms:Decrypt`. Others can see the file structure without accessing the values.

### 4. Never decrypt to a file

```bash
# Bad — creates plain text file on disk
sops decrypt .env > .env.plain

# Good — uses in memory only
sops exec-env .env "command"
```

### 5. Add multiple KMS keys

For redundancy or cross-account access:

```yaml
# .sops.yaml
creation_rules:
  - path_regex: \.env$
    kms: "arn:aws:kms:us-east-1:xxxxxxxxxxxx:alias/sops-key,arn:aws:kms:us-east-1:yyyyyyyyyyyy:alias/sops-backup"
```

Any of the keys can decrypt the file.

---

## Encryption Backends

SOPS isn't exclusive to AWS. It supports multiple backends — choose what makes sense for your infrastructure:

### AWS KMS

Ideal if your team is already on AWS. Access control via IAM, auditing via CloudTrail.

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

Same concept, for teams on Google Cloud. Control via GCP IAM.

```bash
# Create keyring and key (once)
gcloud kms keyrings create sops --location global
gcloud kms keys create sops-key \
    --location global \
    --keyring sops \
    --purpose encryption

# Encrypt
sops encrypt \
    --gcp-kms "projects/my-project/locations/global/keyRings/sops/cryptoKeys/sops-key" \
    .env
```

```yaml
# .sops.yaml
creation_rules:
  - path_regex: \.env$
    gcp_kms: "projects/my-project/locations/global/keyRings/sops/cryptoKeys/sops-key"
```

To decrypt, just have `gcloud auth application-default login` configured with `cloudkms.cryptoKeyVersions.useToDecrypt` permission.

### Azure Key Vault

For teams on Azure:

```yaml
# .sops.yaml
creation_rules:
  - path_regex: \.env$
    azure_keyvault: "https://my-vault.vault.azure.net/keys/sops-key/version-id"
```

### GPG/PGP

Works without any cloud provider. Each team member has their GPG key, and you encrypt for multiple recipients:

```bash
# List available keys
gpg --list-keys

# Encrypt for one or more fingerprints
sops encrypt \
    --pgp "FP_PERSON_1,FP_PERSON_2,FP_PERSON_3" \
    .env
```

```yaml
# .sops.yaml
creation_rules:
  - path_regex: \.env$
    pgp: "FP_PERSON_1,FP_PERSON_2,FP_PERSON_3"
```

When someone joins or leaves the team, you add/remove the fingerprint and re-encrypt:

```bash
# Add new member
sops updatekeys .env
```

Advantage: zero cloud dependency. Disadvantage: manual key management and distribution.

### age (modern, simple)

Modern GPG replacement, simpler to use:

```bash
# Generate key
age-keygen -o key.txt
# Output: public key: age1xxxxxxx...

# Encrypt
sops encrypt --age "age1xxxxxxx..." .env

# Decrypt
export SOPS_AGE_KEY_FILE=key.txt
sops decrypt .env
```

Recommended for personal projects or small teams that don't want GPG complexity.

### Multiple backends simultaneously

You can combine backends — useful for teams distributed across clouds or for redundancy:

```yaml
# .sops.yaml — any of the keys can decrypt
creation_rules:
  - path_regex: \.env$
    kms: "arn:aws:kms:us-east-1:xxxxxxxxxxxx:alias/sops-key"
    gcp_kms: "projects/my-project/locations/global/keyRings/sops/cryptoKeys/sops-key"
    pgp: "FP_OF_ADMIN"
```

This ensures that if one provider becomes unavailable, you still have access via another backend.

### Which one to choose?

| Backend | When to use |
|---------|------------|
| AWS KMS | Team on AWS, IAM-based access control |
| GCP Cloud KMS | Team on GCP, same IAM model |
| Azure Key Vault | Team on Azure |
| GPG/PGP | No cloud, team with existing GPG keys |
| age | Personal projects, maximum simplicity |
| Multiple | Multi-cloud teams or for redundancy |

---

## Conclusion

SOPS with KMS solves the secrets problem with a simple workflow:

1. **Encrypt** — `sops encrypt .env`
2. **Commit** — the encrypted file goes to the repository
3. **Edit** — `sops edit .env` when you need to change
4. **Use** — `sops exec-env .env "command"` to inject without exposing

No extra servers, no vault to maintain, no shared passwords. Whoever has IAM access to the KMS key can work. Whoever doesn't, sees only encrypted values.

For projects already on AWS, it's the simplest way to move from plain text `.env` to something secure and auditable.

---

## References

- [SOPS - GitHub](https://github.com/getsops/sops)
- [SOPS Documentation](https://getsops.io/)
- [AWS KMS Developer Guide](https://docs.aws.amazon.com/kms/latest/developerguide/)
- [age encryption](https://github.com/FiloSottile/age)
