# Conditional Access Validation: Hybrid-Joined Device & Pilot User

This extends `ad-lab-iac`'s hybrid identity work with a first pass at
Conditional Access (CA), validated manually against Microsoft Entra ID
P2 before any policy is codified as IaC. Three policies were built and
tested against `intune-test01` (a dedicated, non-privileged pilot
account) and WS01 (hybrid Azure AD joined).

## Why manual validation before IaC

Given the AdminSDHolder/writeback issue encountered earlier in this
project, policies were validated directly in the Entra ID admin center
first, in Report-only mode, before any consideration of codifying them
via Terraform (`azuread` provider) in `iac-foundation`. Report-only mode
evaluates policies against real sign-ins without enforcing them,
making it safe to validate behavior with zero risk of lockout.

## Policies built

| Policy | Grant Control | Scope | State |
|---|---|---|---|
| `CA001-Pilot-RequireMFA-AllApps` | Require multifactor authentication | `Intune-Pilot` group, All resources | Report-only |
| `CA002-Pilot-BlockLegacyAuth` | Block access | `Intune-Pilot` group, Exchange ActiveSync + Other clients | Report-only |
| `CA003-Pilot-RequireCompliantDevice` | Require device marked as compliant | `Intune-Pilot` group, All resources | Report-only |

All three are scoped to a dedicated `Intune-Pilot` security group rather
than all users — consistent with the least-privilege, scoped-group
pattern used throughout this project (`EntraSync-Include`, delegated
service accounts, etc.).

## Validation results

Verified via **Entra ID admin center → Sign-in logs → Report-only tab**,
not just policy configuration — confirming each policy evaluates real
sign-in events correctly, not just that it exists.

A single interactive sign-in from `intune-test01` produced:

| Policy | Result | Why |
|---|---|---|
| `CA001` | Report-only: Success | MFA was satisfied at sign-in |
| `CA002` | Report-only: Not applied | Modern-auth browser sign-in; policy correctly scoped to legacy clients only, so it doesn't false-positive on this sign-in type |
| `CA003` | Report-only: Failure | WS01 has not completed Intune MDM enrollment (see `hybrid-identity.md` / MDM enrollment status), so it isn't marked compliant — this is the expected, correct result |

## Known limitation

`CA002` (legacy auth block) was validated for correct *configuration*
only — confirming the right conditions, scope, and grant control are
set — not for a live-fire test. Triggering a genuine legacy-auth
sign-in would require deliberately configuring an old-protocol client
(e.g., IMAP/basic auth), which was out of scope for this validation
pass. This is a deliberate, documented gap, not an oversight.

## Relationship to MDM enrollment

`CA003`'s Report-only: Failure result is a useful, expected side-effect
of the MDM enrollment issue tracked separately — it confirms Conditional
Access correctly detects and would block a non-compliant device once
enforced, directly demonstrating why completing MDM enrollment matters
beyond just device inventory/management.

## Next steps

- Resolve the parked MDM enrollment issue, re-test `CA003` for an
  expected `Report-only: Success` once WS01 is compliant
- Codify all three policies as Terraform (`azuread` provider),
  cross-referenced with the identity-as-code module in `iac-foundation`
- Flip `CA001` to enforced (`On`) once confident, given its clean
  Report-only result
