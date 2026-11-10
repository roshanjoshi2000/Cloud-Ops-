# Data refresh strategy

TEST has its **own** RDS MySQL. This file is how I would get data into it and keep
it useful without pretending there is one correct copy method.

## Options I considered

| Approach | What it is | Strengths | Weaknesses |
| --- | --- | --- | --- |
| A. Raw Prod snapshot restore | Copy Prod volume/snapshot straight into TEST | Perfect realism | Live PII in TEST; passwords/tokens present; policy risk |
| B. Anonymised snapshot restore | Snapshot → TEST RDS → scrub scripts before open | Realistic shape + safer to browse | Scrub must stay complete; takes longer; can break referential quirks if clumsy |
| C. Synthetic seed | Generate companies, workers, jobs from fixtures | Clean privacy; repeatable in CI | Misses messy Prod edges; easy to under-test |
| D. Logical subset dump | `mysqldump` filtered tables / rows | Smaller; can exclude hot tables | Easy to leave a PII column; slower to automate well |

## Chosen: B (anonymised snapshot restore)

Primary path for release-check TEST.

Secondary: keep a **small synthetic fixture** (C) for “empty TEST” bootstrap and for
pipeline jobs that should not depend on Prod snapshot access.

I am not claiming B is the only correct method. For Jobbser specifically, the schema
pack shows enough relational weirdness (`job_posts` vs hire-ledger `jobs`,
polymorphic `addresses`, referral tables) that I want Prod-shaped rows more than a
perfectly clean fake dataset. Privacy still wins over convenience, so scrub is
mandatory before anyone uses the env.

## High-level steps

1. **Preflight**
   - Confirm TEST RDS exists and is empty-or-replaceable.
   - Confirm nobody is mid-release on TEST.
   - Note Prod snapshot source (automated backup or manual copy snapshot).
   - Assumption: a break-glass role can share/copy the snapshot into the TEST
     account or VPC; inventory is silent on cross-account setup.

2. **Snapshot**
   - Use a recent Prod automated backup, or take a manual snapshot in a change window.
   - Tag it `jb_purpose=test-refresh`, timestamp, operator.

3. **Restore onto TEST**
   - Restore into the **TEST** instance identifier (or replace the TEST instance from
     the snapshot per the team’s RDS practice).
   - Apply TEST security groups only (API/MsgHub TEST SGs).
   - Rotate the master password to a TEST-only value; store in TEST Secrets Manager.
   - Do **not** open the app to engineers yet.

4. **Anonymise in place (on TEST)**
   - Run scrub SQL / job from a locked-down runner that can reach TEST MySQL only.
   - Scrub columns below; keep ids and foreign keys stable where possible so joins
     still work.
   - Re-hash any password fields to a known TEST bcrypt (or invalidate all logins and
     use a seed admin).
   - Wipe or replace oauth tokens, password reset rows, invite / referral codes that
     could collide with Prod links.
   - Point asset URLs at TEST bucket placeholders or rewrite hostnames.

5. **Re-point app config**
   - Update TEST secret: connection string, JWT (TEST key), Stripe test keys,
     Firebase non-prod, hub URL `hub.test.…`, SES from-address.
   - Force new ECS task deployment so tasks pick up the secret.

6. **Verify**
   - Run the smoke list in `runbook-refresh.md`.
   - Spot-check PII queries (no real emails remaining in `companies` / `workers`).
   - Only then announce TEST ready.

7. **Tidy**
   - Expire old refresh snapshots per retention (kickoff question).
   - Keep one previous scrubbed snapshot for rollback.

## PII / sensitive map (from schema pack)

Treat this as the minimum scrub list. Inventory + schema are sanitized; real DB may
have more columns.

| Table / area | Sensitive fields | Scrub idea |
| --- | --- | --- |
| `companies` | email, password, name (optional), website | Fake emails `company-{id}@test.jobbser.local`; known TEST hash; optional name prefix `TEST ` |
| `workers` | email, password, first/last name | Same pattern; keep id stability |
| `addresses` | street, email, lat/long if sensitive | Replace streets; jitter or null coords if needed |
| `applications` | email | Derive from scrubbed worker or null |
| `connections` | user_email | Match scrubbed worker email |
| `oauth_access_tokens` | tokens | Truncate table or replace with invalid values |
| `password_resets` | tokens / emails | Truncate |
| `invite_links` / `referral_links` | codes | Rotate to TEST-only codes |
| `devices` | device tokens | Truncate or fake |
| `messages` | body | Optional redaction if messages may hold PII |
| `notifications` | payload fields | Redact if free text |
| Secrets / Stripe | n/a in DB | Handled in Secrets Manager, not SQL |

Hashed passwords are still sensitive. I would not leave Prod hashes in TEST.

## What I would not scrub away

- Ids and foreign keys (unless a column is itself a secret)
- Job post structure, application statuses, hire-ledger rows
- Metric counters (usually fine)
- Flags like `job_posts.paid` / `active` (useful for entitlement-adjacent tests)

If a future subscription schema lands (Task 1 territory), scrub billing emails /
customer ids the same way.

## S3 / assets

Full Prod bucket copy is usually unnecessary and can leak profile images.

Pragmatic approach:

- Empty TEST bucket + a small set of placeholder logos, **or**
- Copy a limited prefix after confirming objects are non-sensitive, and rewrite DB
  keys to match.

I would default to placeholders unless QA specifically needs real image bytes.

## Rollback

If a refresh is bad (scrub failed, app broken, wrong snapshot):

1. Stop announcing TEST; scale TEST ECS desired count to 0 if needed.
2. Restore the **previous scrubbed** TEST snapshot (kept on purpose).
3. Re-deploy previous known-good task digests if the refresh was paired with a bad
   image.
4. Re-run smoke.
5. Only then investigate the failed scrub in a throwaway instance, not in the live
   TEST endpoint.

Do not “fix forward” by copying Prod again without scrub. That recreates the PII
problem.

## Cadence

- **Before a major release** or schema migration rehearsal: full anonymised refresh.
- **Weekly or on-demand** if TEST drifts too far (product call).
- **Synthetic only** when Prod snapshot access is blocked or for PR-level API tests.

## Trade-off summary (what I would say to a tutor)

I picked anonymised restore because Jobbser’s value in TEST is “does this behave on
data that looks like ours?” Synthetic is safer and faster, and I still want it for
bootstrap. Raw restore is the wrong default once you have seen the schema pack’s
auth and profile tables. The cost of B is discipline: scrub scripts must be owned,
tested, and updated when new columns land.
