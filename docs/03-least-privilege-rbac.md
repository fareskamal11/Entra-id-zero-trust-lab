# 03 · Least Privilege, RBAC & Privileged Identity Management (PIM)

**Status:** ✅ Completed
**Zero Trust pillar:** Least privilege / Assume breach
**License required:** Microsoft Entra ID P2 (PIM)

---

## Objective

Eliminate **standing administrative access**. Instead of holding admin rights permanently,
a privileged user is made *eligible* for a role and must **activate it just-in-time** —
verified with MFA, backed by a written justification, capped to a short window, and
automatically revoked when that window expires.

> **The principle:** admin rights should be a *possibility*, not a *permission*.

---

## Design decisions

| Decision | What I chose | Why |
|---|---|---|
| Which role to grant | **User Administrator** | A narrowly-scoped role (manage users only) rather than Global Administrator. Granting the least powerful role that still does the job is least privilege in practice. |
| Assignment type | **Eligible**, not Active | Active = standing admin (the thing we're eliminating). Eligible means the role can be activated on demand and expires on its own. |
| Eligibility duration | **Permanently eligible** | Riley's job function is ongoing, so the *eligibility* persists — but each individual **activation** is still short-lived. Eligibility ≠ access. |
| Activation max duration | **2 hours** | Long enough to complete admin work, short enough that forgotten sessions self-close. Reduced from the 8-hour default. |
| Require MFA on activation | **Yes** | Re-verifies identity at the moment privilege is granted, not just at sign-in. |
| Require justification | **Yes** | Creates a written, auditable business reason for every elevation. |
| Require approval | **No** (single-operator lab) | In production I would enable approval to enforce separation of duties. Documented as a known gap rather than pretending otherwise. |

---

## Implementation

### 1. Licensing prerequisite
Assigned an **Entra ID P2** license to `Riley.SecAdmin` (PIM is a P2 feature).

> **Gotcha hit and solved:** the assignment failed with
> *"License assignment cannot be done for user with invalid usage location."*
> **Root cause:** the account had no country set, and Entra will not assign a license
> without a usage location. **Fix:** set **Usage location = Egypt** on the user, then
> re-assign. Worth setting on every test account up front.

### 2. Create the eligible assignment
`Entra admin center → PIM → Microsoft Entra roles → Roles → User Administrator → Add assignments`

| Field | Value |
|---|---|
| Role | User Administrator |
| Scope type | Directory (`Default Directory`) |
| Member | `Riley.SecAdmin@faresalkamalgmail.onmicrosoft.com` |
| Assignment type | **Eligible** |
| Duration | Permanently eligible |

### 3. Configure the activation guardrails
`PIM → Microsoft Entra roles → Settings (Role settings) → User Administrator → Edit`

| Setting | Value |
|---|---|
| Activation maximum duration | **2 hour(s)** |
| On activation, require | **Azure MFA** |
| Require justification on activation | **Yes** |
| Require ticket information | No |
| Require approval to activate | No (see design decisions) |

### 4. Verify from the admin side
On `User Administrator → Assignments`:
- **Eligible assignments** → Riley SecAdmin listed (Direct, Default Directory, Permanent)
- **Active assignments** → *empty*

That empty Active tab is the proof: no standing admin exists in the tenant.

### 5. Demonstrate the just-in-time flow (as the end user)
Signed in as `Riley.SecAdmin` in a private browser session:

```
Sign in (Conditional Access → MFA challenge)
        ↓
PIM → My roles → Microsoft Entra roles
        ↓
User Administrator shows as ELIGIBLE  →  [Activate]
        ↓
Re-verify MFA  +  enter written justification  +  choose duration (≤ 2h)
        ↓
Role state = ACTIVATED, with an automatic End time
        ↓
Access self-expires (or is released early via Deactivate)
```

---

## Outcome

Working just-in-time privileged access:

- Riley holds **no admin rights at rest** — sign-in alone grants nothing.
- Elevation requires a **deliberate request**, **MFA re-verification**, and a **written reason**.
- Access is **time-boxed to 2 hours** and **revokes itself** with no cleanup step.
- Every activation is **logged and auditable** in PIM.

**Security impact:** if Riley's credentials were phished, the attacker would inherit a
standard user account — not a User Administrator. To gain privilege they would additionally
need to pass MFA and leave a justified, timestamped audit record of doing so. The blast
radius of a credential compromise is dramatically reduced.

---

## Evidence captured

| Screenshot | Shows |
|---|---|
| `03-pim-eligible-assignment.png` | Riley under **Eligible assignments**; Active tab empty |
| `03-pim-role-settings.png` | 2-hour cap, Azure MFA required, justification required |
| `03-pim-my-roles-eligible.png` | Riley's own view: role Eligible with an **Activate** action |
| `03-pim-activated.png` | **Active assignments** — state `Activated` with an expiry timestamp |

---

## What I'd do next (production hardening)

- Enable **require approval to activate** with a named approver (separation of duties).
- Assign eligibility via the **`SG-PrivilegedAdmins`** group rather than to the user directly,
  so privilege follows group membership and is reviewable in one place.
- Require **ticket information** on activation to tie every elevation to a work item.
- Add **phishing-resistant MFA** (FIDO2 / passkeys) as the activation authentication strength.
- Configure **PIM alerts** for anomalies (e.g. roles activated outside business hours).
- Run recurring **access reviews** over eligible assignments — see [`04-access-reviews-jml.md`](04-access-reviews-jml.md).

---

## Interview talking points

- **"How did you enforce least privilege?"** — Access assigned through groups; chose the
  narrowest role that did the job (User Administrator over Global Administrator); no
  permanent admin assignments anywhere in the tenant.
- **"How did you handle privileged access?"** — PIM eligible-not-active, with a
  request → MFA → justification → time-boxed → auto-expire lifecycle.
- **"Why 2 hours?"** — Long enough for real admin work, short enough that an unattended
  session closes itself. Cut from the 8-hour default deliberately.
- **"What's the security value?"** — A stolen password no longer equals a stolen tenant.
  Privilege requires a second verified, justified, logged step.
- **"What would you improve?"** — Approval workflow and group-based eligibility; I left
  both out for a single-operator lab and documented them as known gaps.
