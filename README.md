# ad-lab-iac

Part of a broader infrastructure-as-code portfolio — see also [`fortigate-iac`](https://github.com/CamParent/fortigate-iac) (network security automation) and [`iac-foundation`](https://github.com/CamParent/iac-foundation) (Azure landing zone).

[![Harden AD Lab](https://github.com/CamParent/ad-lab-iac/actions/workflows/harden.yml/badge.svg)](https://github.com/CamParent/ad-lab-iac/actions/workflows/harden.yml)

Ansible automation for hardening a self-built Active Directory lab running on Proxmox, extended into a working hybrid identity build against Microsoft Entra ID. This repo automates security controls implemented manually first — starting with built-in Administrator account lockdown, then extending into Entra Connect Sync, group-scoped Password Hash Sync, and Hybrid Azure AD Join — packaged as idempotent, re-runnable roles.

## Environment

Four-tier lab built on Proxmox VE:

| Host | Role | OS |
|---|---|---|
| DC01 | Forest root domain controller for `camlab.local` | Windows Server 2025 |
| MGMT01 | Admin jump host (RSAT tools, GPO management) | Windows Server 2025 |
| WS01 | Domain-joined client workstation, Hybrid Azure AD joined | Windows 11 |
| SYNC01 | Microsoft Entra Connect Sync server | Windows Server 2025 |

Supporting controls already in place on the lab itself (not part of this repo, done manually):
- RDP to DC01 restricted to MGMT01 only, enforced via Group Policy (Windows Firewall rules scoped by source IP)
- External NTP sync configured on the PDC emulator
- DNS forwarders configured for internet resolution
- WinRM enabled on all hosts for Ansible remote management (NTLM over HTTPS/5986, matching the domain's existing auth pattern)

## Relevance

This models the identity-hardening discipline — privileged account lockdown, delegated (not over-privileged) service accounts, and the split between domain and local account semantics — that carries directly into AD-to-Entra ID hybrid identity migrations and other identity-as-code work.

## What this automates

### Local admin hardening

The `local_admin_hardening` role renames the built-in Administrator account to `svc-legacy-adm` and disables it, using two different mechanisms depending on host type:

- **Domain controllers** — no local SAM database exists on a DC (it's replaced by Active Directory during promotion), so the account is a **domain** object. The role uses `Get-ADUser` / `Rename-ADObject` / `Set-ADUser` / `Disable-ADAccount` to modify it directly in AD.
- **Member servers/workstations** — these have a real local SAM, so the role uses `Rename-LocalUser` and `Disable-LocalUser` against the local account.

Both paths query current state first and only act if it differs from the target state — running the playbook against an already-hardened host produces zero changes.

### Hybrid identity extension (Microsoft Entra ID)

Two roles stage everything Microsoft Entra Connect Sync needs, split by host group since the AD-side and sync-host-side prerequisites are fundamentally different operations:

- **`entra_connect_prereqs`** (runs against `domain_controllers`) — adds a verified UPN suffix to the AD forest, creates a `Groups` OU containing the `EntraSync-Include` security group used for group-based sync scoping, creates a `ServiceAccounts` OU containing `svc-entraconnect` (the delegated AD DS Connector account), and grants it exactly two rights at the domain root — `Replicating Directory Changes` and `Replicating Directory Changes All` — via `dsacls`, checked for existing grant before applying to stay idempotent.
- **`entra_connect_host_hardening`** (runs against `sync_hosts`) — enforces TLS 1.2 and strong .NET crypto on the sync server, and opens outbound HTTPS for Entra ID sync endpoints.

What's deliberately **not** automated: the Entra Connect Sync installation and configuration wizard itself. Microsoft's own Entra Connect FAQ states installation is supported only through the wizard — no unattended/silent install path exists. Everything the wizard needs (UPN suffix, scoping group, delegated account, host hardening) is staged by the roles above; the wizard consumes that state rather than being scripted around.

**Design decisions:**
- **Password Hash Sync over Pass-Through Authentication** — decouples cloud sign-in availability from on-prem uptime, appropriate for a single-host lab with no HA.
- **Group-based sync scoping (`EntraSync-Include`)** — rather than syncing the whole directory, only members of one security group (both user and computer objects) sync to Entra ID. Configured directly in the Entra Connect Sync wizard's "Filter users and devices" step.
- **Delegated connector account, not Domain Admin** — `svc-entraconnect` has exactly the two rights Entra Connect Sync needs, granted explicitly via `dsacls`, not inherited from a privileged group.

## Why two code paths (local admin hardening)

I initially tried to do this entirely through Group Policy (`Accounts: Rename administrator account` / `Accounts: Administrator account status` under Security Options). That worked cleanly on the member servers for the rename, but:

- On the DC, those GPO settings had no effect at all — they only govern local SAM accounts, and a domain controller doesn't have one.
- On the member server, the GPO-driven disable didn't reliably commit even though the setting was correctly configured and applied (confirmed via `gpresult` and `secedit` export) — `Disable-LocalUser` run directly succeeded immediately where the GPO path silently no-op'd.

That's the reasoning baked into this role: it doesn't rely on GPO for either the rename or the disable, since GPO cannot reach the DC's account at all, and cannot be trusted to reliably enforce the disable state even where it applies.

## Structure

```
ad-lab-iac/
├── .github/
│   └── workflows/
│       └── harden.yml                     # CI trigger: push to main + manual dispatch
├── inventory/
│   ├── hosts.yml                          # dc01, mgmt01, ws01, sync01 grouped by role
│   └── group_vars/
│       └── all/
│           ├── vars.yml                   # local_admin_target_name, domain_name
│           └── vault.yml                  # ansible-vault encrypted: svc-entraconnect password
├── roles/
│   ├── local_admin_hardening/
│   │   └── tasks/
│   │       ├── main.yml                   # routes by host group
│   │       ├── domain_controller.yml      # AD-based rename/disable
│   │       └── member_server.yml          # local SAM-based rename/disable
│   ├── entra_connect_prereqs/
│   │   ├── defaults/main.yml              # UPN suffix, group/OU/account names
│   │   └── tasks/main.yml                 # UPN suffix, OUs, scoping group, connector account + perms
│   └── entra_connect_host_hardening/
│       └── tasks/main.yml                 # TLS 1.2, outbound HTTPS firewall on SYNC01
├── playbooks/
│   ├── harden_local_admin.yml
│   └── hybrid-identity.yml                # two plays: domain_controllers, then sync_hosts
├── docs/
│   └── hybrid-identity.md                 # architecture, design decisions, issues encountered
└── ansible.cfg
```

## Usage

### Manual run — local admin hardening

Requires the `ansible.windows` collection and WinRM configured on target hosts (HTTPS listener, NTLM auth).

```bash
ansible-galaxy collection install ansible.windows

ansible-playbook -i inventory/hosts.yml playbooks/harden_local_admin.yml --ask-pass
```

### Manual run — hybrid identity prerequisites

```bash
ansible-galaxy collection install microsoft.ad community.windows

ansible-playbook -i inventory/hosts.yml playbooks/hybrid-identity.yml --ask-pass
```

Both prompt for the WinRM connection password. The `vault_entra_connector_password` variable (used to create `svc-entraconnect`) is supplied automatically via `ansible.cfg`'s `vault_password_file` setting — see [`docs/hybrid-identity.md`](docs/hybrid-identity.md) for the full manual steps that follow (Entra Connect Sync wizard install, sync validation, Hybrid Azure AD Join).

### Automated run (GitHub Actions)

`.github/workflows/harden.yml` runs the local admin hardening playbook automatically on every push to `main`, and can also be triggered manually from the Actions tab (`workflow_dispatch`).

It runs on a **self-hosted runner** rather than a GitHub-hosted one, since the target hosts (`192.168.1.x`) sit on a private LAN that GitHub's hosted runners can't reach. The runner is registered on the same host that already runs runners for two other repos (`fortigate-iac`, `iac-foundation`), each as its own independent systemd service.

The WinRM connection password is supplied via a GitHub Actions repository secret (`WINRM_PASSWORD`) and passed to Ansible with `--extra-vars` at runtime — not through Ansible Vault, for the reason described below.

## Notes on credential handling

This repo intentionally does not commit any plaintext credentials.

For manual runs, the WinRM connection password is supplied interactively via `--ask-pass`. An earlier iteration attempted to store the connection password as an Ansible Vault-encrypted variable, but hit a reproducible issue on the Ansible version used at the time, where vault-encrypted values assigned directly to connection variables (`ansible_password`) weren't reliably decrypted before the WinRM connection plugin consumed them, while the same vaulted value worked fine as a regular templated variable. Prompting at runtime sidesteps this and keeps no credential material on disk at all.

For the automated GitHub Actions run, which can't respond to an interactive prompt, the WinRM password is instead stored as a GitHub repository secret and injected via `--extra-vars` at runtime — never committed to the repo, never written to disk on the runner outside of process memory for the duration of the job.

The `svc-entraconnect` account password is a different category of secret — it's data a task needs (to create the account), not a connection credential — so it's stored in an `ansible-vault`-encrypted `vault.yml`, decrypted automatically via a `vault_password_file` referenced in `ansible.cfg`. The vault password file itself (`~/.ansible_vault_pass`) lives outside the repo on both hosts that run these playbooks (`ansible-controller` and `github-runner`) and is `.gitignore`'d.

**Known limitation:** the vault password file currently has to be manually copied to any new machine that clones this repo and needs to run playbooks against it — it isn't centrally distributed. Worth revisiting with GitHub Actions secrets or a proper secrets manager if this repo grows beyond two control hosts.

## Verification performed

### Local admin hardening
- `ansible.windows.win_ping` connectivity confirmed against all three hosts prior to running the hardening role
- First run confirmed correct rename/disable behavior against hosts in a known, un-hardened state (validated manually beforehand)
- Second run against already-hardened hosts confirmed `changed=0` across all three hosts, validating idempotency
- `whoami` / `Get-ADUser -Properties MemberOf` used to confirm the replacement domain admin account (`cparent-adm`) had working Domain Admin rights before any built-in Administrator account was disabled, to avoid a lockout scenario
- End-to-end CI pipeline confirmed working: a push to `main` triggered the self-hosted runner, which checked out the repo and executed the playbook successfully against all three live hosts over WinRM

### Hybrid identity extension
- `entra_connect_prereqs` and `entra_connect_host_hardening` both confirmed idempotent — re-running against already-configured hosts produces `changed=0` (with one known, explained exception: the service account creation task reports `changed` on every run since Ansible can't compare AD account passwords against existing state)
- Sync validated end-to-end with a real object, not just "sync completed without errors": added `cparent-adm` to `EntraSync-Include`, triggered `Start-ADSyncSyncCycle -PolicyType Delta`, confirmed the account appeared in the Entra admin center with `On-premises sync: Yes`
- Password Hash Sync validated by signing into `myaccount.microsoft.com` as `cparent-adm@teslacyberheartgmail.onmicrosoft.com` using the on-prem AD password — proves the synced hash actually authenticates, not just that the sync job ran
- Hybrid Azure AD Join validated on WS01 via `dsregcmd /status`: `AzureAdJoined: YES`, `TpmProtected: YES`, `DeviceAuthStatus: SUCCESS`

## Issues encountered (hybrid identity build)

Documented in detail in [`docs/hybrid-identity.md`](docs/hybrid-identity.md), briefly:

- **Wrong UPN suffix assumed** — tenant's actual default domain (`teslacyberheartgmail.onmicrosoft.com`) didn't match the intuitive assumption from the sign-in email; caught via the admin center's UPN suffix dropdown, corrected via `Set-ADForest` and the Ansible role default.
- **Personal Microsoft account rejected by Entra Connect Sync** — a live.com-backed MSA with Global Administrator rights still isn't a native tenant identity; Entra Connect Sync requires the latter. Fixed by creating a dedicated cloud-native admin account.
- **Hybrid join failed with `error_missing_device`** — WS01's computer object was added to the scoping group, but no sync cycle had run since, so Entra ID had no device object to register against yet. Fixed by triggering a delta sync before retrying the join.
- **Privileged account in sync scope caused a persistent export failure** — `cparent-adm` (Domain Admins/Enterprise Admins member) was initially included in `EntraSync-Include`. AD's AdminSDHolder/SDProp process disables permission inheritance on protected accounts, silently overriding the domain-root `dsacls` grant given to `svc-entraconnect`. This surfaced as a `ms-DS-ConsistencyGuid` writeback failure (error 8344, "Insufficient access rights") on every export cycle — expected, since `svc-entraconnect` was deliberately granted only read-side replication rights, and no domain-root grant reaches a protected object regardless. Fixed by removing the privileged account from sync scope (correct by design — break-glass/admin accounts shouldn't sync to the cloud) and clearing the resulting orphaned connector-space object via `Remove-ADSyncCSObject` (GUI delete was unavailable for this object state in the installed Entra Connect Sync version).

## Conditional Access validation (Microsoft Entra ID P2)

A first pass at Conditional Access was built and validated manually
against the hybrid identity foundation above — three Report-only
policies (MFA, legacy auth block, device compliance), tested against
real sign-in logs, not just policy configuration. Full writeup in
[`docs/conditional-access.md`](docs/conditional-access.md).

## Roadmap

Hybrid identity build (Entra Connect Sync, group-scoped PHS, Hybrid Azure AD Join) — **done**. Next: validating Conditional Access policy behavior against the hybrid-joined WS01, cross-referenced with the identity-as-code module in [`iac-foundation`](https://github.com/CamParent/iac-foundation).
