# Task 3: TEST environment like-for-like with production

**Name:** Roshan Joshi  
**Task:** 3 (Cloud · Ops)  
**Time spent:** ~3 hours

Design / thinking only. No live AWS account, no real credentials, no restore against
Jobbser Prod.

---

## What I read first

I used the AWS inventory pack and skimmed the DB schema pack for PII columns.

Jobbser on AWS today (inventory shape):

- Region typically **eu-west-1**
- ECS Fargate for **Jobbser-Api** and **JobbserMsgHub**
- ALB + VPC (private tasks, NAT where used)
- RDS MySQL (own primary DB; schema pack shows heavy PII)
- S3 uploads, SES email, Secrets Manager for runtime config
- Clients hit API + hub over HTTPS / realtime via the ALB
- Outside AWS: Stripe, Firebase, GitHub Actions, App Store / Play

Environments in the inventory: **Prod** (always-on), **Dev** (expensive when left on;
sometimes switched off), **TEST** (aspirational, Task 3).

Workers stay free in the product. That does not change this design. I am not inventing
worker billing infra.

---

## What “like-for-like” means here

Same *kinds* of services, networking pattern, and app deploy path as Prod. Not a
blind clone of every Prod size and Multi-AZ knob.

| Layer | Prod (inventory) | TEST (this design) |
| --- | --- | --- |
| Region | eu-west-1 | eu-west-1 |
| ECS services | Jobbser-Api + JobbserMsgHub | Same two services, TEST cluster / task families |
| Entry | ALB HTTPS | Own ALB, TEST DNS names |
| Network | VPC, awsvpc, NAT as used | Own VPC (or clearly isolated CIDR); no peer that can reach Prod RDS |
| Data | Own RDS MySQL | **Own** RDS MySQL (never shared) |
| Assets | S3 | Own TEST bucket |
| Email | SES | Own config / sandbox or restricted identities |
| Config | Secrets Manager per env | Own secret(s); Prod ARNs forbidden |
| Images | ECR | Same ECR repo OK; TEST tags / digests |
| Deploy | CI → ECS service update | Same pipeline shape, `environment: test` |

I would size TEST a notch under Prod where the inventory already hints at small
Fargate and small RDS classes. Shape stays the same. Bill does not.

Longer diagram + parity table: [`docs/architecture.md`](docs/architecture.md).

---

## Own database + data strategy (chosen)

**Chosen approach: anonymised Prod snapshot → restore onto TEST RDS → scrub before open.**

Why this one:

1. Release checks need realistic row shapes and join quirks. Pure synthetic data is
   cleaner for privacy, but it often misses the weird cases that break migrations
   (orphan applications, mixed `job_posts` / hire-ledger `jobs`, polymorphic
   addresses, etc.).
2. Raw Prod restore into TEST is not acceptable. The schema pack has emails, hashed
   passwords, tokens, addresses, names. Engineers should not browse live PII in TEST.
3. TEST must own its instance. Snapshot restore creates a separate RDS. Prod keeps
   running. No shared connection string.

I considered pure synthetic seed as the main path. I would still keep a small
synthetic fixture set for empty-bootstrap and CI schema smoke. For “is this release
safe against something that looks like Prod?”, anonymised snapshot wins for me.

Honest trade-offs, steps, PII map, rollback: [`docs/data-refresh.md`](docs/data-refresh.md).  
Stretch playbook + smoke list: [`docs/runbook-refresh.md`](docs/runbook-refresh.md).

---

## Isolation and blast radius

Assumptions where the inventory is silent (called out as such):

1. **Prefer a separate AWS account for TEST** (or at least a hard account boundary
   from Prod). Same-account VPCs with careful SGs are workable early, but one wrong
   SG or secret ARN and you are in Prod. I would ask this in kickoff.
2. **No VPC peering / Transit Gateway path from TEST to Prod RDS.** TEST tasks talk
   only to TEST MySQL.
3. **Separate Secrets Manager secrets.** Naming like `jobbser/test/api-runtime`.
   Prod JWT / Stripe live keys never land in TEST. Stripe test keys + SES sandbox
   (or a dedicated TEST from-address) so a mistaken loop cannot email real users or
   charge real companies.
4. **Separate S3 bucket** (`jb-test-uploads` style). No Prod bucket IAM on the TEST
   task role.
5. **DNS:** `api.test.…` and `hub.test.…` (exact zone TBD) pointing at the TEST ALB.
   Mobile / web builds that target TEST use those hostnames only.
6. **IAM:** TEST deploy role can update TEST ECS services and read TEST secrets only.
7. **Humans:** engineers and CI can reach TEST. Prod break-glass stays a smaller set.
   Bastion / SSM into TEST only; never reuse a Prod jump host that also has Prod DB
   credentials loaded by default.

Blast radius if TEST is compromised: TEST data (already scrubbed), TEST secrets,
TEST bucket. Not Prod DB, not live Stripe, not live user email.

---

## Cost (order of magnitude)

Inventory “Cost reality”: most spend sat in **EC2/NAT + public IPv4 + ECS/Fargate +
ALB**, with **RDS comparatively small**. Always-on Dev was painful enough that it
got switched off.

So a second always-on cluster that mirrors Prod will mostly double that non-prod-ish
footprint if I am careless. My posture:

- Same shape (API + MsgHub + ALB + VPC/NAT + RDS + S3 + Secrets + logs).
- Slightly smaller than Prod: keep Fargate near the inventory small tier
  (0.25 vCPU / 0.5 GB class) with **lower desired counts** than Prod where release
  traffic allows; **single-AZ RDS** for TEST unless we are specifically testing
  failover; shorter log retention; lifecycle on the TEST bucket.
- Do **not** also leave Dev always-on next to TEST. Inventory already taught that
  lesson. Dev parks; TEST stays up for release checks, or parks outside freeze
  windows if the team agrees.
- Shared ECR is fine (images are cheap vs compute).

Rough monthly band if TEST is left always-on at small sizes: **on the order of a few
hundred USD**, dominated by ALB hours + NAT/public IPv4 + two small Fargate services,
not by the micro-class RDS. Exact numbers move; the ordering matches the inventory.

More breakdown: [`docs/cost.md`](docs/cost.md).

---

## Deploy path (parity with Prod)

Same story Prod already uses, with a TEST target:

1. Build / push image tags to ECR (API and MsgHub).
2. GitHub Actions (or current CI) updates the **TEST** ECS services to those digests.
3. Task defs pull **TEST** Secrets Manager ARN at start.
4. ALB health checks on TEST listeners must go green before anyone calls the release
   “TEST-verified”.

I am not inventing a second deploy technology. Like-for-like includes the path, not
just the boxes.

---

## Stretch I included

- Automated / manual refresh playbook ([`docs/runbook-refresh.md`](docs/runbook-refresh.md))
- Smoke / cutover checklist after refresh (same file)
- Feature-flag note: keep TEST pointing at Stripe test mode and a “TEST” banner /
  config flag in the API secret so clients know they are not in Prod
- Optional park schedule outside release weeks (cost), without deleting the env

---

## What I would ask in a kickoff

1. Separate AWS account for TEST, or same account with strict VPC isolation first?
2. Who is allowed to create / copy Prod RDS snapshots, and is there already an
   approved anonymisation script or do we write one?
3. Exact TEST hostnames and who owns the DNS zone?
4. Should MsgHub be in the first TEST cut (I say yes for like-for-like), or API-only
   for week one?
5. SES: sandbox + verified TEST addresses, or a dedicated subdomain?
6. Is overnight parking of TEST OK outside release freezes, given Dev’s cost history?
7. Which Stripe / Firebase projects are the approved non-prod ones?
8. Retention: how long may scrubbed TEST snapshots live in the TEST account?

---

## Testing approach

Required for the brief. Short version here; longer in
[`docs/testing-approach.md`](docs/testing-approach.md).

How I would prove TEST is like-for-like enough, isolated, and safe:

1. **Parity checklist** against the inventory: both ECS services present, ALB,
   private MySQL, own secret, own bucket, TEST DNS.
2. **Isolation probes:** from a TEST task, confirm Prod RDS endpoint is unreachable;
   confirm task role cannot `GetSecretValue` on Prod ARNs; confirm no Prod bucket in
   the role policy.
3. **Data safety:** after refresh, spot-check that emails / tokens / password hashes
   are scrubbed; confirm Stripe keys are test-mode; dry-run an email and prove it
   cannot hit a real customer address allow-list miss.
4. **Release smoke:** deploy a known image to TEST, hit `/health` (assumed; confirm
   path in kickoff), exercise one company job flow and one worker apply flow against
   scrubbed data, confirm MsgHub refresh still works.
5. **Cost sanity:** Cost Explorer filtered by `jb_env=test` (or agreed tags) after
   the first month; watch for a second NAT or forgotten Dev cluster.

No live AWS in this submission. The proof plan is still concrete.

---

## Assumptions (inventory silent or thin)

- Region stays `eu-west-1`.
- Health path `/health` and container port `8080` for the API (same assumption I used
  in Task 2; confirm in kickoff).
- Prod and TEST each own an RDS MySQL; exact instance class can change over time.
- Snapshot + anonymisation is allowed under company policy once an owner signs off.
- TEST may share the ECR registry with Prod; it must not share the DB, secrets, or
  Stripe live keys.
- Infra may still be managed operationally today; this write-up is the target shape,
  not a claim that Terraform already owns Prod.

---

## Layout of this package

```
README.md                 # this write-up
docs/
  architecture.md         # Prod vs TEST diagram + parity table
  data-refresh.md         # chosen strategy, steps, PII, rollback
  runbook-refresh.md      # stretch playbook + smoke / cutover
  testing-approach.md     # prove parity, isolation, safety
  cost.md                 # monthly order-of-magnitude notes
```

Copy this folder into a GitHub repo (e.g. under `task-3/`) and invite the reviewer
when ready. Design files only.
