# Joiner-Mover-Leaver (JML) Runbook

Operational runbook for identity lifecycle management in
`faresalkamalgmail.onmicrosoft.com`. Executed and verified against `Alex.NewHire`.

---

## 🟢 Joiner — new hire

| # | Action | Portal path |
|---|---|---|
| 1 | Create account using the naming convention `First.Role` | `Entra ID → Users → New user` |
| 2 | Set **usage location** (required before any license assignment) | `User → Properties → Settings` |
| 3 | Add to **baseline department group only** | `Groups → SG-<dept> → Members → Add` |
| 4 | Confirm MFA enrolment is enforced at first sign-in (policy `CA01`) | `Conditional Access → CA01` |
| 5 | Confirm **no admin roles** assigned | `User → Assigned roles` = 0 |

**Principle:** access is granted by group membership, never directly to the user.
**Verification:** user appears in exactly one baseline group; assigned roles = 0.

---

## 🟡 Mover — role or department change

| # | Action | Portal path |
|---|---|---|
| 1 | Add to the **new** department group | `Groups → SG-<new> → Members → Add` |
| 2 | ⭐ **Remove from the previous** department group | `Groups → SG-<old> → Members → select → Remove` |
| 3 | Re-evaluate whether any privileged eligibility is still justified | `PIM → Roles → Assignments` |
| 4 | Log the change: who approved, when, why | Review log below |

**Principle:** a move is an *exchange* of access, not an accumulation.
**Verification:** the user's `Groups` tab lists the new group **and nothing else**.

> ⚠️ **Step 2 is the most-skipped step in real IAM.** Adding is easy and always happens;
> removing is invisible and rarely does. Three role changes without step 2 and a standard
> user quietly holds access to three departments. This is privilege creep.

---

## 🔴 Leaver — departure

**Order is critical. Perform in this exact sequence:**

| # | Action | Portal path | Why this order |
|---|---|---|---|
| 1 | **Block sign-in** — uncheck **Account enabled** | `User → Edit properties → Settings` | Stops **new** authentication first |
| 2 | **Revoke sessions** | `User → Overview → Revoke sessions` | Kills **existing** sessions / refresh tokens |
| 3 | **Remove all group memberships** | `User → Groups → Remove membership` | Strips resource access |
| 4 | **Reclaim licenses** | `User → Licenses` | Returns license to the pool |
| 5 | Handle mailbox / data per policy | — | Retention obligations |
| 6 | Document: date, actions taken, owner | Review log below | Audit trail |

> ⚠️ **Why block *before* revoke.** Revoking first kills the live session, but the account is
> still enabled — the user can simply sign in again. Block the door, *then* clear the room.
> (I made this exact mistake during the lab and corrected it.)

> ⚠️ **Blocking alone is not offboarding.** A disabled account can retain a valid refresh
> token for hours. "We disabled the account but they still had access" is a real incident
> pattern. Step 2 is what actually terminates access.

**Verification:** Account status = `Disabled`, group memberships = `0`, assigned roles = `0`.

---

## Access review checklist

For each member under review, decide against:

- [ ] Who has this access?
- [ ] Do they still need it for their current role?
- [ ] Is the level of access appropriate, or excessive?
- [ ] Any risks, exceptions, or dormancy signals (e.g. no sign-in in 30 days)?
- [ ] Decision **documented with a written justification**.

---

## Execution log

| Account | Phase | Owner | Date | Action / decision | Verified |
|---|---|---|---|---|---|
| Riley.SecAdmin | Access review (AR01) | lab admin | 2026-07-16 | **Approved** — eligibility is PIM-only (not standing), MFA + 2h cap | Decision recorded, auto-apply on |
| Alex.NewHire | Joiner | lab admin | 2026-07-05 | Created; baseline access `SG-Sales`; no roles | Groups = SG-Sales, roles = 0 |
| Alex.NewHire | Mover | lab admin | 2026-07-16 | Sales ➜ IT: added `SG-IT`, **removed `SG-Sales`** | "Member successfully removed" |
| Alex.NewHire | Leaver | lab admin | 2026-07-16 | Sign-in blocked → sessions revoked → `SG-IT` removed | Status `Disabled`, groups = 0 |

---

## Production hardening

| Manual step today | Automated in production |
|---|---|
| Manual group add/remove | **Lifecycle Workflows** triggered by HR attributes |
| One-time access review | **Recurring quarterly** reviews with reviewer reminders |
| Central admin as reviewer | **Group owner / manager** as reviewer (business context) |
| Contractor access removed by hand | **Access packages** with hard expiry dates |
| Offboarding checklist | Automated leaver workflow + alerting on gaps |
