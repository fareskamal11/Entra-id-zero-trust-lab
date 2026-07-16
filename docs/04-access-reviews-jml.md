# 04 · Access Reviews & Joiner-Mover-Leaver

**Status:** ✅ Completed
**Zero Trust pillar:** Continuous verification / governance
**License required:** Microsoft Entra ID P2 / Entra ID Governance (Access Reviews)

---

## Objective

Granting access correctly once is not security — access has to be **governed over time**.
This phase proves two things: that stale or excessive access gets caught and removed
(**access reviews**), and that identity is controlled across its entire life in the
organisation (**Joiner-Mover-Leaver**).

> **The problem being solved:** access accumulates silently. People change teams, projects
> end, contractors linger. Nobody remembers to take it away. That's **privilege creep**, and
> it's how a standard user quietly ends up able to see everything.

---

## Part 1 — Access Review

### Configuration

`Identity Governance → Access reviews → New access review` (Resource review template)

| Setting | Value | Why |
|---|---|---|
| Review type | Resource review → **Teams + Groups** | Reviewing group membership, not an entitlement catalog |
| Resource | **SG-PrivilegedAdmins** | Reviewed the most sensitive group first — where excess access does the most damage |
| Scope | **All users** | Default was "Guest users only", which would have reviewed nobody |
| Reviewer | **lab admin** (explicitly selected) | Chose a named reviewer over self-review — self-attestation is a weak control |
| Recurrence | One time, 3-day duration | Lab scope; production would be recurring quarterly |
| **Auto apply results to resource** | **Enabled** | ⭐ Makes the review an *enforcement* control, not a survey |
| If reviewers don't respond | **No change** | Safe default — "Remove access" would strip everyone on an ignored review |
| Decision helper | **No sign-in within 30 days** | Surfaces stale accounts as recommendations |

> **The setting that matters most is auto-apply.** Plenty of organisations run reviews and
> then never action the results, so the access simply stays. Auto-apply closes the loop
> automatically — a Deny decision actually removes the membership.

### Decision made

Review **AR01 - Privileged Admins Quarterly Review** went `Not started → Active`, then was
actioned in the reviewer portal (`myaccess.microsoft.com`):

| Member | System recommendation | Decision | Reviewed by |
|---|---|---|---|
| Riley SecAdmin | Approve (recent sign-in activity) | **Approved** | lab admin |

**Justification recorded:** Riley requires eligibility for User Administrator to perform
identity support tasks. Access is **PIM-eligible only — not standing** — gated by MFA with a
2-hour activation cap. Approved for continued eligibility.

Note the decision ties directly back to the Phase 3 control design: the access was approved
*because* it is just-in-time rather than permanent. The review and the PIM configuration
reinforce each other.

---

## Part 2 — Joiner-Mover-Leaver lifecycle

Ran `Alex.NewHire` through a complete identity lifecycle.

```
  JOINER              MOVER                      LEAVER
  ────────            ─────                      ──────
  Create account      Add SG-IT                  Block sign-in
  Baseline group      REMOVE SG-Sales  ◀── the   Revoke sessions
  MFA enrolled        Re-check privilege   step  Remove all groups
  No admin roles                          most   Reclaim licenses
                                          orgs
                                          skip
```

### 🟢 Joiner
- Account created with a clear naming convention.
- Added to **baseline group only** (`SG-Sales`) — department access, nothing more.
- MFA enrolment enforced at first sign-in by Conditional Access policy `CA01`.
- **Assigned roles: 0** — no administrative access simply for existing.

### 🟡 Mover — Sales ➜ IT
1. Added `Alex.NewHire` to **SG-IT** (new department access granted).
2. **Removed `Alex.NewHire` from SG-Sales** — confirmed: *"Member successfully removed."*
3. Verified end state: Alex holds **SG-IT only**, with no residual Sales access.

> **Why step 2 is the whole point.** Adding new access is trivial and everyone does it.
> Removing the old access is the step that gets skipped — and it's precisely how someone who
> has moved teams three times ends up with access to everything. Demonstrating the removal is
> what separates real lifecycle management from "clicking add".

### 🔴 Leaver
Executed in this order:

| # | Action | Purpose |
|---|---|---|
| 1 | **Block sign-in** (`Edit properties → Settings → uncheck "Account enabled"` → Save) | Stops all **new** authentication |
| 2 | **Revoke sessions** | Invalidates **existing** sessions / refresh tokens — force sign-out everywhere |
| 3 | **Remove all group memberships** (SG-IT) | Strips resource access |
| 4 | **Reclaim licenses** | Returns the license to the pool (Alex had 0 assigned) |

**End state:** Account status `Disabled`, Group memberships `0`, Assigned roles `0`.

---

## Lessons learned

**Order matters in offboarding — and I got it wrong first.**
I initially revoked sessions *before* blocking sign-in. That sequence leaves a gap: revoking
kills the current session, but because the account is still enabled the user can simply
authenticate again. The correct order is **block first, then revoke** — close the door, *then*
clear out anyone already inside. I corrected the sequence and re-revoked.

**"Block sign-in" is labelled "Account enabled".**
The control lives at `Edit properties → Settings` as a positively-worded checkbox, not as a
"block" button. Unchecking it is the block action.

**Blocking alone is not offboarding.**
Disabling an account stops new sign-ins but leaves live sessions valid — sometimes for hours.
"We disabled the account but they still had access" is a real incident pattern. Revoking
sessions is the step that actually terminates access.

**Defaults are not safe defaults.**
The access review wizard defaulted to *"All Microsoft 365 groups with guest users"* and
*"Guest users only"* — a review that would have examined nobody in this tenant. Reading the
scope rather than accepting defaults is the difference between a real control and theatre.

---

## Outcome

- A privileged-group access review executed with a **named reviewer, a documented decision,
  a recorded justification, and auto-apply enforcement**.
- A complete identity lifecycle demonstrated end to end, including the two steps most
  commonly missed in practice: **removing prior access on a role change** and **revoking
  sessions on exit**.

---

## Evidence captured

| Screenshot | Shows |
|---|---|
| `04-review-config.png` | Review scoped to SG-PrivilegedAdmins, All users |
| `04-review-settings-autoapply.png` | Auto-apply enabled, "No change" on non-response |
| `04-review-active.png` | AR01 status `Active` |
| `04-review-decision.png` | Riley **Approved**, reviewed by lab admin |
| `04-jml-joiner-sg-sales.png` | Alex in SG-Sales only, 0 roles |
| `04-jml-mover-removed-sg-sales.png` | *"Member successfully removed"* — Alex out of SG-Sales |
| `04-jml-leaver-before.png` | Account status `Disabled`, group memberships `0` |

---

## What I'd do next (production hardening)

- **Recurring quarterly reviews** rather than one-time, with reminders to reviewers.
- Reviewer should be the **group owner or the user's manager**, not a central admin — the
  person with business context makes better decisions than the person with portal access.
- **Lifecycle Workflows** to automate joiner/mover/leaver tasks on HR-driven triggers instead
  of manual clicks.
- **Access packages** (entitlement management) for contractors, with hard expiry dates so
  access ends automatically rather than relying on someone remembering.
- Review **PIM eligible assignments** on a schedule, not just group membership.
- Alert on offboarding gaps — e.g. accounts disabled but still holding group membership.

---

## Interview talking points

- **"How do you stop privilege creep?"** — Periodic access reviews with auto-apply, plus
  enforcing access *removal* at the Mover stage, not just addition.
- **"Walk me through offboarding."** — Block sign-in, then revoke sessions, then strip
  groups, then reclaim licenses — in that order, and here's why the order matters.
- **"Why does revoking sessions matter?"** — Disabling stops new logins; live refresh tokens
  survive. Without revocation an offboarded user can retain access for hours.
- **"Who should review access?"** — Ideally the group owner or manager with business context.
  I used a named admin reviewer and deliberately avoided self-review.
- **"What did you get wrong?"** — I revoked sessions before disabling the account, which
  leaves a re-authentication gap. I caught it, corrected the order, and it's documented.
