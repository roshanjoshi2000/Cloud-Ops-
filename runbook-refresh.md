# Runbook: TEST data refresh (stretch)

Operational playbook for the anonymised snapshot path in `data-refresh.md`.
Safe to paste into a team wiki later. Still design-only here; no live AWS.

## Roles

| Role | Does |
| --- | --- |
| Refresh operator | Runs snapshot / restore / scrub steps |
| App owner | Confirms TEST can be wiped; owns smoke sign-off |
| Security / DBA (if exists) | Approves Prod snapshot copy + scrub script changes |

## Manual path (first versions)

### 0. Freeze window

- [ ] Post in eng channel: “TEST refresh starting; do not deploy to TEST”
- [ ] Confirm no open QA session that needs current TEST data
- [ ] Note current TEST API + MsgHub image digests (rollback)

### 1. Snapshot Prod

- [ ] Identify latest successful Prod automated backup, or create manual snapshot
- [ ] Tag: `jb_purpose=test-refresh`, `jb_operator=…`, `jb_source=prod`
- [ ] Wait until snapshot status is available

### 2. Copy / share into TEST boundary

- [ ] If separate account: copy snapshot with the approved role
- [ ] If same account: still treat restore target as TEST-only subnet / SG
- [ ] Verify snapshot is not shared more widely than TEST

### 3. Restore to TEST RDS

- [ ] Restore into TEST instance (or replace TEST instance from snapshot)
- [ ] Attach **TEST** security groups only
- [ ] Confirm publicly_accessible = false
- [ ] Rotate master password; put connection string in `jobbser/test/api-runtime`
- [ ] Take a “pre-scrub” snapshot only if needed for forensics; do not leave it
      world-readable

### 4. Scrub

- [ ] Run anonymisation script against TEST endpoint from the locked-down runner
- [ ] Script exit 0; save log artifact (no row dumps of real emails)
- [ ] Spot SQL checks:
  - no `companies.email` / `workers.email` matching production domains you care about
  - `oauth_access_tokens` empty or obviously fake
  - `password_resets` empty
- [ ] Drop pre-scrub snapshot if policy requires (prefer keep only scrubbed backups)

### 5. App cutover on TEST

- [ ] Secret contains: TEST DB URL, TEST JWT, Stripe **test** keys, non-prod Firebase,
      `hub.test` URL, SES TEST from-address
- [ ] Force new deployment of Jobbser-Api and JobbserMsgHub on TEST
- [ ] Wait for ALB target health = healthy on both services

### 6. Smoke / cutover checklist (after refresh)

**Isolation**

- [ ] From TEST API task / bastion: cannot connect to Prod RDS endpoint
- [ ] Task role denied on Prod secret ARNs
- [ ] TEST bucket list works; Prod bucket list denied

**Data safety**

- [ ] Login with scrubbed TEST user works (known password) or seed admin works
- [ ] Random sample of 20 company + worker emails are clearly fake
- [ ] Stripe mode is test (config flag or key prefix check)
- [ ] Trigger one SES message to an allow-listed engineer address only

**Product smoke (like-for-like)**

- [ ] `GET` health on API (path assumed `/health`; confirm in kickoff)
- [ ] Company can see job posts (scrubbed)
- [ ] Worker can open a listing and submit an application (or equivalent test user flow)
- [ ] Message or notification path causes MsgHub refresh (one realtime check)
- [ ] Upload a small image to TEST S3 via API; object lands in TEST bucket only

**Deploy path**

- [ ] Re-run the TEST deploy workflow with the same digests (or the release candidate)
- [ ] Confirm task defs still point at TEST secret ARN

### 7. Announce and unfreeze

- [ ] Post “TEST refresh complete” + scrubbed snapshot id + image digests
- [ ] Allow TEST deploys again
- [ ] File retention: delete expired refresh snapshots

## Automated path (when the manual path is boring)

I would wrap the above in a workflow_dispatch GitHub Action (TEST account role via
OIDC), roughly:

1. Input: confirm string `refresh-test`, optional snapshot id.
2. Job steps: copy snapshot → restore → wait for available → run scrub container →
   rotate secret password field → ECS force-deploy → hit health + a few HTTP checks.
3. On failure: auto scale TEST services to 0, notify, leave previous scrubbed
   snapshot id in the failure message for rollback.
4. Never auto-refresh on a schedule without a human confirm string. Prod snapshot
   access deserves a person in the loop.

Terraform / Actions files are Task 2 territory; this runbook is the ops contract
those workflows would call.

## Feature flags / client safety

- API secret includes something like `Environment=Test` (name TBD) so logs and any
  internal banner stay honest.
- Mobile / web TEST builds hard-code TEST DNS. Prod store builds never read TEST DNS
  from remote config by accident.
- Stripe test keys only. A “pay to activate job” click in TEST must not hit live
  Checkout.

## Rollback (short)

1. Desired count 0 on TEST services if serving bad data.
2. Restore previous scrubbed TEST snapshot.
3. Redeploy last known-good digests.
4. Re-smoke isolation + health + one company / one worker path.
5. Root-cause the scrub or snapshot choice offline.

## When not to refresh

- During a Prod incident (people distracted)
- When scrub script is mid-change without a review
- When TEST is the only place holding unique QA evidence you still need (export first)
