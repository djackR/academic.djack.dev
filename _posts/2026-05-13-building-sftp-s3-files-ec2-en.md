---
title: 'Building an SFTP with S3 Files on EC2'
lang: en
uid: sftp-s3-files-ec2
date: 2026-05-13
permalink: /posts/2026/05/building-sftp-s3-files-ec2/
tags:
  - aws
  - s3
  - s3-files
  - sftp
  - cdk
  - infrastructure
excerpt: 'S3 Files (launched November 2025) is the new AWS NFS file system backed by S3 buckets. Instead of testing with a "hello world", I picked a useful case from the start: an SFTP on EC2 with `/home/sftp` mounted via S3 Files, files landing straight in the bucket. Along the way, four lessons the docs do not highlight: peculiar IAM principal, wrong mount type, hidden botocore dependency, and versioning that changes what delete means.'
---
# Building an SFTP with S3 Files on EC2

S3 Files (launched Nov/2025) is the new AWS NFS file system backed by S3 buckets. Instead of testing with a "hello world", I picked a useful case from the start: an **SFTP server on EC2** with `/home/sftp` mounted via S3 Files — every partner upload lands straight in the bucket, no agent, no worker, no cron.

The full setup runs at ~$14/month (t4g.small + EBS + bucket + S3 Files). As a market reference, Transfer Family has a fixed ~$216/month, but the comparison isn't the focus here — the focus is understanding the new service by building something real on top of it.

Tested on **AL2023 + amazon-efs-utils 3.1.0**, May/2026.

**TL;DR — the four lessons:**

1. The IAM principal is `elasticfilesystem.amazonaws.com`, not `s3files.*`
2. `mount -t efs` fails silently; you need `-t s3files`
3. The helper needs `botocore` AND `elasticfilesystem:DescribeMountTargets` (not in any managed policy)
4. Versioning is mandatory → `rm` creates a delete marker; without lifecycle, cost grows forever

This post is the walkthrough of what worked + the literal errors that came up. It's not a mechanical tutorial ("run these commands"); it's the actual sequence with the decisions I had to make.

## What S3 Files is

In one sentence: you mount a local directory on an EC2 instance and anything you write there shows up in S3 within seconds, with the bucket as source of truth (versioning, lifecycle, replication).

## Architecture

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
   NFS  │     │ (OS + host keys)
        ▼
  ┌──────────────┐
  │ S3 Files     │
  │ /home/sftp   │
  └──────┬───────┘
         │ auto sync
         ▼
   ┌─────────┐
   │ Bucket  │
   │  S3     │
   └─────────┘
```

EC2 t4g.small running Amazon Linux 2023, minimal IAM, IMDSv2 required, SG with restricted egress. Provisioned via AWS CDK (Python).

---

## Lesson 1: the IAM principal is called `elasticfilesystem`, not `s3files`

S3 Files needs a **service role** that the service assumes to read/write your bucket. The first question I had to answer was: which service principal is correct?

The prerequisites page ([s3-files-prereq-policies.html](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-files-prereq-policies.html)) answers it in the very first trust policy block — and the answer is less obvious than the service name suggests:

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

Two details worth highlighting:

1. **The principal is `elasticfilesystem.amazonaws.com`, not `s3files.*`**. S3 Files reuses the EFS control plane behind the scenes. Trusting the service name to guess the principal lands you on `Invalid principal in policy: "SERVICE":"s3files.amazonaws.com"`.

2. The `aws:SourceAccount` and `aws:SourceArn` conditions are the **confused deputy countermeasure** — they close the gap where another file system, possibly from another account, could trick the EFS principal into assuming your role. The docs ship them ready to use; keep them in your code.

Lesson: for any new AWS service, opening the IAM/prerequisites page before writing code saves a broken deploy.

## Lesson 2: the mount type is `s3files`, not `efs`

The docs list `amazon-efs-utils` (≥ 3.0.0) as a prerequisite and describe a "mount helper". The instinct, coming from someone who has used EFS, is to mount with `-t efs`:

```bash
echo "$FS_ID:/ /home/sftp efs _netdev,tls,noatime 0 0" >> /etc/fstab
mount -a -t efs
```

And nothing happens. The log at `/var/log/amazon/efs/mount.log` shows:

```
Failed to resolve fs-XXX.efs.us-east-1.amazonaws.com
```

The helper, with `-t efs`, looks for an EFS endpoint — which doesn't exist for an S3 Files file system. The correct command lives in [Step 3 of the Getting Started tutorial](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-files-getting-started.html):

```bash
sudo mount -t s3files <file-system-id>:/ /mnt/s3files
```

And in fstab:

```
fs-XXXXXXXXXXXXXXXXX:/  /home/sftp  s3files  _netdev,noatime  0 0
```

The maturity here is realizing `amazon-efs-utils` has become an umbrella package: the same binary responds to two configs (`/etc/amazon/efs/efs-utils.conf` and `/etc/amazon/efs/s3files-utils.conf`) depending on the mount type declared. With `-t s3files`, the helper opens a local stunnel TLS at `127.0.0.1:<random-port>` and tunnels NFS to that endpoint:

```
$ mount | grep sftp
127.0.0.1:/ on /home/sftp type nfs4 (rw,noatime,vers=4.2,...,port=20423,...)
```

Lesson: for services that reuse existing packages/infrastructure (`amazon-efs-utils`, `elasticfilesystem` principal), the Getting Started tutorial usually carries the detail the prerequisites page omits. Worth reading both before you implement.

## Lesson 3: `amazon-efs-utils` doesn't pull `botocore`, and needs an extra IAM permission

Even with `-t s3files` correct, two errors show up in sequence. Both share a root cause: `mount.s3files` needs to **discover the mount target IP at runtime**, and to do that it makes an API call with `boto3`.

### Error 1: `botocore` missing

```
ERROR - Failed to import botocore, please install botocore first.
```

The docs do warn about this in a whole section ([Step 2: Install botocore](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-files-prereq-policies.html)). What's worth recording: on Amazon Linux 2023 the `amazon-efs-utils` package **does not declare** `python3-botocore` as a dependency — `dnf install amazon-efs-utils` alone won't resolve it. Definitive fix in user-data:

```bash
dnf install -y amazon-efs-utils python3-botocore
```

### Error 2: missing IAM permission

After fixing `botocore`, the next error:

```
User: ... is not authorized to perform:
  elasticfilesystem:DescribeMountTargets on the specified resource
```

This part is worth highlighting: efs-utils 3.1.0 **calls the EFS API** to discover the mount target IP, even when the file system is S3 Files. The [IAM role for attaching your file system](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-files-prereq-policies.html) section recommends either the managed policy `AmazonS3FilesClientFullAccess` or granular `s3files:ClientMount` + `s3files:ClientWrite`. Neither includes `elasticfilesystem:*`. Solution:

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

I added it as an explicit statement for two reasons: it makes visible **why** that permission exists (efs-utils, not the service), and it survives a future package update that may not need it anymore.

> The docs mention the `AmazonElasticFileSystemUtils` managed policy in another context (CloudWatch). I haven't tested, but it likely covers these two actions already — possible simplification.

Lesson: when an SDK/client is shared between services, always look at **the APIs it calls internally**, not just the target service's. The mount helper's log tells you exactly which call failed — that's the fastest path to correct IAM.

## Lesson 4: versioning is mandatory — and that changes what "delete" means

S3 Files **requires** versioning enabled on the bucket. The docs state it plainly in the [AWS account and compute setup](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-files-prereq-policies.html) section:

> *Your S3 bucket has versioning enabled. **S3 Files requires S3 Versioning** to synchronize changes between your file system and your S3 bucket.*

Without it, file system creation fails. The operational consequence is non-trivial:

- When an SFTP user runs `rm file.txt` inside the chroot, S3 Files passes it to the bucket. Since the bucket is versioned, **there is no real delete**: a *delete marker* is created and the original version is preserved.
- `aws s3 ls` (lists visible objects) shows the object gone, but `aws s3api list-object-versions` keeps showing both: the old version and the delete marker hiding it.
- Without a **lifecycle policy**, those versions stay forever in the bucket, paying storage.

I lived through this when removing a test user:

```bash
userdel testuser
rm -rf /home/sftp/testuser
```

`aws s3 ls` showed the bucket "clean". But `list-object-versions` revealed three versions + three delete markers for `testuser/upload/teste.txt`. For a real purge:

```python
import json, subprocess
v = json.loads(subprocess.run([
    "aws", "s3api", "list-object-versions",
    "--bucket", BUCKET, "--prefix", "testuser/", "--region", REGION,
    "--output", "json"], capture_output=True, text=True).stdout)
objs = [{'Key': x['Key'], 'VersionId': x['VersionId']}
        for x in (v.get('Versions') or []) + (v.get('DeleteMarkers') or [])]
```

Then send the batch to `delete-objects`. The structural fix is to **always add a lifecycle policy** when creating the bucket:

```python
bucket = s3.Bucket(self, "SftpBucket",
    versioned=True,  # S3 Files requirement
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

Without this, cost grows linearly with file movement and it's easy to forget.

Lesson: when a service **requires** versioning, lifecycle isn't optimization — it's part of the design. Treating it as a mandatory bucket feature (same construct, same PR) prevents it from becoming operational debt.

## When S3 Files makes sense

For a low-to-medium volume SFTP (a few GB/month) the full setup runs at ~$14/month: t4g.small + EBS + bucket + S3 Files cost. As a market reference, Transfer Family has a fixed ~$216/month before the first byte transferred — that's when S3 Files starts to make a lot of sense for teams comfortable maintaining an EC2.

The **breakeven** vs Transfer Family sits around 4.3 TB/month of traffic — above that, S3 Files cache cost ($0.06/GB stored in cache + $0.03/GB sync) grows past the fixed Transfer Family cost. For most partner-SFTP cases (a few files per day), you stay far from that limit.

What S3 Files does **not** ship out of the box (that managed services do):
- Centralized SSH user and key management.
- AS2 and FTPS support.
- Zero-touch operation — you still have an EC2 to patch and monitor.

Where it earns a spot as a new tool in the kit:
- Pipelines where the **bucket must be source of truth** (lifecycle, replication, Object Lock).
- Legacy workloads that speak NFS but the team already has S3 tooling (Glue, Athena, Lambda).
- Replacing EFS in write-once/read-many scenarios, where S3 storage ($0.023/GB) beats EFS ($0.30/GB).

## References

- [Prerequisites for S3 Files](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-files-prereq-policies.html) — IAM, SG, botocore and versioning (source of lessons #1, #3 and #4)
- [Tutorial: Getting started with S3 Files](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-files-getting-started.html) — where `mount -t s3files` finally appears
- [AWS::S3Files::FileSystem CFN](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-s3files-filesystem.html) — resource properties
- [AWS S3 Files just made Transfer Family SFTP obsolete](https://dev.to/aws-builders/aws-s3-files-just-made-transfer-family-sftp-obsolete-for-most-use-cases-4me) — original post that motivated the experiment
