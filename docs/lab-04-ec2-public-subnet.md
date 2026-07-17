# Lab 04 — Launch an EC2 in the public subnet (and debug why it wouldn't connect)

**Phase 2 · AWS Core + Identity** · Region `ap-south-1` (Mumbai) · Date: 2026-07-17

> Goal: launch a `t3.micro` into the public subnet of `cs-lab-vpc`, reach it over SSH from my
> laptop, and prove the public/private split actually works. What made this a *real* lab was that
> it didn't work on the first try — and the fix is the whole lesson.

---

## What I built

| Resource | Value |
|---|---|
| Instance | `vpc-lab-public-ec2` (`i-0dac420301da54d4a`) |
| Type / AMI | `t3.micro` · Amazon Linux 2023 (free-tier eligible) |
| VPC | `cs-lab-vpc` (`vpc-0abc52a28b394f7e7`) · `10.0.0.0/16` |
| Subnet | public `10.0.1.0/24` (`subnet-07bb9a5dee4cd00af`, `ap-south-1a`) |
| Security Group | `vpc-lab-sg` (`sg-070873d486c81b6e8`) — inbound SSH/22 from **My IP /32** only |
| Metadata | **IMDSv2 required** (blocks the SSRF → credential-theft path) |
| Public IP | auto-assigned (enabled at instance level) |

**Security choices, on purpose:**
- SSH restricted to a single `/32` (my own IP), never `0.0.0.0/0`. Open 22-to-the-world is the
  single most-scanned misconfiguration on the internet.
- **IMDSv2 required** — forces token-based access to the instance metadata service, closing the
  attack class behind the 2019 Capital One breach.
- Key pair (`.pem`) is **git-ignored** — no private keys in the repo.

---

## The problem: SSH connection timed out

After launch, the instance was **Running** with a public IP (`3.108.196.93`), but:

```
$ ssh -i vpc-lab-key.pem ec2-user@3.108.196.93
ssh: connect to host 3.108.196.93 port 22: Connection timed out
```

A **timeout** (not "connection refused") is a signal: the packet never reaches the host, or the
reply can't get back. That points at the **network path** — Security Group, route table, IGW, or
NACL — not the SSH service itself.

### Debugging, in order

**1. Security Group — ruled out.**
Inbound rule was correct: `SSH / TCP / 22 / Source 103.220.213.37/32` (my current IP). A good SG +
a timeout means the traffic isn't being *routed*, not *filtered*.

**2. Route table — found it.**
The public subnet was associated with route table `rtb-0bfad63dec0362c2b`, which contained **only**:

| Destination | Target |
|---|---|
| `10.0.0.0/16` | local |

There was **no `0.0.0.0/0 → igw` route.** The instance had a public IP but no path off the VPC — so
inbound SSH had no return route. **That is the root cause.**

> 🔑 The lesson in one line: *a public IP does not make a subnet public. The route to the Internet
> Gateway does.* Both are required.

### The fix (done the correct way — segmentation preserved)

Rather than dump an internet route onto a shared/main table (which would have exposed the private
subnet too), I created a **dedicated public route table**:

1. **Confirmed** the Internet Gateway `cs-lab-igw` was **Attached** to `cs-lab-vpc`.
2. **Created** route table `public-rt` in `cs-lab-vpc`.
3. **Added route:** `0.0.0.0/0` → **Internet Gateway** `cs-lab-igw`.
4. **Associated** only the public subnet `10.0.1.0/24` with `public-rt`.
5. Left the private subnet on its **local-only** table → still no inbound internet path.

Resulting public route table:

| Destination | Target |
|---|---|
| `10.0.0.0/16` | local |
| `0.0.0.0/0` | `cs-lab-igw` |

---

## Verification — it works

```
$ ssh -i vpc-lab-key.pem ec2-user@3.108.196.93
   ,     #_
   ~\_  ####_        Amazon Linux 2023
  ~~  \_#####\
...
[ec2-user@ip-10-0-1-55 ~]$
```

The hostname `ip-10-0-1-55` confirms the instance is inside `10.0.1.0/24` (the public subnet).

Proof of internet egress through the IGW, from inside the instance:

```bash
[ec2-user@ip-10-0-1-55 ~]$ curl -s https://checkip.amazonaws.com
3.108.196.93          # returns the instance's own public IP → outbound internet works
```

Segmentation proof, from my laptop:

```bash
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=vpc-0abc52a28b394f7e7" \
  --query "RouteTables[].Routes"
# public subnet's table → 0.0.0.0/0 via igw ; private subnet's table → local only
```

*(Screenshots: `docs/lab04-instance-running.png`, `docs/lab04-route-table.png` — add if desired.)*

---

## Takeaways (interview-ready)

1. **"Public" is a routing decision, not a checkbox.** A subnet is public only because its route
   table sends `0.0.0.0/0` to an Internet Gateway. A public IP alone does nothing without that route.
2. **Timeout vs. refused is a diagnostic.** *Timed out* → network path (routing/SG/NACL). *Refused*
   → reached the host but the port/service rejected it. Knowing the difference saved time.
3. **Debug from the outside in:** Security Group → route table → IGW attachment → NACL. The SG was
   fine, so the fault had to be routing.
4. **Fix without breaking segmentation:** a dedicated public route table keeps the internet route
   off the private subnet. One shared table with an IGW route would have quietly made everything
   public.

## Cleanup

- **Terminated** `vpc-lab-public-ec2` after the lab (its EBS volume deletes with it) → no overnight
  free-tier charges.
- No NAT Gateway or Elastic IP left allocated.

## Next

- **Lab 5 — 3-tier VPC:** add a data subnet + a NAT Gateway (outbound-only for private), across two
  AZs. This lab's public/private split is the seed for it.
