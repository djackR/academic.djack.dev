---
title: 'Montando SFTP com S3 Files no EC2'
lang: pt
uid: sftp-s3-files-ec2
date: 2026-05-13
permalink: /posts/2026/05/montando-sftp-s3-files-ec2/
tags:
  - aws
  - s3
  - s3-files
  - sftp
  - cdk
  - infrastructure
excerpt: 'O S3 Files (lançado em novembro/2025) é o novo NFS da AWS lastreado por bucket S3. Em vez de testar com um "hello world", escolhi um caso útil de primeira: SFTP em EC2 com `/home/sftp` montado via S3 Files, com dados caindo direto no bucket. No caminho, quatro aprendizados que a documentação não destaca: principal IAM peculiar, mount com tipo errado, dependência implícita do botocore e versionamento que muda o significado de delete.'
---
# Montando SFTP com S3 Files no EC2

S3 Files (lançado nov/2025) é o novo NFS da AWS lastreado por bucket S3. Em vez de testar com um "hello world", escolhi um caso útil de primeira: **servidor SFTP em EC2** com `/home/sftp` montado via S3 Files — cada upload de parceiro cai direto no bucket, sem agente, sem worker, sem cron.

Saiu por ~$14/mês (t4g.small + EBS + bucket + S3 Files). Como referência de mercado, Transfer Family tem fixo de ~$216/mês, mas a comparação não é o foco aqui — o foco é entender o serviço novo construindo algo de verdade em cima dele.

Testado em **AL2023 + amazon-efs-utils 3.1.0**, maio/2026.

**TL;DR — os quatro aprendizados:**

1. O principal IAM é `elasticfilesystem.amazonaws.com`, não `s3files.*`
2. `mount -t efs` falha silenciosamente; precisa ser `-t s3files`
3. O helper precisa de `botocore` E de `elasticfilesystem:DescribeMountTargets` (não está em nenhuma managed policy)
4. Versionamento é obrigatório → `rm` cria delete marker; sem lifecycle vira custo eterno

O post é o passo a passo do que funcionou + os erros literais que apareceram. Não é tutorial mecânico ("execute esses comandos"); é a sequência real, com as decisões que tive que tomar.

## O que é S3 Files

Em uma frase: você monta um diretório local em uma EC2 e tudo o que escrever lá aparece no S3 em segundos, mantendo o bucket como fonte da verdade (com versionamento, lifecycle e replicação).

## Arquitetura do experimento

```
        Internet
           │
        :22 TCP
           │
     ┌─────▼─────┐
     │ EC2 AL2023│
     │   ARM     │
     │           │
     │  sshd +   │
     │  chroot   │
     │  fail2ban │
     └──┬─────┬──┘
        │     │
   :2049│     │ EBS gp3
   NFS  │     │ (SO + host keys)
        ▼
  ┌──────────────┐
  │ S3 Files     │
  │ /home/sftp   │
  └──────┬───────┘
         │ sync auto
         ▼
   ┌─────────┐
   │ Bucket  │
   │  S3     │
   └─────────┘
```

EC2 t4g.small com Amazon Linux 2023, IAM minimal, IMDSv2 obrigatório, SG com egress restrito. Tudo provisionado via AWS CDK (Python).

---

## Aprendizado 1: o principal IAM se chama `elasticfilesystem`, não `s3files`

S3 Files exige uma **service role** que o serviço assume para ler e escrever no seu bucket. A primeira pergunta que precisei responder foi: qual é o service principal correto?

A doc de pré-requisitos ([s3-files-prereq-policies.html](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-files-prereq-policies.html)) responde já no primeiro bloco de trust policy — e a resposta é menos óbvia do que parece pelo nome do serviço:

```python
assumed_by=iam.PrincipalWithConditions(
    iam.ServicePrincipal("elasticfilesystem.amazonaws.com"),
    conditions={
        "StringEquals": {"aws:SourceAccount": self.account},
        "ArnLike": {
            "aws:SourceArn": f"arn:aws:s3files:{self.region}:{self.account}:file-system/*"
        },
    },
)
```

Dois detalhes valem destaque:

1. **O principal é `elasticfilesystem.amazonaws.com`, não `s3files.*`**. S3 Files reaproveita a infra de controle do EFS por trás dos panos. Confiar no nome do serviço pra adivinhar o principal te leva pro erro `Invalid principal in policy: "SERVICE":"s3files.amazonaws.com"`.

2. As condições `aws:SourceAccount` e `aws:SourceArn` são **contra-medida ao confused deputy** — fecham a brecha em que outro file system, possivelmente de outra conta, poderia induzir o principal EFS a assumir sua role. A doc traz isso pronto; manter no código.

Lição: pra serviço novo da AWS, abrir a página de IAM/pré-requisitos antes de escrever código economiza um deploy quebrado.

## Aprendizado 2: o tipo do mount é `s3files`, não `efs`

A doc lista `amazon-efs-utils` (≥ 3.0.0) como pré-requisito e descreve um "mount helper". O instinto, vindo de quem já usou EFS, é montar com `-t efs`:

```bash
echo "$FS_ID:/ /home/sftp efs _netdev,tls,noatime 0 0" >> /etc/fstab
mount -a -t efs
```

E nada acontece. O log em `/var/log/amazon/efs/mount.log` mostra:

```
Failed to resolve fs-XXX.efs.us-east-1.amazonaws.com
```

O helper, com `-t efs`, busca um endpoint EFS — que não existe para um file system S3 Files. O comando correto está no [Step 3 do tutorial Getting Started](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-files-getting-started.html):

```bash
sudo mount -t s3files <file-system-id>:/ /mnt/s3files
```

E no fstab:

```
fs-XXXXXXXXXXXXXXXXX:/  /home/sftp  s3files  _netdev,noatime  0 0
```

A maturidade aqui é entender que `amazon-efs-utils` virou um pacote guarda-chuva: o mesmo binário responde a duas configs (`/etc/amazon/efs/efs-utils.conf` e `/etc/amazon/efs/s3files-utils.conf`) dependendo do tipo declarado no mount. Com `-t s3files`, o helper abre um stunnel TLS local em `127.0.0.1:<porta-aleatória>` e tunela NFS pra esse endpoint:

```
$ mount | grep sftp
127.0.0.1:/ on /home/sftp type nfs4 (rw,noatime,vers=4.2,...,port=20423,...)
```

Lição: para serviços que reusam pacotes/infraestrutura existentes (`amazon-efs-utils`, principal `elasticfilesystem`), o tutorial Getting Started costuma trazer o detalhe que o pré-requisitos omite. Vale ler os dois antes de implementar.

## Aprendizado 3: o pacote `amazon-efs-utils` não puxa `botocore` e exige uma permissão IAM extra

Mesmo com `-t s3files` correto, dois erros aparecem em sequência. Os dois têm raiz comum: o `mount.s3files` precisa **descobrir o IP do mount target em runtime**, e pra isso ele faz uma chamada de API com `boto3`.

### Erro 1: `botocore` ausente

```
ERROR - Failed to import botocore, please install botocore first.
```

A doc avisa numa seção inteira ([Step 2: Install botocore](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-files-prereq-policies.html)). O detalhe que vale registrar é que no Amazon Linux 2023 o pacote `amazon-efs-utils` **não declara** `python3-botocore` como dependência — `dnf install amazon-efs-utils` sozinho não resolve. Fix definitivo no user-data:

```bash
dnf install -y amazon-efs-utils python3-botocore
```

### Erro 2: permissão IAM ausente

Resolvido o `botocore`, o próximo erro:

```
User: ... is not authorized to perform:
  elasticfilesystem:DescribeMountTargets on the specified resource
```

Essa parte vale o destaque: o efs-utils 3.1.0 **chama a API do EFS** para descobrir o IP do mount target, mesmo quando o file system é S3 Files. A doc de [IAM role for attaching your file system](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-files-prereq-policies.html) recomenda a managed policy `AmazonS3FilesClientFullAccess` ou as ações granulares `s3files:ClientMount` + `s3files:ClientWrite`. Nenhuma delas inclui `elasticfilesystem:*`. Solução:

```python
iam.PolicyStatement(
    sid="EfsUtilsMountTargetLookup",
    actions=[
        "elasticfilesystem:DescribeMountTargets",
        "elasticfilesystem:DescribeFileSystems",
    ],
    resources=["*"],
)
```

Adicionei como statement explícito por dois motivos: deixa visível **por que** essa permissão existe (referência ao efs-utils, não ao serviço), e sobrevive a uma futura atualização do pacote que talvez nem precise mais dela.

> A doc menciona a managed policy `AmazonElasticFileSystemUtils` em outro contexto (CloudWatch). Não testei, mas é provável que ela já cubra essas duas ações — fica como possível simplificação.

Lição: quando um SDK/cliente é compartilhado entre serviços, sempre olhe **as APIs que ele chama internamente**, não só as do serviço-alvo. O log do mount helper diz exatamente qual chamada falhou — esse é o caminho mais rápido pra IAM correto.

## Aprendizado 4: bucket versionado é obrigatório — e isso muda o significado de "deletar"

O S3 Files **exige** versionamento ligado no bucket. Está explícito na seção [AWS account and compute setup](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-files-prereq-policies.html) da doc:

> *Your S3 bucket has versioning enabled. **S3 Files requires S3 Versioning** to synchronize changes between your file system and your S3 bucket.*

Sem isso, a criação do file system falha. A consequência operacional não é trivial:

- Quando um usuário SFTP roda `rm arquivo.txt` no chroot, o S3 Files repassa para o bucket. Como o bucket é versionado, **não há delete de fato**: cria-se um *delete marker* e a versão original fica preservada.
- `aws s3 ls` (lista as visíveis) mostra o objeto sumido, mas `aws s3api list-object-versions` continua mostrando ambos: a versão antiga e o delete marker que a esconde.
- Sem **lifecycle policy**, essas versões ficam indefinidamente no bucket pagando armazenamento.

Eu vivenciei isso ao apagar um usuário de teste:

```bash
userdel testuser
rm -rf /home/sftp/testuser
```

`aws s3 ls` apresentou o bucket "limpo". Mas `list-object-versions` revelou três versões + três delete markers do `testuser/upload/teste.txt`. Para purge real:

```python
import json, subprocess
v = json.loads(subprocess.run([
    "aws", "s3api", "list-object-versions",
    "--bucket", BUCKET, "--prefix", "testuser/", "--region", REGION,
    "--output", "json"], capture_output=True, text=True).stdout)
objs = [{'Key': x['Key'], 'VersionId': x['VersionId']}
        for x in (v.get('Versions') or []) + (v.get('DeleteMarkers') or [])]
```

E enviar o lote ao `delete-objects`. A solução estrutural é **sempre adicionar uma lifecycle policy** ao criar o bucket:

```python
bucket = s3.Bucket(self, "SftpBucket",
    versioned=True,  # exigência do S3 Files
    lifecycle_rules=[
        s3.LifecycleRule(
            id="DeleteOldVersions",
            noncurrent_version_expiration=Duration.days(90),
        ),
        s3.LifecycleRule(
            id="CleanupDeleteMarkers",
            expired_object_delete_marker=True,
        ),
    ],
)
```

Sem isso, o custo cresce linearmente com a movimentação de arquivos e fica fácil esquecer.

Lição: quando o serviço **exige** versionamento, lifecycle não é otimização — é parte do desenho. Tratar como recurso obrigatório do bucket (no mesmo construct, no mesmo PR) evita que vire dívida operacional.

## Quando faz sentido usar S3 Files

Para um SFTP de volume baixo a médio (alguns GB/mês) o setup completo sai por ~$14/mês: t4g.small + EBS + bucket + custo do S3 Files. Como referência de mercado, o Transfer Family tem fixo de ~$216/mês antes do primeiro byte transferido — aí o S3 Files começa a fazer muito sentido para times que topam manter uma EC2.

O **breakeven** com Transfer Family sai por volta de 4,3 TB/mês de tráfego — acima disso o custo de cache do S3 Files ($0,06/GB armazenado em cache + $0,03/GB sync) cresce além do fixo do Transfer Family. Para a maioria dos casos de SFTP de parceiros (alguns arquivos por dia), você fica muito longe desse limite.

O que o S3 Files **não** entrega de fábrica (e que serviços managed entregam):
- Gerência centralizada de usuários e chaves SSH.
- Suporte a AS2 e FTPS.
- Operação zero-touch — você ainda tem uma EC2 pra patchar e monitorar.

E onde ele entra como ferramenta nova no toolkit:
- Pipelines em que o **bucket precisa ser fonte da verdade** (lifecycle, replicação, Object Lock).
- Workloads legadas que falam NFS mas o time já tem todo o ferramental S3 (Glue, Athena, Lambda).
- Substituir EFS em cenários de mais escrita-uma-vez/leitura-muitas, onde o custo do S3 ($0,023/GB) supera o do EFS ($0,30/GB).

## Referências

- [Prerequisites for S3 Files](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-files-prereq-policies.html) — IAM, SG, botocore e versionamento (fonte dos aprendizados #1, #3 e #4)
- [Tutorial: Getting started with S3 Files](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-files-getting-started.html) — onde finalmente aparece `mount -t s3files`
- [AWS::S3Files::FileSystem CFN](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-s3files-filesystem.html) — propriedades do recurso
- [AWS S3 Files just made Transfer Family SFTP obsolete](https://dev.to/aws-builders/aws-s3-files-just-made-transfer-family-sftp-obsolete-for-most-use-cases-4me) — post original que motivou o experimento
