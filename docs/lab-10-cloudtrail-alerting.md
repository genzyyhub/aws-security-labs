# Lab 10 — CloudTrail → CloudWatch → Alarm → SNS — Phase 3 Flagship

**Phase 3 · Security Fundamentals + Logging/Monitoring** · Region `eu-north-1` (Stockholm) · Date: 2026-07-30

> Turning CloudTrail from a passive recorder into an actual detection: an alarm that fires and
> emails a human when someone signs in as the root account — built CLI-first and proved with a real
> root login, a real state transition, and a real notification.

---

## The one-line lesson

Logging and detection are not the same thing. CloudTrail has been recording every API call in this
account since Lab 03 — and nobody was reading it. The gap between *collecting* logs and *being told
something happened* is where most real-world breaches live undetected for months.

## The five-link chain

```
event happens
  → CloudTrail        records the API call
  → CloudWatch Logs   receives it live
  → metric filter     turns matching lines into a number
  → CloudWatch Alarm  watches that number
  → SNS               tells a human
```

Break any one link and you're back to logging. Each link below is a separate AWS resource with its
own failure mode — which is exactly why this is worth building by hand once.

## What was built

| Resource | Name | Purpose |
|---|---|---|
| CloudTrail trail | `account-activity-trail` | Multi-region, global service events **on** (required — root sign-in is a global event recorded in us-east-1) |
| CloudWatch log group | `/cloudtrail/security-events` | Destination, 7-day retention |
| IAM role | `CloudTrail-CWLogs-Role` | The identity CloudTrail assumes to write into the log group |
| Metric filter | `RootLoginFilter` | Matches root console logins → `RootLoginCount` |
| SNS topic | `security-alerts` | Email subscription (confirmed) |
| CloudWatch alarm | `RootLoginAlarm` | Sum ≥ 1 over 300s → publishes to the topic |

### The IAM role — least privilege applied for real

Two policies doing two different jobs. Confusing them is the most common IAM misconception.

**Trust policy** — *who* may become this role:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "cloudtrail.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
```

**Permission policy** — *what* it can do once it is that role:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["logs:CreateLogStream", "logs:PutLogEvents"],
    "Resource": "arn:aws:logs:eu-north-1:<acct>:log-group:/cloudtrail/security-events:log-stream:*"
  }]
}
```

Two verbs, scoped to the streams of **one** log group. Not `logs:*`, not `"Resource": "*"`. If these
credentials leaked, the blast radius is *"an attacker can append lines to one log file"* — which is
close to nothing. That's the point of least privilege: it doesn't prevent the breach, it makes the
breach worthless.

### The metric filter — where security judgment gets injected

```bash
aws logs put-metric-filter --region eu-north-1 \
  --log-group-name /cloudtrail/security-events \
  --filter-name RootLoginFilter \
  --filter-pattern '{ $.userIdentity.type = "Root" && $.eventName = "ConsoleLogin" }' \
  --metric-transformations \
      metricName=RootLoginCount,metricNamespace=SecurityMetrics,metricValue=1,defaultValue=0
```

CloudTrail records a root login with exactly the same enthusiasm as a routine `DescribeInstances` —
it has no concept of "suspicious." **The metric filter is where a human declares what matters.**
Everything before this step is plumbing; this is the security decision.

`defaultValue=0` matters more than it looks: without it the metric publishes *nothing* when no root
login occurs, and the alarm sits in `INSUFFICIENT_DATA` permanently instead of a healthy `OK`.
Explicit zeros are what make "nothing is wrong" observable.

**Stricter CIS variant** (CIS AWS Foundations Benchmark control 1.7) — catches *all* root usage, not
just console logins:

```
{ $.userIdentity.type = "Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != "AwsServiceEvent" }
```

The two extra clauses strip noise: `invokedBy NOT EXISTS` excludes calls AWS services made on your
behalf, `eventType != "AwsServiceEvent"` excludes AWS's own platform events. Without them you get
false positives from routine activity.

### The alarm

```bash
aws cloudwatch put-metric-alarm --region eu-north-1 \
  --alarm-name RootLoginAlarm \
  --namespace SecurityMetrics --metric-name RootLoginCount \
  --statistic Sum --period 300 --evaluation-periods 1 \
  --threshold 1 --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions "$TOPIC"
```

- **`Sum`, not `Average`** — this is event counting. Average would have reported `1.0` for the two
  logins that actually occurred and hidden the second one. (See the live proof below: the datapoint
  was `2.0`.)
- **`evaluation-periods 1`** — fire on the first bad window. Zero tolerance is right for a security
  event; a noisy metric like CPU would use 2–3 to stop it flapping.
- **`treat-missing-data notBreaching`** — no data means fine, because we're counting bad events. A
  heartbeat check would use `breaching` instead, where silence means the thing died.
- **`--alarm-actions` is the entire point.** An alarm without an action is a red light in a room
  nobody is standing in — and that's a real production failure mode, not a hypothetical.

## Proving it live

Signed in to the console as root once, then checked:

```
- State Change:     OK -> ALARM
- Reason:           Threshold Crossed: 1 datapoint [2.0 (30/07/26 17:15:00)]
                    was greater than or equal to the threshold (1.0).
- Timestamp:        Thursday 30 July, 2026 17:20:12 UTC
- Alarm Arn:        arn:aws:cloudwatch:eu-north-1:<acct>:alarm:RootLoginAlarm
```

Email delivered to the subscribed inbox. The full chain — sign-in → CloudTrail → CloudWatch Logs →
metric filter → alarm → SNS → inbox — fired end to end.

The datapoint reading `2.0` is itself a small proof: two root `ConsoleLogin` events landed inside the
same 300-second window and `Sum` counted both, exactly as intended.

## Gotchas — the four things that actually cost time

### 1. A multi-region trail still has exactly one home region

`update-trail` against `ap-south-1` (where every other lab in this repo lives) returned:

```
TrailNotFoundException: Unknown trail: arn:aws:cloudtrail:ap-south-1:<acct>:trail/account-activity-trail
```

`describe-trails` happily lists the trail from *any* region — it shows shadow copies for visibility —
but **every mutating API call must target the trail's actual home region**, which turned out to be
`eu-north-1`. Find it with:

```bash
aws cloudtrail describe-trails --query 'trailList[*].{Name:Name,Home:HomeRegion}' --output table
```

### 2. The log group must live in the home region too

After fixing the region on the call itself:

```
InvalidCloudWatchLogsLogGroupArnException: You must specify a log group that is in the current region.
```

CloudTrail will **not** deliver cross-region. The log group had to be recreated in `eu-north-1`, and
the IAM permission policy's resource ARN updated to match. This cascades further than it first
appears — the region affinity chain is:

```
trail home region → log group → metric filter → custom metric → alarm → SNS topic
```

All six in the same region. Put the SNS topic in the wrong one and `put-metric-alarm` gives you an
alarm that never fires, **with no error message**, because an alarm watching a metric that doesn't
exist in its region simply sits in `INSUFFICIENT_DATA` forever.

### 3. IAM changes need a moment to propagate

The first `update-trail` immediately after creating the role failed with `Access denied. Verify in
IAM that the role has adequate permissions.` The policy was correct — IAM is eventually consistent
and the change hadn't propagated yet. Retrying seconds later succeeded. Worth recognising so you
don't rewrite a policy that was already right.

### 4. Alarms don't latch — current state is not proof

Post-test, `describe-alarms` showed `OK` with *"1 datapoint [0.0] was not greater than or equal to
the threshold"*, which reads like failure. It wasn't: the alarm had already fired at 17:20:12, then
the next 5-minute window contained zero root logins and it correctly reset to `OK`.

**The right question isn't "what state is the alarm in," it's "has it ever fired":**

```bash
aws cloudwatch describe-alarm-history --region eu-north-1 \
  --alarm-name RootLoginAlarm --history-item-type StateUpdate \
  --query 'AlarmHistoryItems[*].{When:Timestamp,Summary:HistorySummary}' --output table
```

State history is the evidence. Current state is a snapshot that hides everything that already
happened.

### Bonus: CloudShell resets environment variables

CloudShell wipes exported variables between sessions, and **root's CloudShell is a completely
separate home directory from an IAM user's** — so neither the exports nor `~/.bashrc` carry across.
Several confusing `ParamValidation: expected one argument` errors traced back to an empty `$LG` or
`$TOPIC` rather than anything wrong with AWS. Fix: persist static values in `~/.bashrc` (CloudShell
does keep the home directory), and use literal values in short-lived or cross-identity shells.

## Interview answers this lab earns

- **"How would you detect and get notified about a root account login?"** — CloudTrail with global
  service events enabled → CloudWatch Logs → metric filter on `userIdentity.type = Root` →
  alarm on Sum ≥ 1 → SNS. Named as CIS AWS Benchmark control 1.7.
- **"CloudTrail vs CloudWatch?"** — CloudTrail records *what happened* (API audit trail);
  CloudWatch *reacts* (metrics, alarms, notification). One is the recorder, the other is the reflex.
- **"What regional constraints apply?"** — a multi-region trail has one home region; the log group,
  metric, alarm and SNS topic must all sit in it. Cross-region delivery isn't supported and fails
  silently at the alarm stage.

## Cost / cleanup

**Nothing to tear down** — unlike the VPC labs, where the NAT Gateway had to be deleted before bed,
this is designed to keep running. It's a live detector on the account from now on.

Everything sits inside free tier: one trail (the first per region is free — a second bills per
event), one log group at 7-day retention, one custom metric, one alarm (10 free), one SNS topic
(1,000 free email notifications).

```bash
aws cloudtrail describe-trails --query 'length(trailList)'   # must return 1
```

Log group retention is the sleeper cost here: the default is *never expire*, and CloudTrail is
chatty. Setting 7-day retention is what keeps this free.

## What's next

- Phase 3 remaining: **Security+ domains** (threats/attacks, cryptography applied to KMS/TLS,
  architecture, operations & IR, governance) and **EventBridge** for event-driven detection routing.
- Then Phase 4 — Terraform + Python/boto3, with Checkov/tfsec scanning and Prowler posture checks.
