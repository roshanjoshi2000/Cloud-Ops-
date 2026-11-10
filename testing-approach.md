# Testing approach

README has the short version. This is how I would prove the design before anyone
trusts TEST for a release gate.

No live AWS in this submission. The checks below are what I would run against a
real TEST once it exists.

## 1. Like-for-like parity

Walk the inventory list and tick TEST:

| Check | Pass looks like |
| --- | --- |
| Jobbser-Api on Fargate | Service running in TEST cluster; awsvpc; secrets injected |
| JobbserMsgHub on Fargate | Separate service; clients can open realtime via TEST ALB |
| ALB HTTPS | Listener + target groups for API and hub |
| VPC isolation | Tasks private; DB private |
| RDS MySQL | Own endpoint; schema matches app expectations |
| S3 | Own bucket; upload succeeds |
| SES | Sends only under TEST identities / sandbox |
| Secrets Manager | Own secret; task def references it |
| DNS | `api.test` / `hub.test` (or agreed names) resolve to TEST ALB |
| Deploy path | Same CI shape updates TEST services by digest |

If MsgHub is missing, I would not call the env like-for-like yet. Realtime is part
of how the app behaves in Prod.

## 2. Isolation from Prod

Active probes, not just architecture slides:

1. From a shell in a TEST task (or SSM session): `nc` / client attempt to Prod RDS
   port fails.
2. `aws secretsmanager get-secret-value` with the **TEST task role** on a Prod secret
   ARN returns AccessDenied.
3. `s3:ListBucket` / `GetObject` on Prod uploads bucket denied for TEST task role.
4. Security group review: no rule allowing TEST CIDR onto Prod 3306.
5. No VPC peering / routes that make the above accidentally true later.

Document the probe outputs in the refresh ticket the first time; sample them after
big network changes.

## 3. Data safety

After every refresh:

1. SQL sample: emails in `companies` and `workers` match the scrub pattern
   (`@test.jobbser.local` or whatever the script uses).
2. `oauth_access_tokens` and `password_resets` empty or obviously fake.
3. Password hashes are the known TEST hash (or logins only via seed users).
4. Stripe keys in the TEST secret are test-mode.
5. Send one email to an allow-listed engineer; confirm SES configuration set /
   from-address is TEST.
6. Confirm no Prod connection string substring appears in the TEST secret.

Failure on any of these = TEST stays closed.

## 4. Functional release smoke

Minimum path before “TEST verified this build”:

1. Deploy candidate digests to TEST API + MsgHub.
2. ALB health green.
3. Authenticate as scrubbed company; list job posts; open one.
4. Authenticate as scrubbed worker; apply (or equivalent).
5. Confirm MsgHub signals a refresh after a state-changing API call.
6. Upload a tiny asset; object appears only in TEST bucket.
7. Optional: Stripe test Checkout for job activation if that path is in scope for
   the release.

This is not a full mobile regression suite. It is enough to catch “wrong secret”,
“DB empty”, “hub down”, “pointing at Prod”.

## 5. Cost and footprint

1. Tags present: `jb_env=test` (or agreed taxonomy).
2. Cost Explorer: TEST monthly band in the expected order of magnitude (see
   `cost.md`).
3. Confirm Dev is not also always-on “just in case”.
4. Single NAT (or documented exception). No surprise second ALB.

## 6. What I am not claiming in this package

- Live snapshot restore against Jobbser accounts
- A finished anonymisation script checked into Prod
- Load-test parity with Prod traffic
- Perfect Multi-AZ failover testing on the cheap TEST RDS (that is a special run)

## 7. How this maps to the brief

| Brief ask | Where I prove it |
| --- | --- |
| Like-for-like enough | Section 1 + architecture parity table |
| Isolated from Prod | Section 2 |
| Safe to use for release checks | Sections 3-4 |
| Cost awareness | Section 5 + `cost.md` |
