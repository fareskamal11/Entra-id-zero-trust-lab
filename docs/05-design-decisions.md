# 05 · Design Decisions & Security Rationale

The other documents describe *what* was built. This one explains *why* — the reasoning,
the trade-offs, and the things I would do differently at production scale.

---

## The thesis

Identity is the control plane. If an attacker holds a valid identity, network controls,
endpoint controls, and encryption largely stop mattering — the attacker is simply a user.
So this lab is built around one question:

> **If an attacker steals a valid password from this tenant, how much do they actually get?**

Every control below exists to shrink that answer.

| Attack step | Control that interrupts it | Phase |
|---|---|---|
| Use stolen password | MFA required (`CA01`) | 2 |
| Bypass MFA via old protocols | Legacy auth blocked (`CA03`) | 2 |
| Sign in from anywhere | Location-aware MFA (`CA04`) | 2 |
| Use credentials leaked/sold online | Risk-based block (`CA05`) | 2 |
| Land on an account that *is* an admin | No standing admin — PIM eligible only | 3 |
| Keep access indefinitely | 2-hour activation cap, auto-expiry | 3 |
| Exploit access nobody remembers | Access reviews + JML removal | 4 |

**Net result:** stealing a password in this tenant yields a standard user account with no
administrative rights, subject to MFA, and reviewed periodically. The credential alone is
close to worthless.

---

## Decision log

### 1. Report-only before enforcement
**Chose:** every Conditional Access policy created in **report-only**, then enabled after
review.
**Why:** a Conditional Access policy is evaluated on *every* sign-in. A misconfigured policy
doesn't degrade access, it removes it — for everyone, instantly. Report-only logs what
*would* have happened with zero user impact, so the blast radius of a mistake is a log entry
rather than an outage.
**Trade-off:** slower to roll out. Correct trade in every production tenant I can imagine.

### 2. Break-glass exclusion on every policy
**Chose:** `lab admin` excluded from all five CA policies.
**Why:** the classic self-inflicted incident is locking every administrator out of the tenant
with the policy meant to protect it — including the administrator who would fix it. One
excluded account guarantees a way back in.
**Trade-off:** deliberately creates a weaker-protected account. Accepted, because
recoverability outranks uniform enforcement here. **Known gap:** in production this account
must be alarmed on *every* sign-in, hardware-token protected, credentials split and sealed.
An unmonitored break-glass account is a backdoor.

### 3. User Administrator, not Global Administrator
**Chose:** made Riley eligible for **User Administrator**.
**Why:** the persona needs to manage users. Global Administrator would also grant that — and
everything else. Least privilege isn't "give a smaller-sounding role," it's "give the least
authority that still completes the task."
**Trade-off:** none meaningful. This is the cheap, obvious win people skip out of convenience.

### 4. Eligible, not Active
**Chose:** PIM eligible assignment with no standing active assignment anywhere in the tenant.
**Why:** standing privilege means the *account* is the security boundary. Just-in-time means
**the activation event** is the boundary — and an activation requires MFA, a written
justification, and produces a log entry. It converts privilege from a property of an identity
into an auditable action.
**Evidence:** the **Active assignments** tab is empty at rest. That empty tab is the control.

### 5. Two hours, cut from the eight-hour default
**Chose:** 2-hour maximum activation.
**Why:** the default assumes a full workday of admin. Real admin tasks are minutes. An
8-hour window mostly means privilege sitting idle and forgotten in a browser tab — which is
standing admin with extra steps. Two hours covers genuine work while forcing re-authentication
for anything longer.
**Trade-off:** occasional re-activation. Cheap.

### 6. Auto-apply on access reviews
**Chose:** enabled **auto apply results to resource**.
**Why:** the common failure isn't running reviews, it's running them and never actioning the
output — the access stays and the organisation now has documentation proving it knew. That's
worse than not reviewing. Auto-apply makes a Deny actually remove the membership.
**Paired with:** "no change" on non-response — an ignored review must not silently strip
everyone's access.

### 7. Named reviewer, not self-review
**Chose:** `lab admin` explicitly assigned as reviewer.
**Why:** self-attestation approaches a rubber stamp. Nobody denies their own access.
**Known gap:** the *right* reviewer is the group owner or the user's manager — they hold the
business context to know whether access is still warranted. A central admin knows the portal,
not the job.

### 8. Skipped the Insights dashboard
**Chose:** verified enforcement via raw **sign-in logs** instead of the Conditional Access
Insights workbook.
**Why:** the workbook is backed by Log Analytics and requires provisioning a workspace,
configuring diagnostic settings, and paying for ingestion. It returned 401 in this tenant
because no workspace existed and the account lacked subscription-level rights. Since the
workbook is an aggregate *view* over data that sign-in logs already expose directly, I used
the source. The raw log naming the exact policy on a specific sign-in is stronger, more
precise evidence than a summary chart.
**Trade-off:** no trend dashboards. Irrelevant at lab scale, deliberate on cost grounds.

---

## Mapping to Zero Trust principles

**Verify explicitly** — MFA on every sign-in (`CA01`), stricter on admins (`CA02`), evaluated
against context: role, location (`CA04`), client type (`CA03`), and risk (`CA05`). No implicit
trust from being "inside" anything.

**Use least privilege** — access assigned through groups rather than to individuals; narrowest
sufficient role; no standing admin in the tenant; help desk can reset passwords but cannot
grant roles or edit policy — a deliberate closure of a classic escalation path.

**Assume breach** — the design presumes a credential *will* be stolen. Hence PIM (a stolen
account isn't an admin account), risk-based blocking (`CA05`), session revocation on exit,
and periodic review to remove access nobody remembers granting.

---

## Mistakes made, and what changed

**Revoked sessions before disabling the account.**
During offboarding I revoked Alex's sessions first, then blocked sign-in. That order leaves a
gap: revocation kills the live session, but the account is still enabled, so the user can
simply authenticate again. Correct order is **block, then revoke** — close the door, then
clear the room. I corrected it and re-revoked.
**What changed:** I now think about offboarding as a *sequence with a race condition*, not a
checklist of independent tasks. Order is part of the control.

**Assumed the license was the blocker.**
P2 assignment failed with *"invalid usage location."* My instinct was a licensing or
entitlement problem. The actual cause was an empty attribute on the user object — Entra won't
assign a license without a usage location, for tax/regional-compliance reasons.
**What changed:** I read the error literally before theorising. The message said *usage
location*; it meant usage location.

**Nearly accepted the wizard defaults.**
The access review wizard defaulted to *"All Microsoft 365 groups with guest users"* +
*"Guest users only."* This tenant has no guests — that review would have run, completed, and
examined **nobody**, producing a clean audit artifact that proved nothing.
**What changed:** I read scope before clicking Next. A control that runs but inspects nothing
is worse than no control, because it manufactures false assurance.

**Built a location policy with no trusted location.**
`CA04` initially applied to "any location" because the named location didn't exist yet, so the
exclusion silently had nothing to reference.
**What changed:** I verify a policy's *effective* scope after saving, rather than trusting
that what I intended is what got configured.

---

## Known gaps (honest limitations)

| Gap | Risk | Production fix |
|---|---|---|
| Break-glass account unmonitored | Backdoor if compromised | Alert on every sign-in; hardware token; sealed split credentials |
| PIM eligibility assigned to the user, not `SG-PrivilegedAdmins` | Privilege doesn't follow group membership | Assign eligibility to the role-assignable group |
| No approval workflow on activation | No separation of duties — Riley self-elevates | Require approval with a named approver |
| Password-based MFA (Authenticator) | Phishable | FIDO2 / passkeys via authentication strengths |
| No device compliance signal | Access allowed from unmanaged devices | Intune compliance + require compliant device |
| One-time access review | Point-in-time, not continuous | Recurring quarterly reviews |
| Manual JML | Human error, delay on terminations | Lifecycle Workflows on HR triggers |
| No SIEM integration | Detection depends on manual log review | Stream sign-in/audit logs to a SIEM |
| Contractor access has no end date | Lingers after engagement ends | Access packages with hard expiry |

Listing these is the point. A lab that claims no weaknesses hasn't been examined honestly.

---

## What I'd build next

1. **Phishing-resistant MFA** for admins — passkeys via authentication strengths. Authenticator
   push is phishable; a real-time proxy defeats it.
2. **Device compliance** — identity alone is half the signal; "known user on unknown device"
   should not be treated as trusted.
3. **Lifecycle Workflows** — remove the human from joiner/mover/leaver. Most offboarding
   failures are timing failures, and automation fixes timing.
4. **Log streaming to a SIEM** with detections for break-glass use, PIM activation outside
   business hours, and repeated CA blocks against one account.
5. **Access packages** for contractors — time-bound by construction rather than by memory.

---

## Interview talking points

- **"Walk me through your Zero Trust design."** Three pillars: verify explicitly (MFA +
  context), least privilege (groups, narrowest role, no standing admin), assume breach (PIM,
  risk blocking, session revocation, reviews). Then the attack-path table above.
- **"How did you avoid locking yourself out?"** Report-only first, break-glass excluded from
  every policy, and I verified effective scope after saving.
- **"Why is PIM more than a checkbox?"** It moves the security boundary from the account to
  the activation event — MFA'd, justified, time-boxed, logged.
- **"What's the weakest part of this build?"** The break-glass account. It's standing Global
  Admin excluded from every policy — necessary for recoverability, and a backdoor if
  unmonitored. In production it gets alerting and a hardware token.
- **"What did you get wrong?"** Offboarding order — I revoked sessions before disabling the
  account, which leaves a re-authentication gap. Caught it, corrected it, documented it.
- **"What would you do with more budget?"** Phishing-resistant MFA, device compliance, and
  SIEM integration — in that order.
