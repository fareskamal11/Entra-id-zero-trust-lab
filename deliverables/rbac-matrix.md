# RBAC Matrix — Least Privilege Mapping

Access model for the lab tenant `faresalkamalgmail.onmicrosoft.com`.
Access is assigned **to groups, not individuals**; admin roles are **never standing**.

**Legend:** `—` none · `R` read · `RW` read/write · `A` admin · **`JIT`** just-in-time via PIM

---

## Identities

| User | Group(s) | Persona |
|---|---|---|
| `Alex.NewHire` | SG-Sales | Standard employee (joiner/mover/leaver subject) |
| `Sam.Manager` | SG-Sales | Team manager |
| `Jordan.Contractor` | *(none — scoped access only)* | External contractor, limited & time-bound |
| `IT.Helpdesk` | SG-IT, SG-HelpDesk | Support / service desk |
| `Riley.SecAdmin` | SG-PrivilegedAdmins | Security admin — **eligible** for User Administrator |
| `lab admin` | *(excluded from CA policies)* | Break-glass / emergency access |

---

## Access matrix

| Role (group) | Files / Sites | GitHub app | M365 | Devices | Security tools | Directory admin |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| Standard User (`SG-Sales`) | R / RW own | R | R | own only | — | — |
| Manager (`SG-Sales` + mgr) | RW team | R | R | team | — | — |
| Help Desk (`SG-HelpDesk`) | R | — | R | RW (support) | R | password reset only |
| Contractor (`Jordan.Contractor`) | R scoped | — | R scoped | — | — | — |
| Security Admin (`SG-PrivilegedAdmins` / Riley) | R | R | R | R | A | **JIT** — User Administrator via PIM |
| Break-glass (`lab admin`) | A | A | A | A | A | A (standing, emergency only) |

---

## Least-privilege notes

- **No standing admin.** The only privileged assignment in the tenant is Riley's
  **eligible** (not active) User Administrator role, activated via PIM with MFA +
  justification and a 2-hour cap. Verified: PIM **Active assignments** is empty at rest.
- **Narrowest role that works.** Riley gets **User Administrator**, not Global
  Administrator — enough to manage users, nothing more.
- **Group-based assignment.** Membership drives access, so onboarding is one add and
  offboarding is one remove. `SG-PrivilegedAdmins` was created with
  *"Entra roles can be assigned to the group" = Yes* so eligibility can move to the group.
- **Help desk is deliberately capped.** Can reset passwords; **cannot** grant roles or
  change Conditional Access policy — a classic privilege-escalation path closed.
- **Contractors are scoped and read-mostly**, with no group membership granting broad access
  and tighter Conditional Access rules applied.
- **Break-glass is the one exception.** `lab admin` holds standing Global Admin and is
  **excluded from every Conditional Access policy** so a misconfigured policy can never lock
  the tenant out. Trade-off accepted deliberately; in production this account would be
  monitored with alerting on every use.
- **Every `A` and `JIT` row is a review candidate** — covered by access reviews in
  [`../docs/04-access-reviews-jml.md`](../docs/04-access-reviews-jml.md).

---

## Known gaps (production hardening)

| Gap | Production fix |
|---|---|
| Eligibility assigned to the user directly | Assign eligibility to `SG-PrivilegedAdmins` so privilege follows membership |
| No approval workflow on activation | Enable *require approval* with a named approver (separation of duties) |
| Break-glass account unmonitored | Alert on any sign-in by the break-glass account |
| Contractor access has no hard end date | Use entitlement management / access packages with expiry |
