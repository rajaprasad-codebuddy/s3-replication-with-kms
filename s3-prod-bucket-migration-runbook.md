# S3 Production Bucket Migration Runbook (Old AWS Account → New AWS Account)

Worked example: `agenthub-prod-s3-bucket` → `studio-prod-s3-bucket` (SSE-KMS, ap-south-1)

## Variables (replace per bucket)

| Item | Value used |
|---|---|
| Old account ID | `890290782969` |
| New account ID | `254265893962` |
| Region | `ap-south-1` |
| Source bucket (old) | `agenthub-prod-s3-bucket` |
| Destination bucket (new) | `studio-prod-s3-bucket` |
| Old KMS key (source objects) | `arn:aws:kms:ap-south-1:890290782969:key/7b7d6b19-2d07-4a73-b5f1-cfd44cf5c488` |
| New KMS key (destination) | `arn:aws:kms:ap-south-1:254265893962:key/bf423d2d-993a-4034-af23-3429dbca95e8` (alias `studio-prod-kms-key`) |
| Replication role (old account) | `s3-replication-role-studio-prod` |
| Report bucket (no replication rule) | `agenthub-development-agent-bucket` |

---

## 0. How it works

- Live replication copies only NEW objects after the rule is created.
- Existing objects need S3 Batch Replication.
- Versioning is required on both buckets.
- A replication config uses ONE role for all rules on that bucket, so use a separate role per prod bucket.
- Check the old objects' encryption (object level, not bucket default). Default can say SSE-S3 while objects are SSE-KMS.
- If objects are SSE-KMS: the role needs `kms:Decrypt` on the old key and the rule must have "Replicate objects encrypted with KMS" with a destination key.

---

## PART 1 - NEW account (254265893962)

### 1.1 Create KMS key
- KMS → Customer managed keys → Create key (Symmetric, Encrypt/decrypt, Single-Region, `ap-south-1`).
- Alias: `studio-prod-kms-key`.
- Copy the key ARN.

### 1.2 Key policy
Add this statement to the key policy (keep the default root `kms:*` statement):

```json
{
  "Sid": "AllowOldAccountReplicationRole",
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::890290782969:root" },
  "Action": ["kms:Encrypt", "kms:GenerateDataKey", "kms:DescribeKey"],
  "Resource": "*"
}
```

- Use the account root, not the role ARN. The role does not exist yet and the save fails with `MalformedPolicyDocumentException: invalid principals`.
- The old account's IAM role policy (1.5 below) is what restricts who can actually use it.

### 1.3 Create destination bucket
- Name: `studio-prod-s3-bucket`, region `ap-south-1`.
- Enable **Versioning**.
- Block Public Access: ON.
- Default encryption: SSE-KMS with `studio-prod-kms-key` (bucket key enabled).

### 1.4 Destination bucket policy
Bucket → Permissions → Bucket policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowReplicationFromOldAccount",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::890290782969:root" },
      "Action": [
        "s3:GetBucketVersioning",
        "s3:PutBucketVersioning",
        "s3:ReplicateObject",
        "s3:ReplicateDelete",
        "s3:ReplicateTags",
        "s3:ObjectOwnerOverrideToBucketOwner"
      ],
      "Resource": [
        "arn:aws:s3:::studio-prod-s3-bucket",
        "arn:aws:s3:::studio-prod-s3-bucket/*"
      ]
    }
  ]
}
```

- The console's auto-generated policy misses `ReplicateTags` and `ObjectOwnerOverrideToBucketOwner`. Use the full list above.

---

## PART 2 - OLD account (890290782969)

### 2.1 Create IAM role
- IAM → Roles → Create role → Trusted entity: **Custom trust policy**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "s3.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

- Name: `s3-replication-role-studio-prod`.

### 2.2 Add inline policy to the role
Name: `s3-replication-role-studio-prod-policy`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SourceBucketRead",
      "Effect": "Allow",
      "Action": ["s3:GetReplicationConfiguration", "s3:ListBucket"],
      "Resource": "arn:aws:s3:::agenthub-prod-s3-bucket"
    },
    {
      "Sid": "SourceObjectRead",
      "Effect": "Allow",
      "Action": [
        "s3:GetObjectVersionForReplication",
        "s3:GetObjectVersionAcl",
        "s3:GetObjectVersionTagging"
      ],
      "Resource": "arn:aws:s3:::agenthub-prod-s3-bucket/*"
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
      "Resource": ["arn:aws:s3:::studio-prod-s3-bucket/*"]
    },
    {
      "Sid": "DecryptOldKey",
      "Effect": "Allow",
      "Action": "kms:Decrypt",
      "Resource": "arn:aws:kms:ap-south-1:890290782969:key/7b7d6b19-2d07-4a73-b5f1-cfd44cf5c488"
    },
    {
      "Sid": "EncryptNewAccountKey",
      "Effect": "Allow",
      "Action": ["kms:Encrypt", "kms:GenerateDataKey"],
      "Resource": "arn:aws:kms:ap-south-1:254265893962:key/bf423d2d-993a-4034-af23-3429dbca95e8"
    }
  ]
}
```

### 2.3 Old KMS key policy
- The old key must have the default root-delegation statement (`arn:aws:iam::890290782969:root`, `kms:*`). Then the IAM `kms:Decrypt` above is enough.
- If the key policy has no root statement, add the role ARN as a principal with `kms:Decrypt`.

### 2.4 Old (source) bucket policy - check for Deny statements
- If the bucket policy has a Deny (e.g. outside-VPC / `DenyAccessOutsideAppVPC`), it blocks the replication role and the Batch job role too.
- Add both roles to the Deny's `aws:PrincipalArn` exemption list (inside the `StringNotLike` condition):

```json
"aws:PrincipalArn": [
  "arn:aws:iam::890290782969:user/raja-cdbd-devops",
  "arn:aws:iam::890290782969:user/agenthub-prod-project-user",
  "arn:aws:iam::890290782969:role/s3-replication-role-studio-prod",
  "arn:aws:iam::890290782969:role/*S3BatchOperations-REPLICATE_*"
]
```

- Keep the existing Allow statements (e.g. CloudFront `AllowCloudFrontServicePrincipal`) untouched.
- The two role entries are TEMPORARY. Remove them after migration.
- Symptom if missed: Batch job fails with "Access denied ... s3:PutInventoryConfiguration required for the role".

### 2.5 Create the replication rule
- Source bucket → Management → Replication rules → Create replication rule.
- Rule name: e.g. `replicate-to-studio-prod`.
- Status: Enabled.
- Scope: whole bucket.
- Destination: "Specify a bucket in another account" → account ID `254265893962`, bucket `studio-prod-s3-bucket`.
- Tick **Change object ownership to destination bucket owner**.
- IAM role: choose existing `s3-replication-role-studio-prod`.
- Encryption: tick **Replicate objects encrypted with AWS KMS**, enter the new key ARN `arn:aws:kms:ap-south-1:254265893962:key/bf423d2d-993a-4034-af23-3429dbca95e8`.
- Optional: enable Replication metrics.
- Save. When asked "Replicate existing objects?" choose **No** (do it via Batch in the next step, with the checks below).
- Older rules on the same bucket (e.g. DR rule to `agenthub-prod-s3-bucket-replication`) must use this same role. Disable or delete rules that are not needed.

### 2.6 Replicate existing objects (S3 Batch Replication)

Create the job from the replication rules page (not Clone):

- Source bucket → Management → Replication rules → select the rule → **Create replication job**.
- Do NOT use "Clone job". A clone carried the report bucket into the Source field and replicated the wrong bucket.
- Settings:
  - Scope: replicate objects that haven't been replicated yet (or all objects).
  - IAM role: **Create new IAM role** (creates `S3BatchOperations-REPLICATE_xxxxxx`).
  - Completion report: Failed tasks only (or All tasks).
  - Report destination bucket: a DIFFERENT bucket with no replication rule (here `agenthub-development-agent-bucket`).
  - Use **Review before running** (job goes to "Awaiting your confirmation").
- Before running, open the job → **Scope → Source** and confirm it is `agenthub-prod-s3-bucket`. Then Run job.
- Report bucket must NOT have a replication rule, or the reports replicate onward.
- If the role needs more rights, it must have on the source bucket: `s3:PutInventoryConfiguration`, `s3:InitiateReplication`, `s3:GetReplicationConfiguration`, plus write access to the report bucket.

Result in this migration: 718,946 / 718,946 succeeded, 0 failed, about 27 min.

---

## PART 3 - Verify

- Batch job: Status Completed, Failed = 0.
- Open a sample object in the destination bucket:
  - Replication status = **REPLICA**.
  - Server-side encryption = SSE-KMS with the NEW key ARN.
  - Owner = new account.
  - Metadata (Content-Type, Content-Encoding, etc.) preserved.
  - Last modified preserved.
- Compare counts: Metrics tab, or CLI per account:
  - `aws s3 ls s3://agenthub-prod-s3-bucket --recursive --summarize | tail -2`
  - `aws s3 ls s3://studio-prod-s3-bucket --recursive --summarize | tail -2`
- Test live replication: upload a new file to the old bucket, confirm it appears in the new bucket (status REPLICA).

---

## PART 4 - Prepare the new bucket for the apps (before cutover)

- Add the new-account apps' IAM roles/users to the destination bucket policy (read/write) and to the `studio-prod-kms-key` policy (`kms:Decrypt`, `kms:GenerateDataKey`, `kms:Encrypt`).
- Recreate old access controls on the new bucket:
  - CloudFront OAC statement (with the NEW distribution ARN).
  - VPC / VPC endpoint restriction, if used.
- Update app configs, env vars and CORS to the new bucket name.
- Keep the Deny exemption entries on the old bucket until the migration finishes.

---

## PART 5 - Cutover and cleanup

- [ ] Final live-replication check (new uploads appear in the new bucket).
- [ ] Switch apps to `studio-prod-s3-bucket` and verify reads, writes and CloudFront.
- [ ] Disable the replication rule on the old bucket.
- [ ] Remove the two temporary role entries from the Deny exemption on `agenthub-prod-s3-bucket`.
- [ ] Delete the unused Batch job roles (`S3BatchOperations-REPLICATE_*`).
- [ ] Keep the old bucket (and old KMS key) until apps are verified. Do not delete early.
- [ ] KMS keys have a 7-30 day deletion wait. Schedule deletion of the old key only after all data is confirmed.

---

## Troubleshooting

| Symptom | Cause / Fix |
|---|---|
| KMS policy save: `MalformedPolicyDocumentException` invalid principals | Role doesn't exist yet. Use the old account root as principal, or create the role first |
| Objects don't replicate (Failed) | Rule missing "Replicate KMS objects", or role lacks `kms:Decrypt` on the old key / `kms:Encrypt` on the new key |
| `AccessDenied` on destination | Destination bucket policy missing `ReplicateTags` / `ObjectOwnerOverrideToBucketOwner`, or new key policy missing old account principal |
| Batch job fails: `s3:PutInventoryConfiguration required for the role` | Source bucket Deny blocks the Batch role. Add it to the `aws:PrincipalArn` exemption |
| Batch job replicated the wrong bucket | Created via Clone. Create from the replication rules page and check Scope → Source |
| Destination still empty after job | Wrong source in the job. Check the job's source bucket |
| Reports show up in the destination | Report bucket has its own replication rule. Use a bucket without one |
| Default encryption says SSE-S3 but objects use KMS | Check the object's encryption, not the bucket default. Treat as KMS |
| Existing objects not copied | Live replication is new-only. Use Batch Replication |

---

## Checklist (per prod bucket)

- [ ] New KMS key created + key policy (old account root)
- [ ] New bucket created, versioning ON, SSE-KMS default
- [ ] Destination bucket policy applied (full 6 actions)
- [ ] New role + inline policy created in old account
- [ ] Old key policy has root delegation
- [ ] Source Deny policy exemption updated (role + Batch role)
- [ ] Replication rule created (KMS replicate, ownership override, new role)
- [ ] Batch job created from the replication rules page, source verified, run
- [ ] Sample object verified as REPLICA with the new key
- [ ] Object counts compared, live replication tested
- [ ] New bucket/key access granted to apps
- [ ] Cutover done, temporary exemptions removed
