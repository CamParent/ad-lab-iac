# ad-lab-iac

Ansible automation for hardening a self-built Active Directory lab running on Proxmox. This repo automates a security control I implemented manually first — renaming and disabling the built-in Administrator account across a domain controller, a management jump host, and a domain-joined workstation — and packages it as an idempotent, re-runnable role.

## Environment

Three-tier AD lab built on Proxmox VE:

| Host | Role | OS |
|---|---|---|
| DC01 | Forest root domain controller for `camlab.local` | Windows Server 2025 |
| MGMT01 | Admin jump host (RSAT tools, GPO management) | Windows Server 2025 |
| WS01 | Domain-joined client workstation | Windows 11 |

Supporting controls already in place on the lab itself (not part of this repo, done manually):
- RDP to DC01 restricted to MGMT01 only, enforced via Group Policy (Windows Firewall rules scoped by source IP)
- External NTP sync configured on the PDC emulator
- DNS forwarders configured for internet resolution
- WinRM enabled on all three hosts for Ansible remote management

## What this automates

The `local_admin_hardening` role renames the built-in Administrator account to `svc-legacy-adm` and disables it, using two different mechanisms depending on host type:

- **Domain controllers** — no local SAM database exists on a DC (it's replaced by Active Directory during promotion), so the account is a **domain** object. The role uses `Get-ADUser` / `Rename-ADObject` / `Set-ADUser` / `Disable-ADAccount` to modify it directly in AD.
- **Member servers/workstations** — these have a real local SAM, so the role uses `Rename-LocalUser` and `Disable-LocalUser` against the local account.

Both paths query current state first and only act if it differs from the target state — running the playbook against an already-hardened host produces zero changes.

## Why two code paths

I initially tried to do this entirely through Group Policy (`Accounts: Rename administrator account` / `Accounts: Administrator account status` under Security Options). That worked cleanly on the member servers for the rename, but:

- On the DC, those GPO settings had no effect at all — they only govern local SAM accounts, and a domain controller doesn't have one.
- On the member server, the GPO-driven disable didn't reliably commit even though the setting was correctly configured and applied (confirmed via `gpresult` and `secedit` export) — `Disable-LocalUser` run directly succeeded immediately where the GPO path silently no-op'd.

That's the reasoning baked into this role: it doesn't rely on GPO for either the rename or the disable, since GPO cannot reach the DC's account at all, and cannot be trusted to reliably enforce the disable state even where it applies.

## Structure

```
ad-lab-iac/
├── .github/
│   └── workflows/
│       └── harden.yml         # CI trigger: push to main + manual dispatch
├── inventory/
│   └── hosts.yml              # dc01, mgmt01, ws01 grouped by role
├── group_vars/
│   └── all/
│       └── vars.yml           # local_admin_target_name, domain_name
├── roles/
│   └── local_admin_hardening/
│       └── tasks/
│           ├── main.yml               # routes by host group
│           ├── domain_controller.yml  # AD-based rename/disable
│           └── member_server.yml      # local SAM-based rename/disable
├── playbooks/
│   └── harden_local_admin.yml
└── ansible.cfg
```

## Usage

### Manual run

Requires the `ansible.windows` collection and WinRM configured on target hosts (HTTPS listener, NTLM auth).

```bash
ansible-galaxy collection install ansible.windows

ansible-playbook -i inventory/hosts.yml playbooks/harden_local_admin.yml --ask-pass
```

You'll be prompted for the WinRM connection password for the domain admin account used to run the play.

### Automated run (GitHub Actions)

`.github/workflows/harden.yml` runs the same playbook automatically on every push to `main`, and can also be triggered manually from the Actions tab (`workflow_dispatch`).

It runs on a **self-hosted runner** rather than a GitHub-hosted one, since the target hosts (`192.168.1.x`) sit on a private LAN that GitHub's hosted runners can't reach. The runner is registered on the same host that already runs runners for two other repos (`fortigate-iac`, `iac-foundation`), each as its own independent systemd service.

The WinRM connection password is supplied via a GitHub Actions repository secret (`WINRM_PASSWORD`) and passed to Ansible with `--extra-vars` at runtime — not through Ansible Vault, for the reason described below.

## Notes on credential handling

This repo intentionally does not commit any vaulted or plaintext credentials.

For manual runs, the connection password is supplied interactively via `--ask-pass`. An earlier iteration attempted to store the connection password as an Ansible Vault-encrypted variable, but hit a reproducible issue on the Ansible version used at the time, where vault-encrypted values assigned directly to connection variables (`ansible_password`) weren't reliably decrypted before the WinRM connection plugin consumed them, while the same vaulted value worked fine as a regular templated variable. Prompting at runtime sidesteps this and keeps no credential material on disk at all.

For the automated GitHub Actions run, which can't respond to an interactive prompt, the password is instead stored as a GitHub repository secret and injected via `--extra-vars` at runtime — never committed to the repo, never written to disk on the runner outside of process memory for the duration of the job.

## Verification performed

- `ansible.windows.win_ping` connectivity confirmed against all three hosts prior to running the hardening role
- First run confirmed correct rename/disable behavior against hosts in a known, un-hardened state (validated manually beforehand)
- Second run against already-hardened hosts confirmed `changed=0` across all three hosts, validating idempotency
- `whoami` / `Get-ADUser -Properties MemberOf` used to confirm the replacement domain admin account (`cparent-adm`) had working Domain Admin rights before any built-in Administrator account was disabled, to avoid a lockout scenario
- End-to-end CI pipeline confirmed working: a push to `main` triggered the self-hosted runner, which checked out the repo and executed the playbook successfully against all three live hosts over WinRM
