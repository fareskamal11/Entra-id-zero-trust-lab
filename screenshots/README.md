# Evidence Screenshots

Captured live during the build of this lab. All accounts are test identities in a
disposable tenant — no production or real user data.

## Phase 0 — Lab setup
| File | Shows |
|---|---|
| `00-p2-trial-checkout.png` | Entra ID P2 trial, 1-month term, USD 0.00 |
| `00-p2-trial-activated.png` | Trial activation confirmed |
| `00-p2-license-100-available.png` | P2 licensed: 100 total / 100 available |
| `00-lab-admin-licensed.png` | Break-glass admin licensed with Entra ID P2 |

## Phase 1 — Identity foundation
| File | Shows |
|---|---|
| `01-users-list.png` | 6 test identities created, no roles assigned |
| `01-groups-list.png` | 4 security groups incl. role-assignable `SG-PrivilegedAdmins` |
| `01-app-github.png` | Enterprise application registered as a policy target |
| `01-tenant-overview.png` | Tenant summary: P2 license, 7 users, 4 groups, 1 app |

## Phase 2 — MFA + Conditional Access
| File | Shows |
|---|---|
| `02-ca01-require-mfa-all-users.png` | CA01 — MFA for all users (report-only, admin excluded) |
| `02-ca01-breakglass-exclusion.png` | Break-glass account on the Exclude tab — lockout protection |
| `02-ca02-require-mfa-admins.png` | CA02 — MFA enforced on 3 directory roles |
| `02-ca03-block-legacy-auth.png` | CA03 — Block legacy auth (ActiveSync + other clients) |
| `02-ca04-untrusted-location-config.png` | CA04 — "Any location" include, trusted location excluded |
| `02-ca04-untrusted-location.png` | CA04 — policy summary |
| `02-ca05-risky-signin-block.png` | CA05 — Block on sign-in risk High + Medium (P2) |
| **`02-signin-logs-mfa-enforced.png`** | ⭐ Live proof: sign-in **Interrupted (50055)** → MFA → **Success** |

## Phase 3 — Least privilege + PIM
| File | Shows |
|---|---|
| `03-license-error-usage-location.png` | The "invalid usage location" failure — root-caused and fixed |
| `03-riley-licensed-p2.png` | P2 assigned after setting usage location |
| `03-pim-assignment-type-eligible.png` | Assignment type set to **Eligible**, not Active |
| `03-pim-role-settings-default-8h.png` | Default activation window (8 hours) — before hardening |
| `03-pim-role-settings.png` | Hardened: **2h cap, Azure MFA, justification required** |
| **`03-pim-eligible-assignment.png`** | ⭐ Riley under **Eligible assignments** — Active tab empty = no standing admin |
| `03-pim-my-roles-eligible.png` | End-user view: role eligible, requires **Activate** |
| **`03-pim-activated.png`** | ⭐ Role **Activated** with an automatic expiry timestamp |

## Phase 4 — Access reviews + JML
| File | Shows |
|---|---|
| `04-review-config.png` | Review scoped to `SG-PrivilegedAdmins`, All users |
| `04-review-reviewers.png` | Named reviewer assigned (not self-review) |
| `04-review-settings-autoapply.png` | **Auto-apply results** on; "No change" on non-response |
| `04-review-active.png` | AR01 status `Active` |
| **`04-review-decision.png`** | ⭐ Riley **Approved**, reviewed by lab admin, with recommendation |
| `04-jml-joiner-sg-sales.png` | Joiner: baseline group only |
| `04-jml-mover-added-sg-it.png` | Mover: new department access granted |
| **`04-jml-mover-removed-sg-sales.png`** | ⭐ Mover: *"Member successfully removed"* — old access revoked |
| `04-jml-leaver-before.png` | Leaver: pre-offboarding state (Enabled, 1 group) |

⭐ = headline evidence
