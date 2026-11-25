# S3 Dev Bucket Migration Runbook (Old AWS Account → New AWS Account)

Replicates S3 buckets **with data and KMS encryption** from the old account to the new account, one bucket at a time. Validated on `agenthub-dev-s3-bucket` → `studio-dev-project-bucket`.

---

## 0. Variables (fill these in)

| Variable | Value |
|---|---|
| `OLD_ACCOUNT_ID` | `890290782969` |
| `NEW_ACCOUNT_ID` | `254265893962` |
| `REGION` | `ap-south-1` |
| `REPL_ROLE` (old account) | `s3-replication-role` |
| `REPL_ROLE_ARN` | `arn:aws:iam::890290782969:role/s3-replication-role` |
| `DEV_KEY_ARN` (new account) | `arn:aws:kms:ap-south-1:254265893962:key/c2f8d666-0ad4-4b38-8a69-4cbe3292dd5c` |
| `DEV_KEY_ALIAS` | `studio-dev-kms-key` |

### Per-bucket mapping (one row per bucket)

| # | Source bucket (old) | Source KMS key ARN (old) | Destination bucket (new) | Status |
|---|---|---|---|---|
| 1 | `agenthub-dev-s3-bucket` | `arn:aws:kms:ap-south-1:890290782969:key/761a36b0-8ec0-4f17-8ed7-000ef2ddc599` | `studio-dev-project-bucket` | Done |
| 2 | | | | |
| 3 | | | | |

Notes:
- Bucket names are global, so every destination bucket needs a new unique name.
- Find a source bucket's KMS key: S3 → bucket → Properties → Default encryption, or  
  `aws s3api get-bucket-encryption --bucket <SRC> --profile old`
- If a source bucket uses SSE-S3 (no KMS), you can skip every KMS step for that bucket (no `kms:Decrypt` needed, and untick "Replicate objects encrypted with KMS" is fine, but leaving it on with the destination key is harmless).

---

## Part A: One-time setup (do once, reuse for all dev buckets)

### A1. Create the dev KMS key (NEW account)

1. Switch to the **new account**, region `ap-south-1`.
2. KMS → Customer managed keys → **Create key**.
3. Key type **Symmetric**, usage **Encrypt and decrypt**.
4. Alias: `studio-dev-kms-key`. Add yourself as key administrator and the app roles that will use the buckets as key users.
5. Create the key and copy the **key ARN** (`DEV_KEY_ARN`).

> The key must be a **customer-managed key** in the **same region** as the destination buckets. The AWS-managed `aws/s3` key cannot be used cross-account.

### A2. Create the replication IAM role (OLD account)

The role must exist **before** you add it to the KMS key policy, otherwise KMS rejects it with `MalformedPolicyDocumentException - invalid principals`.

1. IAM → Roles → **Create role**.
2. Trusted entity: **AWS service → S3**.
3. Name: `s3-replication-role`.
4. Add an inline policy (see A3).

### A3. Role permissions policy (OLD account)

Start with the first bucket. For every additional bucket, **add its resources** to the lists (see Part B, step B6).

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SourceBucketRead",
      "Effect": "Allow",
      "Action": ["s3:GetReplicationConfiguration", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::agenthub-dev-s3-bucket"
      ]
    },
    {
      "Sid": "SourceObjectRead",
      "Effect": "Allow",
      "Action": [
        "s3:GetObjectVersionForReplication",
        "s3:GetObjectVersionAcl",
        "s3:GetObjectVersionTagging"
      ],
      "Resource": [
        "arn:aws:s3:::agenthub-dev-s3-bucket/*"
      ]
    },
    {
      "Sid": "DestinationWrite",
      "Effect": "Allow",
      "Action": [
        "s3:ReplicateObject",
        "s3:ReplicateDelete",
        "s3:ReplicateTags",
        "s3:ObjectOwnerOverrideToBucketOwner"
      ],
      "Resource": [
        "arn:aws:s3:::studio-dev-project-bucket/*"
      ]
    },
    {
      "Sid": "DecryptWithOldKeys",
      "Effect": "Allow",
      "Action": "kms:Decrypt",
      "Resource": [
        "arn:aws:kms:ap-south-1:890290782969:key/761a36b0-8ec0-4f17-8ed7-000ef2ddc599"
      ]
    },
    {
      "Sid": "EncryptWithNewKey",
      "Effect": "Allow",
      "Action": ["kms:Encrypt", "kms:GenerateDataKey"],
      "Resource": "arn:aws:kms:ap-south-1:254265893962:key/c2f8d666-0ad4-4b38-8a69-4cbe3292dd5c"
    }
  ]
}
```

### A4. Add the role to the new KMS key policy (NEW account)

KMS → the key → **Key policy → Edit**. Keep the default root statement and add the second one:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::254265893962:root"},
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "AllowOldAccountReplicationRole",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::890290782969:role/s3-replication-role"},
      "Action": ["kms:Encrypt", "kms:GenerateDataKey", "kms:DescribeKey"],
      "Resource": "*"
    }
  ]
}
```

Do **not** remove the root statement or you can lock yourself out of the key.

---

## Part B: Per-bucket steps (repeat for every bucket)

### B1. Create the destination bucket (NEW account)

1. S3 → **Create bucket**.
2. Name: the destination name from the mapping table. Region: `ap-south-1`.
3. Object Ownership: ACLs disabled (bucket owner enforced).
4. Block all public access: **on** (unless the old bucket served public content, then match the old setting).
5. **Bucket Versioning: Enable** (required for replication).
6. Default encryption: **SSE-KMS** → choose `studio-dev-kms-key` (or paste `DEV_KEY_ARN`). Bucket Key: **Enabled**.
7. Copy tags if needed (for example `Environment=dev`, `Project=...`).
8. Create bucket.

### B2. Destination bucket policy (NEW account)

Bucket → Permissions → Bucket policy. Replace `DEST_BUCKET`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowReplicationFromOldAccount",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::890290782969:root"},
      "Action": [
        "s3:GetBucketVersioning",
        "s3:PutBucketVersioning",
        "s3:ReplicateObject",
        "s3:ReplicateDelete",
        "s3:ReplicateTags",
        "s3:ObjectOwnerOverrideToBucketOwner"
      ],
      "Resource": [
        "arn:aws:s3:::DEST_BUCKET",
        "arn:aws:s3:::DEST_BUCKET/*"
      ]
    }
  ]
}
```

> `s3:ObjectOwnerOverrideToBucketOwner` and `s3:ReplicateTags` are required. The console auto-generated policy omits them, so add them if you let the console generate the policy.
> To be stricter, replace the principal `root` with the role ARN `arn:aws:iam::890290782969:role/s3-replication-role`.

### B3. Confirm source bucket versioning (OLD account)

Source bucket → Properties → **Bucket Versioning must be Enabled**. Enable it if not.

### B4. Allow the role to decrypt the source key (OLD account)

Only if the source bucket uses a customer-managed KMS key:
- Add the source key ARN to the `DecryptWithOldKeys` statement of the role policy (A3).
- The old key's policy must allow the role. The default key policy (root statement with `kms:*`) already delegates to IAM, so no change is needed. If the key policy has no root statement, add `kms:Decrypt` for the role.

### B5. Create the replication rule (OLD account)

Source bucket → **Management → Replication rules → Create replication rule**.

| Field | Value |
|---|---|
| Rule name | `new-account-migration` |
| Status | Enabled |
| Rule scope | Apply to all objects in the bucket |
| Destination | **Specify a bucket in another account** |
| Account ID | `254265893962` |
| Bucket name | destination bucket name |
| Change object ownership to destination bucket owner | **Tick** |
| IAM role | **Choose from existing IAM roles → `s3-replication-role`** (do not choose "Create new role") |
| Encryption | **Replicate** objects encrypted with AWS KMS |
| Encryption key type | Tick **Specify destination key** → Enter KMS key ARN → `DEV_KEY_ARN` |
| Replication metrics | Optional, recommended (progress and failure visibility) |
| Delete marker replication | Off for a one-time migration |

Click **Save**.

> The console may then ask about existing objects. If it does, choose **Yes, replicate existing objects**. If you saved the rule without being asked, do step B6 manually.

### B6. Replicate existing objects: Batch Replication (OLD account)

A replication rule only copies **new** uploads. To copy existing data:

1. Source bucket → Management → Replication rules → select `new-account-migration` → **Actions → Create replication job**  
   (or S3 → Batch Operations → Create job).
2. **Step 1, Region and scope:** "Generate an object list based on a replication configuration", source = `s3://<SRC_BUCKET>`, no filters.
3. **Step 2, Operation:** Replicate.
4. **Step 3, Options:** enable the completion report (failed tasks only) and pick a bucket in the old account for the report. IAM role: create a new role if prompted.
5. **Step 4:** Review and create the job.
6. If the job status is "Awaiting your confirmation to run", select it and click **Run job**.
7. Wait for status Complete. Check **Total succeeded** and **Total failed**.

### B7. Update the replication IAM role for the new bucket (OLD account)

Before running B6 for bucket #2 onward, add to the role policy (A3):
- Source bucket ARN under `SourceBucketRead`
- Source `/*` ARN under `SourceObjectRead`
- Destination `/*` ARN under `DestinationWrite`
- Source KMS key ARN under `DecryptWithOldKeys`

Do this before creating the rule and job.

### B8. Verify (both accounts)

- Upload a test file in the source bucket → it appears in the destination within minutes.
- Open an object in the destination bucket:
  - **Replication status: REPLICA**
  - Encryption: SSE-KMS with the **new** key ARN
  - Owner: the new account
- Compare object counts:
  ```bash
  aws s3 ls s3://<SRC_BUCKET>  --recursive --summarize --profile old | tail -2
  aws s3 ls s3://<DEST_BUCKET> --recursive --summarize --profile new | tail -2
  ```
- For a stricter check, spot-check ETags or sizes of a few objects.

### B9. Cutover and cleanup

1. Update application configs, env vars, IAM policies, CloudFront origins, and any bucket-name references to the new bucket name.
2. Grant the new account's app roles access to the new bucket and `studio-dev-kms-key` (key user).
3. Verify the apps work on the new bucket.
4. **Disable** the replication rule on the old bucket.
5. Keep the old bucket for a few days. Delete it only after confirmation. Empty it first, then delete it.

---

## Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| `MalformedPolicyDocumentException - invalid principals` when saving key policy | `s3-replication-role` does not exist yet. Create the role first (A2), or temporarily use `arn:aws:iam::890290782969:root` as the principal. |
| Objects not replicating, nothing shows | Rule set to "Do not replicate" for KMS objects, versioning off on source or destination, or the rule was created after the objects (needs Batch Replication). |
| Replication status `FAILED` on objects | Missing `kms:Decrypt` on the old key, missing `kms:Encrypt` or `kms:GenerateDataKey` on the new key, or the new key policy lacks the role. |
| `AccessDenied` for destination | Destination bucket policy missing `ReplicateObject` / `ObjectOwnerOverrideToBucketOwner` / `ReplicateTags`, or the policy is on the wrong bucket name. |
| Replica owned by old account | "Change object ownership to destination bucket owner" was not ticked. Edit the rule and tick it, then re-run Batch Replication. |
| Batch job shows failures | Open the completion report in the report bucket. Check the error code per object. |
| Rule uses a new auto-created role | The key policy trusts `s3-replication-role` only. Edit the rule → IAM role → choose the existing role. |

---

## Per-bucket checklist (copy for each bucket)

```
Bucket: ______________  →  ______________

[ ] B1  Destination bucket created (versioning ON, SSE-KMS with dev key, bucket key ON)
[ ] B2  Destination bucket policy applied (name correct, includes ReplicateTags + OwnerOverride)
[ ] B3  Source bucket versioning ON
[ ] B4  Source KMS key ARN noted and decrypt access confirmed
[ ] B7  Role policy updated with source, destination and source KMS key ARNs
[ ] B5  Replication rule created (existing role, ownership change, KMS replicate with dev key ARN)
[ ] B6  Batch Replication job run: succeeded = total, failed = 0
[ ] B8  Verified: test upload, REPLICA status, new KMS key, counts match
[ ] B9  Apps cut over, rule disabled, old bucket retained then removed
```

---

## Notes

- Dev buckets can share **one** dev KMS key and **one** replication role. Create a separate key (and ideally a separate role) for prod.
- Optional speed-up: for small buckets, `aws s3 sync s3://<SRC> s3://<DEST>` run with credentials that can read the old bucket and write to the new one, avoids the replication setup entirely. Objects are re-encrypted with the destination bucket default.
- Not every bucket needs data migrated. Logs, flow logs, CloudTrail, Config, CUR, `cf-templates-*` and tfstate buckets can usually be recreated empty in the new account.
