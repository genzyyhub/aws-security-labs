# aws-vpc-lab

A hands-on AWS VPC built from scratch in the console to learn **network segmentation** — the
control that turns a single compromised host into a *contained* incident instead of a full-estate
breach. This repo documents the build, the security reasoning behind each decision, and how to
verify it.

**Region:** `ap-south-1` (Mumbai) · **Phase 2 lab** of a cloud-security learning track.

---

## What this is

A custom VPC (`10.0.0.0/16`) split into a **public** and a **private** subnet, with an Internet
Gateway and route tables wired so that:

- the **public** subnet can send/receive traffic to the internet (via the IGW), and
- the **private** subnet has **no inbound path from the internet** — nothing outside can initiate a
  connection to it.

That single routing decision is the whole point: *a subnet is "public" only because its route table
sends `0.0.0.0/0` to the Internet Gateway.* Remove that route and the same subnet is private.

## Architecture

![VPC architecture](docs/architecture.svg)

```
VPC  cs-lab-vpc  10.0.0.0/16   (ap-south-1)
│
├── Public subnet   public-1a   10.0.1.0/24   (ap-south-1a)
│     route table: public-rt  →  0.0.0.0/0 via cs-lab-igw   ← this makes it "public"
│
├── Private subnet  private-1a  10.0.2.0/24   (ap-south-1a)
│     route table: local only  →  no 0.0.0.0/0 route        ← no inbound from internet
│
└── Internet Gateway  cs-lab-igw  (attached to VPC)
```

> A console screenshot of the built VPC lives at `docs/vpc-console.png`
> (save your AWS "Your VPCs" resource-map screenshot there).

## Design decisions (the security reasoning)

| Decision | Why |
|---|---|
| `/16` VPC, `/24` subnets | Room to grow (256 subnets of 256 hosts) without re-addressing; keeps CIDR math simple. |
| Separate public vs private subnets | Segmentation. The blast radius of a compromised public host stops at the subnet boundary. |
| Public route table → IGW | The **only** thing that grants internet reachability. Explicit, auditable, one line to revoke. |
| Private subnet has no IGW route | Defence in depth — even a misconfigured Security Group can't expose it inbound to the internet. |
| Private subnet reaches out via NAT (when needed) | Outbound-only: instances can pull updates, but nothing on the internet can initiate a connection in. |

### Security Groups vs NACLs (why both exist)

- **Security Group** = *stateful* "bouncer with memory," attached **per instance**. Allow-only
  rules; return traffic for an allowed request is automatically permitted.
- **NACL** = *stateless* "wall around the building," attached **per subnet**, evaluates **both**
  directions independently, supports explicit **deny**.

Layering a per-subnet NACL under per-instance SGs means one misconfigured SG doesn't undo your
subnet-level guardrail.

## How to verify what you built

Read the network back with the CLI (uses your least-privilege IAM user, never root):

```bash
aws ec2 describe-vpcs           --filters "Name=tag:Name,Values=cs-lab-vpc"
aws ec2 describe-subnets        --filters "Name=vpc-id,Values=<vpc-id>"
aws ec2 describe-route-tables   --filters "Name=vpc-id,Values=<vpc-id>"
aws ec2 describe-internet-gateways --filters "Name=attachment.vpc-id,Values=<vpc-id>"
```

Confirm: the public subnet's route table has a `0.0.0.0/0 → igw-...` route, and the private
subnet's route table does **not**.

## Cost / cleanup

Free-tier safe as built (VPC, subnets, route tables, and an Internet Gateway cost nothing).
**NAT Gateways and running EC2 instances do cost money** — delete/terminate them when you're done
so free-tier charges don't sneak in.

## What I'd do next

- Add a **NAT Gateway** so the private subnet gets outbound-only internet.
- Grow to a **3-tier VPC** (web / app / data subnets) across two AZs for high availability.
- Attach a **least-privilege Security Group** and a subnet **NACL**, and test them.
- Reconstruct the whole thing as **Terraform** and scan it with `checkov` / `tfsec`.

---

### ⚠️ Security note

No credentials, access keys, or `.csv` exports are included in this repo — the `.gitignore` blocks
them. Never commit AWS keys to GitHub; automated scanners find and abuse them within minutes.
