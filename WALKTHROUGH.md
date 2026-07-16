# Zero → Hero Walkthrough

The complete build, in order. Each phase tells you **what you're doing**, the **exact clicks**,
**why it matters** (Zero Trust reasoning), and the **evidence** to screenshot for your portfolio.

All work happens in the **Microsoft Entra admin center** → https://entra.microsoft.com

> Portal menus move around. If a path below is slightly different, the feature name is what
> matters — search the top bar for it.

---

## Phase 0 — Get your lab (tenant + licenses)

**What you're doing:** creating a disposable identity environment you fully control, and
turning on the premium features the project needs.

1. **Create a free Azure account** at `azure.microsoft.com/free`. A personal email
   (outlook/gmail) is fine. You'll need a phone number and a **real credit/debit card** for
   identity verification (prepaid/virtual cards are often rejected). This is free — it just
   verifies you. Signing up automatically creates your first Entra ID tenant with a default
   domain `<something>.onmicrosoft.com` (Free tier).

2. **Create a `.onmicrosoft.com` Global Admin — this is the key workaround.** Sign in to
   `entra.microsoft.com`, then Entra ID → **Users** → **+ New user** → **Create new user**.
   Give it a name like `admin@<yourtenant>.onmicrosoft.com` and a password you'll remember.
   Then Entra ID → **Roles & admins** → **Global Administrator** → **Add assignments** → add
   this new user.
   > Why: activating the P2 trial while signed in with your *personal* Azure email fails —
   > Microsoft demands a "work or school" account. A `.onmicrosoft.com` user **is** a
   > work/school account, so activation works when you're signed in as this user.

3. **Activate the P2 trial as the new admin.** Sign out, sign back in to `entra.microsoft.com`
   as `admin@<yourtenant>.onmicrosoft.com`. Go to **Billing → Licenses → Overview** →
   top-right **Quick tasks → Get a free trial** → select **Microsoft Entra ID P2** →
   **Activate**. The trial is **31 days, up to 100 licenses**.

4. **Assign the P2 license.** Billing → Licenses → **All products** → Microsoft Entra ID P2 →
   **Assign** → assign to your admin user now, and to your test users as you create them in
   Phase 1. (Wait a few minutes after activating before assigning.)

5. If **Access Reviews** asks for a license later, start the **Entra ID Governance** trial the
   same way (Get a free trial → Entra ID Governance).

**Why it matters:** the whole project depends on P1/P2 features. MFA is Free, but
**Conditional Access needs P1**, and **PIM, Identity Protection (risky sign-ins), and Access
Reviews need P2 / Governance**. Activating the P2 trial unlocks them in one shot — so plan to
do the hands-on phases in a focused sprint while the clock runs.

**Capture:** screenshot of your tenant Overview showing the P2/trial license active.

---

## Phase 1 — Build the identity foundation

**What you're doing:** creating the users, groups, and an app that everything else targets.

### 1a. Create users
- Entra ID → **Users** → **All users** → **+ New user** → **Create new user**.
- Create a realistic set with clean names:
  `Alex.NewHire`, `Sam.Manager`, `Jordan.Contractor`, `IT.Helpdesk`,
  `Riley.SecAdmin`, `Global.Admin`.

### 1b. Create groups
- Entra ID → **Groups** → **All groups** → **+ New group** → Type = **Security**.
- Make department/access groups: `SG-Sales`, `SG-IT`, `SG-HelpDesk`, `SG-PrivilegedAdmins`.
- For `SG-PrivilegedAdmins`, set **"Microsoft Entra roles can be assigned to the group" = Yes**
  at creation (you can't change this later — it's needed if you assign roles to the group).
- Add the matching users as members.

### 1c. Add an app to protect
- Entra ID → **Enterprise applications** → **+ New application** → pick one from the gallery
  (or just target Microsoft 365 in your policies).

**Why it matters:** you assign access to **groups, not individuals** — that single habit is
what makes least privilege scalable, consistent, and reviewable. Clean names make your
screenshots look like a real environment, not a sandbox.

**Capture:** users list, groups list, the enterprise app.

---

## Phase 2 — MFA + Conditional Access

**What you're doing:** forcing strong, context-aware authentication. This is the "verify
explicitly" pillar of Zero Trust.

Path: Entra ID → **Conditional Access** → **Policies** → **New policy**
(it may appear under **Protection → Conditional Access**). Each policy = Assignments (who /
what / conditions) + Access controls (grant / block / session).

> **Golden rule:** always **exclude a break-glass / emergency account** from blocking
> policies, and build every policy in **Report-only** first, then flip to **On** after checking
> Sign-in logs. This is exactly how it's done in production.

Create these:

| Policy | Assignments | Control |
|---|---|---|
| `CA01 - Require MFA - All Users` | Users: All (exclude break-glass) · Resources: All | Grant → Require MFA |
| `CA02 - Require MFA - Admins` | Users: **Directory roles** (Global/Security Admin) | Grant → Require MFA |
| `CA03 - Block Legacy Auth` | Users: All · Conditions → Client apps → legacy clients | Grant → **Block** |
| `CA04 - Untrusted Location - MFA` | Network: any, exclude trusted IPs* | Grant → Require MFA |
| `CA05 - Risky Sign-in - Block` *(P2)* | Conditions → Sign-in risk = **High** | Grant → Block |

\*For CA04, first define your trusted IP under **Conditional Access → Named locations** and
mark it *Trusted*.

Then have each user register MFA (Microsoft Authenticator) at next sign-in.

**Why it matters:** a stolen password alone should never be enough. MFA + context signals
(role, location, device, risk) mean access decisions adapt to the situation. **No MFA =
unnecessary risk.** Blocking legacy auth closes the biggest MFA-bypass hole.

**Capture:** the policy list, one **allowed** MFA sign-in, one **blocked** attempt.

---

## Phase 3 — Least privilege, RBAC + PIM

**What you're doing:** giving each role only what it needs, and making admin access
**just-in-time** instead of standing.

### 3a. Build the RBAC matrix
- Map roles × resources (see `deliverables/rbac-matrix.md`). Assign access via groups.
- Rule of thumb: if a role doesn't *need* it to do its job, it doesn't get it.

### 3b. Make admin roles eligible (not active) in PIM
- ID Governance → **Privileged Identity Management** → **Microsoft Entra roles** → **Roles**.
- Pick a role (e.g. **User Administrator** or **Security Administrator**) → **Add assignments**
  → select `Riley.SecAdmin` → **Next** → Assignment type = **Eligible** → **Assign**.

### 3c. Configure the activation guardrails
- Same area → select the role → **Role settings** → **Edit**:
  - **Activation maximum duration**: e.g. 2 hours.
  - **On activation, require MFA**: On.
  - **Require justification**: On.
  - Optionally **Require approval** and set an approver.
  - **Update**.

### 3d. Demonstrate a just-in-time activation
- Sign in as `Riley.SecAdmin` → ID Governance → PIM → **My roles** → **Microsoft Entra roles**
  → **Activate** → enter justification + pass MFA → role is active only for the window.

**Why it matters:** standing Global Admin rights are the single biggest prize for an attacker
("assume breach"). Eligible-not-active means a compromised account isn't automatically a
compromised tenant — every elevation is time-boxed, MFA-gated, justified, and logged.
**Too much access creates real risk.**

**Capture:** the eligible assignment, the role settings, a real activation request.

---

## Phase 4 — Access Reviews + Joiner-Mover-Leaver

**What you're doing:** proving you can govern access over time and run the full identity
lifecycle — the "continuously verify" pillar.

### 4a. Run an access review
- ID Governance → **Access Reviews** → **+ New access review**.
- Review a **resource type → Teams + Groups** → select `SG-PrivilegedAdmins`
  (or the contractor).
- Set **reviewer** (yourself / group owner), a **one-time** schedule, and under settings turn
  on **auto-apply results** and decide what happens if the reviewer doesn't respond → **Create**.
- Act as the reviewer: **Approve** or **Deny** each member with a justification.
- *(If prompted for a license, start the Entra ID Governance trial from Phase 0.)*

### 4b. Run the JML lifecycle end-to-end
- **Joiner** — create `Alex.NewHire`, add to baseline group only, confirm MFA registered,
  confirm **no** admin roles.
- **Mover** — add to a new department group **and remove the old one**; re-check that no
  privileged access lingers. (Removing old access is the step everyone forgets.)
- **Leaver** — Users → `Alex.NewHire` → **Block sign-in**; **Revoke sessions** (force sign-out
  everywhere); remove from all groups; reclaim licenses; write down date + actions.

**Why it matters:** access silently piles up and never gets removed ("privilege creep").
Reviews catch stale access; a clean JML process proves you understand lifecycle governance —
exactly what auditors and hiring managers look for.

**Capture:** the review setup, a review **decision**, and the leaver account **disabled +
sessions revoked**.

---

## Phase 5 — Document + publish

**What you're doing:** turning clicks into a portfolio piece.

1. Fill in `docs/05-design-decisions.md` — the *why* behind each choice. This is what
   separates "I clicked buttons" from "I understand identity security."
2. Complete `deliverables/rbac-matrix.md`, `jml-runbook.md`, and drop every screenshot into
   `screenshots/`.
3. **Redact** tenant IDs, object IDs, and anything real. Test data only.
4. Publish:
   ```bash
   git init
   git add .
   git commit -m "Entra ID Zero Trust IAM lab"
   git branch -M main
   git remote add origin https://github.com/<you>/entra-id-zero-trust-lab.git
   git push -u origin main
   ```
5. On GitHub: add repo **topics** (`entra-id`, `zero-trust`, `iam`, `cybersecurity`), and
   **pin** the repo to your profile.

**Why it matters:** the repo is the deliverable. Screenshots prove you did it; the write-up
proves you understood it. Together they demonstrate IAM, Zero Trust, least privilege, PIM,
and governance — the exact skill set for IAM / GRC / SOC / cloud-security roles.

---

## The one-line summary of the whole thing

> You built identities, forced them to prove who they are in context (MFA + Conditional
> Access), gave each only what it needs with no standing admin (RBAC + PIM), and governed
> that access over its whole life (Access Reviews + JML) — then documented *why*. That's
> Zero Trust, end to end.
