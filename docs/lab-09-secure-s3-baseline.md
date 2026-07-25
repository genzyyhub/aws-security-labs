# Lab 09 — Secure S3 Data Store — Phase 2 Closer

**Phase 2 · AWS Core + Identity** · Region `ap-south-1` (Mumbai) · Date: 2026-07-25

> Phase 2's last piece: an S3 bucket that can't be made public, can't be written to unencrypted,
> can't be reached over plain HTTP, and never truly loses data once versioned — proved with real
> denied and allowed API calls, not just console toggles.

---

## The one-line lesson

Default settings and enforced settings are not the same thing. Default encryption quietly protects
what people forget to specify; a Deny policy is what makes the requirement provable and
unbypassable. A secure bucket needs both — the safety net and the wall.

## What was built

- **Bucket:** `aryan-stealth-bpa`
- **Block Public Access:** all four settings on (`BlockPublicAcls`, `IgnorePublicAcls`,
  `BlockPublicPolicy`, `RestrictPublicBuckets`) — closes both the ACL and bucket-policy routes to
  public exposure, and neutralizes anything that already existed.
- **Default encryption:** SSE-S3 (`AES256`), bucket key enabled.
- **Versioning:** enabled.
- **Bucket policy:** two Deny statements — reject any request not over TLS, and reject any
  `PutObject` that doesn't declare `AES256` server-side encryption.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": ["arn:aws:s3:::aryan-stealth-bpa", "arn:aws:s3:::aryan-stealth-bpa/*"],
      "Condition": { "Bool": { "aws:SecureTransport": "false" } }
    },
    {
      "Sid": "DenyUnEncryptedObjectUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::aryan-stealth-bpa/*",
      "Condition": { "StringNotEquals": { "s3:x-amz-server-side-encryption": "AES256" } }
    }
  ]
}
```

Two Resource entries in the first statement on purpose: the bare bucket ARN covers bucket-level
actions (`ListBucket`, `PutBucketVersioning`), the `/*` ARN covers object-level actions
(`GetObject`, `PutObject`) — `s3:*` spans both action types, so both resource shapes are needed or
half the actions have nothing to attach to.

## Proving it live

| Test | Command | Result | Why |
|---|---|---|---|
| Upload, no encryption header | `put-object` (no `--server-side-encryption`) | **AccessDenied** — explicit deny, resource-based policy | `StringNotEquals` treats a missing header as "not AES256" → deny fires |
| Upload, `--server-side-encryption AES256` | `put-object --server-side-encryption AES256` | **Success** | header matches, no deny condition met |
| Plain-HTTP PUT | `curl -X PUT http://...:80` | **403 AccessDenied** | `aws:SecureTransport` = false → deny fires |
| Second upload, same key | `put-object` (encrypted) | new **VersionId** issued, old version retained | versioning active |

`aryan-admin` carries `AdministratorAccess` at the IAM level — full permissions — and the bucket
policy still blocked the unencrypted PUT. Same lesson as Lab 07's IAM work, from the other
direction: a resource-based Deny overrides even an administrator identity.

## Gotcha: versioning didn't take the first time

First pass, `put-bucket-versioning` was run but `get-bucket-versioning` came back **empty** — not an
error, just no `Status` field. The tell was in `list-object-versions`: a second upload to the same
key showed only **one** entry, with `"VersionId": "null"` — the literal string S3 returns for an
object in a bucket that was never actually versioned, not a real version ID. Re-ran
`put-bucket-versioning`, confirmed `get-bucket-versioning` returned `{"Status": "Enabled"}`, then
re-uploaded — this time the object got a real generated `VersionId`. Lesson repeated from Lab 05 and
Lab 08: read back what you built before trusting the command didn't error silently.

## Cost / cleanup

Deleted the specific object versions created during testing (a plain delete on a versioned bucket
only adds a delete-marker — removing the data requires targeting each `VersionId` explicitly). The
bucket itself, with BPA/encryption/versioning/policy intact, is left in place as the portfolio
artifact — empty S3 storage is free.

## What's next

- Phase 2 complete: EC2/VPC segmentation (Labs 04–06), least-privilege IAM (Lab 07), 3-tier VPC
  flagship (Lab 08), secure S3 baseline (this lab).
- Phase 3 — Security Fundamentals + Logging/Monitoring: CloudTrail → CloudWatch Logs, metric filters
  + alarms, EventBridge, first detection pipeline.
