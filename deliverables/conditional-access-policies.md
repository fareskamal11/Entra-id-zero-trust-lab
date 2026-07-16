# Conditional Access Policy Set

Documented intent for each policy. Build each in **report-only** mode, validate, then enable.

| # | Policy name | Users / target | Condition | Grant / control | State |
|---|---|---|---|---|---|
| CA01 | Require MFA - All Users | All users (excl. break-glass) | Any sign-in | Require MFA | On |
| CA02 | Require MFA - Admins | Directory roles | Any sign-in | Require MFA | On |
| CA03 | Block Legacy Auth | All users | Legacy auth clients | Block | On |
| CA04 | Untrusted Location - Require MFA | All users | Outside trusted IPs | Require MFA | On |
| CA05 | Risky Sign-in - Block *(P2)* | All users | Sign-in risk = High | Block | Report-only → On |
| CA06 | Contractor Restrictions | Jordan.Contractor | Sensitive apps | Require compliant device / block | On |

## Rollout notes

- **Always exclude a break-glass / emergency-access account** from blocking policies so a
  misconfiguration can't lock you out of the tenant.
- Report-only mode surfaces impact in sign-in logs before enforcement — no user disruption.
- Screenshot each policy and an allowed vs. blocked result for the portfolio.
