# TEST cost notes

Grounded in the inventory “Cost reality” section. Numbers move; ordering matters
more than fake precision.

## What actually burned money

From the inventory snapshot (order-of-magnitude):

1. EC2 / NAT + public IPv4
2. ECS / Fargate
3. ALB hours
4. RDS comparatively small

Dev being always-on was painful enough that it got switched off. A TEST env that
quietly becomes “second Prod” will recreate that pain.

## TEST shape vs bill

| Piece | TEST choice | Cost posture |
| --- | --- | --- |
| ALB | One TEST ALB | Fixed monthly slice; hard to avoid if like-for-like |
| NAT | Prefer single NAT | Saves vs one-per-AZ; SPOF acceptable for TEST |
| Public IPv4 | Avoid extra idle addresses | Inventory calls this out as material |
| Fargate API + MsgHub | Small tier (0.25 / 0.5 class), lower desired count than Prod | Same shape, less replica burn |
| RDS | Own MySQL, single-AZ, ≤ Prod class | Usually not the dominant line |
| S3 | Own bucket + lifecycle | Small vs compute |
| Secrets / logs | Own secret; short log retention | Secondary |
| ECR | Shared registry | Cheap vs always-on compute |

## Order-of-magnitude monthly band

If TEST is left **always-on** at small sizes in eu-west-1, I would expect something
in the **few-hundred USD / month** ballpark, dominated by ALB + NAT/IPv4 + two small
Fargate services. Not “free”, not “full Prod duplicate”.

I am deliberately not inventing a spreadsheet of list prices. A tutor can argue the
exact pounds; the design point is: do not double every Prod knob, and do not run Dev
always-on beside TEST.

## Knobs I would use

1. Lower desired counts than Prod when release traffic is light.
2. Single-AZ RDS unless testing failover that week.
3. Park TEST outside release freezes if the team agrees (ECS desired 0; ALB still
   costs if left up; same lesson as Task 2).
4. Turn Dev off when TEST is the active non-prod.
5. Tags (`jb_env=test`) so Cost Explorer can show the env without archaeology.

## What I refuse to do for “savings”

- Share Prod RDS with TEST
- Point TEST at Prod secrets “temporarily”
- Drop MsgHub forever to save one Fargate service (breaks like-for-like)

Cheap and isolated beats fake-cheap and coupled.
