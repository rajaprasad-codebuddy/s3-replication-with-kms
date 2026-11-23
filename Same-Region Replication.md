# S3 Same-Region Replication with a Separate KMS Key — Production Runbook

Proven on the ReEnvisionLab account (us-east-1) on 2026-11-23 with no downtime.

## Overview

This runbook sets up one-way, same-region replication from a production S3 bucket to a replica bucket in the same AWS account, with the replica encrypted by its own KMS key.

- **Result:** new objects copy to the replica within minutes. Source objects stay on the source key; replicas are re-encrypted with the replica key.
- **Why a separate key:** disabling or deleting one key cannot make both buckets unreadable.
- **Deletes are not mirrored:** delete marker replication stays off, so deleting an object in the source leaves the replica copy intact.
- **Scope:** same account, same region only.
- **Impact on the application:** none. Replication runs in the background, and the app keeps using the source bucket and source key.

How it works: S3 assumes a replication IAM role, reads each new object version from the source (decrypting with the source key), and writes it to the replica (encrypting with the replica key). The role needs KMS permissions on both keys, which is the step most often missed (Step 4).

## Variables to fill in per project

Fill in this table first for each account; every later step refers to these names. The example column is the ReEnvisionLab setup.

| Variable | Example | Notes |
| --- | --- | --- |
| `ACCOUNT_ID` | `445137568670` | Account that owns both buckets and both keys |
| `REGION` | `us-east-1` | Must be the same for source and replica |
| `SRC` | `reenvision-prod-s3-bucket` | Existing production bucket |
| `DST` | `reenvision-prod-s3-bucket-replica` | Replica bucket, same region |
| `SRC_KEY_ARN` | `arn:aws:kms:us-east-1:445137568670:key/cb74ee23-820c-4f71-8ce6-a01e4420cf85` | Existing key on the source bucket |
| `REPLICA_KEY_ALIAS` | `reenvision-prod-s3-replica-kms-key` | New key, created in Step 1 |
| `REPLICA_KEY_ARN` | `arn:aws:kms:us-east-1:445137568670:key/acfeef2d-922e-4c00-9805-98b121fe63e2` | Known after Step 1 |
| `REPL_ROLE` | `s3crr_role_for_reenvision-prod-s3-bucket` | Created by the console in Step 3. Used for replication. |
| `BATCH_ROLE` | `s3replicate_role_for_reenvision-prod-s3-bucket` | Created in Step 5. Used only by the Batch job. |
| `REPORT_LOC` | `s3://<logs-bucket>/batch-replication-reports/` | Where the Batch completion report is written |

## Prerequisites

Check all of these before starting; none of them changes anything in production.

- [ ] Source bucket has **versioning enabled**. Replication cannot be created without it.
- [ ] Replica bucket exists in the **same region** and the same account, with versioning enabled.
- [ ] Source bucket default encryption is SSE-KMS with a customer managed key. Note its ARN as `SRC_KEY_ARN`. (If the source uses only SSE-S3, skip the source-key statement in Step 4.)
- [ ] You can create KMS keys and IAM roles, and can edit replication rules (`s3:PutReplicationConfiguration`, `iam:CreateRole`, `iam:PutRolePolicy`, `iam:PassRole`, `kms:CreateKey`).
- [ ] No service control policy or bucket policy blocks `s3:ReplicateObject`, `kms:Encrypt` or `kms:Decrypt` for the new role.
- [ ] A change window is agreed. Replication adds no downtime, but a Batch job on a large bucket generates many KMS requests.
- [ ] A logs or utility bucket exists for the Batch completion report (Step 5).

## Step 1: Create the replica KMS key

Create a new single-Region symmetric key in the same region as the buckets. A multi-Region key is not needed for same-region replication.

1. Open **KMS → Customer managed keys → Create key**.
2. Key type **Symmetric**, usage **Encrypt and decrypt**, regionality **Single-Region key**.
3. Alias: `<project>-prod-s3-replica-kms-key`. Description: the same words. Tag it with `Project = <project>`.
4. Key administrators: your admin role only. Do not add the replication role here; it gets access through IAM in Step 4.
5. Keep the default key policy statement that delegates to the account root (`Enable IAM User Permissions`). Without it, the IAM policy in Step 4 has no effect.
6. Optional: open **Key material and rotations** and enable **automatic key rotation**.
7. Copy the key ARN into `REPLICA_KEY_ARN`.

The default key policy created by the console looks like this and is enough:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::<ACCOUNT_ID>:root" },
      "Action": "kms:*",
      "Resource": "*"
    }
  ]
}
```

## Step 2: Prepare the replica bucket

Set the replica bucket's default encryption to the new key so anything written to it directly matches the replicated objects.

1. Open the replica bucket → **Properties → Bucket Versioning**. It must show **Enabled**.
2. Go to **Properties → Default encryption → Edit**.
3. Encryption type **SSE-KMS**. Choose `REPLICA_KEY_ARN`. Set **Bucket Key** to **Enable**.
4. Save. The console warns that changing default encryption can fail in-progress replication if the role lacks KMS permissions. That is expected here because the rule does not exist yet.
5. Confirm **Block all public access** is on and add the `Project` tag.

The rule in Step 3 sets the key for replicated objects explicitly, so this default only matters for direct writes. It also keeps the two buckets consistent for audits.

## Step 3: Create the replication rule

Create one rule on the source bucket that replicates everything to the replica bucket and re-encrypts it with the replica key.

Open the **source** bucket → **Management → Replication rules → Create replication rule** and set:

| Section | Setting | Value |
| --- | --- | --- |
| Rule | Name / Status | `<project>-prod-to-replica` / Enabled |
| Source | Rule scope | Apply to all objects in the bucket |
| Destination | Bucket | A bucket in this account → `DST` (same region) |
| IAM role | Permission | **Create new role** |
| Encryption | KMS objects | **Replicate** |
| Encryption | Use source Multi-Region key | Unchecked |
| Encryption | Specify destination key | **Checked**, then choose `REPLICA_KEY_ARN` |
| Storage class | Change storage class | Unchecked |
| Options | Replication Time Control | Unchecked (15-minute SLA, extra cost) |
| Options | Replication metrics | Unchecked (optional: CloudWatch monitoring of pending and failed objects) |
| Options | Delete marker replication | **Unchecked** (replica keeps objects deleted in the source) |
| Options | Replica modification sync | Unchecked (keeps replication one-way) |

Click **Save**. The console then asks whether to replicate existing objects. Choose **Yes** only if the source already holds data, and follow Step 5; otherwise choose No.

The console creates the role `s3crr_role_for_<SRC>` with a managed policy `s3crr_for_<SRC>_<suffix>`. It adds broad KMS statements (`Resource: "*"`) and no `kms:GenerateDataKey`. Step 4 adds the exact permissions.

## Step 4: Give the replication role KMS permissions

Attach one inline policy to the replication role. It allows Decrypt on the source key and Encrypt plus GenerateDataKey on the replica key, each limited to the matching bucket.

1. Open **IAM → Roles → `s3crr_role_for_<SRC>`**. This is the role named on the replication rule (Management → Replication rules), not the Batch role.
2. **Add permissions → Create inline policy → JSON**. Name it `s3-replication-with-kms-keys`.
3. Paste the policy below with your values substituted, then save.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DecryptSourceKey",
      "Effect": "Allow",
      "Action": "kms:Decrypt",
      "Resource": "<SRC_KEY_ARN>",
      "Condition": {
        "StringLike": {
          "kms:ViaService": "s3.<REGION>.amazonaws.com",
          "kms:EncryptionContext:aws:s3:arn": [
            "arn:aws:s3:::<SRC>",
            "arn:aws:s3:::<SRC>/*"
          ]
        }
      }
    },
    {
      "Sid": "EncryptReplicaKey",
      "Effect": "Allow",
      "Action": ["kms:Encrypt", "kms:GenerateDataKey"],
      "Resource": "<REPLICA_KEY_ARN>",
      "Condition": {
        "StringLike": {
          "kms:ViaService": "s3.<REGION>.amazonaws.com",
          "kms:EncryptionContext:aws:s3:arn": [
            "arn:aws:s3:::<DST>",
            "arn:aws:s3:::<DST>/*"
          ]
        }
      }
    }
  ]
}
```

Notes:

- **Two bucket ARNs per statement** are needed because Bucket Key is on: the encryption context is then the bucket ARN, not the object ARN.
- **Use an inline policy**, not an edit to the console-generated managed policy. The console can regenerate that policy when the rule is edited.
- **The key policies must allow the role.** With the default root-delegation statement on both keys, the IAM policy above is enough. If a key policy lists specific principals, add the role there too.
- The console-generated managed policy stays as it is. The inline policy adds the exact KMS permissions on top of it.

## Step 5: Replicate existing objects

A replication rule only copies objects written after it exists. Existing objects need a one-time S3 Batch Replication job. Skip this step if the source bucket is empty.

1. Finish Step 4 first. The job copies objects through the replication role, so that role must already have the KMS permissions.
2. Open the source bucket → **Management → Replication rules**, select the rule, and choose the option to replicate existing objects. This opens **Create Batch Operations job**. The same job is also offered right after you save the rule in Step 3.
3. **Job run options:** Wait to run the job when it's ready. This lets you check the object count before it starts.
4. **Completion report:** tick *Generate completion report*, scope **All tasks**, destination `REPORT_LOC`. Use a logs bucket, not the data bucket, if one exists.
5. **Manifest:** saving it is optional; the job page shows the object count anyway.
6. **Permissions:** choose **Create new role**. The console names it `s3replicate_role_for_<SRC>` with `s3:InitiateReplication`, `s3:GetReplicationConfiguration` and `s3:PutInventoryConfiguration` on the source bucket. If you add the report, make sure the role can also `s3:PutObject` to `REPORT_LOC`.
7. Save, wait for status **Awaiting your confirmation to run**, check the total objects, then click **Run job**.
8. When it finishes, open the report. Failed tasks must be 0.

Points that cost time on the first rollout:

- The Batch role and the replication role are different roles. KMS permissions belong on the **replication role**, not the Batch role.
- The status *Failed* with the reason "Manifest generation found no keys matching the filter criteria" is not an error. It means there is nothing to copy (an empty bucket or everything already replicated). Clone the job later if data arrives.
- On a large bucket the job makes many KMS requests. Run it off-peak and watch KMS request quotas.

## Step 6: Verify replication and encryption

Upload a small test file to the source bucket and wait one to two minutes. Replication is correct only when both copies show the expected status and key.

| Check | Where | Expected |
| --- | --- | --- |
| Source copy status | Source bucket → object → Properties → Replication status | `COMPLETED` |
| Source copy key | Same page → Server-side encryption settings | `SRC_KEY_ARN` |
| Replica copy status | Replica bucket → object → Properties → Replication status | `REPLICA` |
| Replica copy key | Same page → Server-side encryption settings | `REPLICA_KEY_ARN` (the new key) |
| Integrity | ETag on both objects | Identical |

The same check from the CLI:

```bash
aws s3api head-object --bucket "$SRC" --key <test-key> \
  --query '{Replication:ReplicationStatus,Key:SSEKMSKeyId,ETag:ETag}'
aws s3api head-object --bucket "$DST" --key <test-key> \
  --query '{Replication:ReplicationStatus,Key:SSEKMSKeyId,ETag:ETag}'
```

If the replica shows the source key, the rule's destination key was not applied: edit the rule and set it. If the status is `FAILED`, check the KMS permissions on the replication role (Step 4). Delete the test object only in the source; the replica copy stays because delete markers are not replicated.

## Per-account rollout checklist

Copy this list for each project and account, tick it off in order, and add a row to the tracker below when the account is done.

- [ ] Fill in the variables table (account, region, bucket names, source key ARN).
- [ ] Confirm versioning is on for both buckets and both are in the same region.
- [ ] Step 1: create the replica key, record `REPLICA_KEY_ARN`.
- [ ] Step 2: set the replica bucket default encryption to the replica key (Bucket Key on).
- [ ] Step 3: create the replication rule with the destination key set; note the role name it creates.
- [ ] Step 4: attach the inline KMS policy to that role.
- [ ] Step 6: upload a test file; check `COMPLETED` on the source and `REPLICA` plus the replica key on the copy.
- [ ] Step 5: if the bucket holds data, run the Batch job and confirm zero failed tasks in the report.
- [ ] Optional: enable automatic rotation on the replica key.
- [ ] Record the account in the tracker.

Tracker (add a row per account):

| Project | Account ID | Source bucket | Replica bucket | Replica key ARN | Completed |
| --- | --- | --- | --- | --- | --- |
| ReEnvisionLab (reenvision-prod) | 445137568670 | `reenvision-prod-s3-bucket` | `reenvision-prod-s3-bucket-replica` | `arn:aws:kms:us-east-1:445137568670:key/acfeef2d-922e-4c00-9805-98b121fe63e2` | 2026-11-23 |
