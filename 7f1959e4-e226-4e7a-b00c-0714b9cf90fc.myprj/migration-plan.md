# MIGRATION FROM POWERSHELL TO ANSIBLE

---

## 1. Executive Summary

### Scope
This repository contains Windows infrastructure-as-code automation scripts and Group Policy Object (GPO) backup artifacts targeting a Microsoft Active Directory domain environment (`temp.local`). The repository is organized into two functional modules:

1. **`host-guardian-service/`** — A three-phase PowerShell installation sequence for deploying a Windows Host Guardian Service (HGS) cluster node, including WinRM enablement, Windows Feature installation, certificate provisioning, and HGS server initialization.
2. **`group-policy-baseline/`** — A GPO bulk-import script and seven GPO backup sets representing the 2019 Microsoft Security Baseline, covering Domain Controllers, Member Servers, Workstations, Credential Guard, Virtualization-Based Security, Defender Antivirus, and core domain policies.

### Complexity
**High.** The migration involves:
- Windows-only workloads with no Linux/RHEL equivalents for several components (HGS is a Windows Server role with no direct Ansible module; GPO management requires `community.windows` or `ansible.windows` collections).
- Interactive prompts (`Read-Host`) that must be replaced with Ansible variables and Vault-encrypted secrets.
- Binary/UTF-16 encoded GPO artifacts (`registry.pol`, `GptTmpl.inf`, `gpreport.xml`) that cannot be read as plain text and require special handling.
- A multi-step, reboot-dependent HGS installation sequence that must be orchestrated across Ansible plays with `reboot` handlers.
- Certificate (PFX) and DSRM password secrets currently passed interactively at runtime.

### Timeline Estimate
| Phase | Scope | Estimated Effort |
|---|---|---|
| Phase 1 | Environment setup, inventory, Vault scaffolding | 1 week |
| Phase 2 | HGS role migration (Steps 1–3 + reboot handling) | 2 weeks |
| Phase 3 | GPO baseline role migration (import automation) | 2 weeks |
| Phase 4 | Testing, validation, documentation | 1 week |
| **Total** | | **~6 weeks** |

---

## 2. Module Migration Plan

### 2.1 MODULE INVENTORY

---

#### MODULE 1: Host Guardian Service Installer
- **Description:** A sequential three-script PowerShell workflow that provisions a Windows Server node as an HGS cluster member. The workflow is split across mandatory reboots: Step One installs prerequisites and renames the machine; Step Two initializes the HGS domain and AD forest; Step Three configures the HGS service with TPM attestation mode and dual-purpose signing/encryption certificates.
- **Path:** `host-guardian-service/`
- **Technology:** PowerShell (imperative, interactive — no DSC, no `Param()` blocks, no module manifest)
- **Key Features:**
  - **Step One (`Install-HGS-StepOne.ps1`):**
    - Enables PSRemoting and configures WinRM via `winrm quickconfig -force` and `Enable-PSRemoting -force`
    - Installs a Root CA certificate from a relative path (`.\Root.pem`) into the Enterprise/User root store using `certutil -addstore -f -enterprise -user root`
    - Installs two Windows Features: `HostGuardianServiceRole` (with management tools) and `DNS` (with management tools)
    - Prompts operator for a new computer name via `Read-Host` and calls `Rename-Computer`
    - Triggers `Restart-Computer` — **hard reboot dependency before Step Two**
  - **Step Two (`Install-HGS-StepTwo.ps1`):**
    - Imports `HgsServer` PowerShell module
    - Prompts operator for DSRM (Directory Services Restore Mode) password via `Read-Host -AsSecureString`
    - Calls `Install-HGSServer` with hardcoded HGS domain name `cool-name.net`, the DSRM password, and `-Restart` flag — **second hard reboot dependency before Step Three**
  - **Step Three (`Install-HGS-StepThree.ps1`):**
    - Imports `HgsServer` PowerShell module
    - Prompts operator for PFX certificate password via `Read-Host -AsSecureString`
    - Calls `Initialize-HgsServer` with:
      - Log directory: `C:\Admin`
      - HGS service name: `cool-name-here` (hardcoded placeholder)
      - Protocol: `-Http` (not HTTPS — security concern)
      - Attestation mode: `-TrustTpm`
      - Signing certificate: `C:\Admin\host-guardian-service\HGS-Certificate.pfx` (hardcoded path)
      - Encryption certificate: same PFX file reused for both signing and encryption roles
    - Runs `Get-HGSTrace -RunDiagnostics` as a post-install validation step

---

#### MODULE 2: Group Policy Baseline Importer
- **Description:** A PowerShell script that bulk-imports seven GPO backups from a user-selected folder into an Active Directory domain. The GPO backups represent the Microsoft Security Baseline 2019 with local modifications, applied to `temp.local`. The script reads each backup's `gpreport.xml` to resolve the GPO display name and calls `Import-GPO` with `-CreateIfNeeded`.
- **Path:** `group-policy-baseline/`
- **Technology:** PowerShell (imperative); GPO backup artifacts in Microsoft Group Policy Backup Schema v2.0 format
- **Key Features:**
  - **Import Script (`ImportGPOBulk.ps1`):**
    - Imports `ActiveDirectory` and `GroupPolicy` PowerShell modules
    - Uses a COM `Shell.Application` object to open an interactive folder browser dialog — **blocks unattended execution**
    - Iterates child directories of the selected folder, reads each `gpreport.xml` to extract the GPO display name, and calls `Import-GPO -BackupId <GUID> -TargetName <Name> -path <folder> -CreateIfNeeded`
    - No error handling, no idempotency checks, no logging
  - **GPO Backup Sets (7 total, all dated 2019-09-11, domain: `temp.local`, DC: `WIN-6GTSCUPOSSN.temp.local`):**

    | Backup GUID | GPO Display Name | Extensions Present | Security Groups Referenced |
    |---|---|---|---|
    | `{0B2CD77D-...}` | MSFT Windows Server 2019 - Domain Controller Virtualization Based Security | Registry | Enterprise Admins, Domain Admins |
    | `{19F3D672-...}` | MSFT Windows 10 1809 and Server 2019 Member Server - Credential Guard | Registry | Enterprise Admins, Domain Admins |
    | `{B0D41476-...}` | MSFT Windows 10 1809 and Server 2019 - Defender Antivirus | Registry | Enterprise Admins, Domain Admins |
    | `{2931D09B-...}` | Default Workstation Policy | Registry, Security (GptTmpl.inf), Audit Policy (audit.csv) | Administrators, Users, NETWORK SERVICE, LOCAL SERVICE, SERVICE, Remote Desktop Users, Guests, Local account + Admins, Enterprise Admins, Domain Admins |
    | `{396FE7D5-...}` | Default Member Servers Policy | Registry, Security (GptTmpl.inf), Audit Policy (audit.csv), Preferences/Registry, User registry.pol | Local account, Administrators, SERVICE, LOCAL SERVICE, NETWORK SERVICE, Local account + Admins, Authenticated Users, Enterprise Admins, Domain Admins |
    | `{BD2A67B3-...}` | Default Domain Policy | Registry, Security (GptTmpl.inf), Scripts (Startup/Shutdown) | Event Log Readers, NETWORK SERVICE, Enterprise Admins, Domain Admins |
    | `{D5624B22-...}` | Default Domain Controllers Policy | Registry, Security (GptTmpl.inf), Audit Policy (audit.csv), Preferences/Registry, User registry.pol | Administrators, NETWORK SERVICE, LOCAL SERVICE, SERVICE, ENTERPRISE DOMAIN CONTROLLERS, Authenticated Users, Enterprise Admins, Domain Admins |

  - **Audit Policy (audit.csv) — present in Default Workstation Policy, Default Member Servers Policy, Default Domain Controllers Policy:**
    - 23 audit subcategories configured, including: Credential Validation (S+F), Logon (S+F), Account Lockout (F), Process Creation (S), Sensitive Privilege Use (S+F), System Integrity (S+F), Removable Storage (S+F), File Share (S+F), MPSSVC Rule-Level Policy Change (S+F)
  - **Binary artifacts (unreadable as UTF-8):** `gpreport.xml` (all 7 GPOs), `GptTmpl.inf` (security templates), `registry.pol` (registry policy files) — all encoded as UTF-16 LE with BOM

---

### 2.2 Infrastructure Files

| File | Purpose | Migration Considerations |
|---|---|---|
| `host-guardian-service/Root.pem` | Root CA certificate installed into the Enterprise/User root store on the HGS node | Must be stored as a file asset in the Ansible role (`files/` directory); path reference in Step One is relative to script location — must be converted to an absolute Ansible-managed path |
| `host-guardian-service/HGS-Certificate.pfx` | PFX certificate used for both HGS signing and encryption roles; referenced at hardcoded path `C:\Admin\host-guardian-service\HGS-Certificate.pfx` | **High-sensitivity secret.** Must be stored encrypted (Ansible Vault or external secrets manager). The PFX password is currently entered interactively and must become a Vault variable. The reuse of a single PFX for both signing and encryption roles should be reviewed — Microsoft recommends separate certificates. |
| `group-policy-baseline/manifest.xml` | Top-level GPO backup manifest listing all 7 backup instances with GUIDs, domain, DC hostname, backup timestamps, and display names | Serves as the authoritative source of truth for GPO identity mapping in the Ansible role; the target domain (`temp.local`) and DC hostname (`WIN-6GTSCUPOSSN.temp.local`) are hardcoded and must be parameterized as Ansible variables |
| `group-policy-baseline/{GUID}/Backup.xml` (×7) | Per-GPO backup schema defining security group ACLs, GPO extension GUIDs, and SYSVOL file paths | Contains domain SIDs specific to `temp.local` (e.g., `S-1-5-21-3520277273-3852588721-1217316554-519`) that will not match a new target domain; SID migration or re-ACLing will be required |
| `group-policy-baseline/{GUID}/bkupInfo.xml` (×7) | Per-GPO backup metadata (GUID, domain, DC, timestamp, display name) | Used by `Import-GPO` to resolve backup identity; must remain co-located with backup data in the Ansible role's `files/` directory |
| `group-policy-baseline/{GUID}/DomainSysvol/GPO/Machine/registry.pol` (×7) | Binary registry policy files (UTF-16 LE) defining machine-side registry settings | Cannot be parsed or modified as text; must be transferred as binary file assets. Consider using `lgpo.exe` or `Parse-PolFile` (PolicyFileEditor module) to extract human-readable settings for documentation and validation. |
| `group-policy-baseline/{GUID}/DomainSysvol/GPO/Machine/microsoft/windows nt/SecEdit/GptTmpl.inf` (×4) | Security template files (UTF-16 LE) defining account policies, user rights assignments, and security options | Cannot be read as UTF-8; contain domain-specific SIDs that must be remapped for the target domain. These are the most security-critical GPO artifacts. |
| `group-policy-baseline/{GUID}/DomainSysvol/GPO/Machine/microsoft/windows nt/Audit/audit.csv` (×3) | Advanced audit policy CSV files defining 23 subcategory audit settings | Readable as UTF-8; can be parsed and translated to `ansible.windows.win_audit_policy_system` tasks or `auditpol.exe` invocations |
| `group-policy-baseline/{396FE7D5-...}/DomainSysvol/GPO/Machine/Preferences/Registry/` | GPO Preferences registry entries for Default Member Servers Policy | Binary `.xml` preference files; must be treated as file assets |
| `group-policy-baseline/{D5624B22-...}/DomainSysvol/GPO/Machine/Preferences/Registry/` | GPO Preferences registry entries for Default Domain Controllers Policy | Binary `.xml` preference files; must be treated as file assets |

---

### 2.3 Target Details

| Attribute | Value | Source |
|---|---|---|
| **Operating System** | Windows Server 2019 (primary target); Windows 10 1809 (workstation target) | GPO display names explicitly reference "Windows Server 2019" and "Windows 10 1809 and Server 2019"; HGS role requires Windows Server 2016+ |
| **VM Technology** | Not specified in repository; inferred as Hyper-V (HGS/TPM attestation is a Hyper-V shielded VM technology) | `Install-HGS-StepThree.ps1` uses `-TrustTpm` attestation mode, which is the HGS mode for physical TPM-equipped Hyper-V hosts |
| **Cloud Platform** | On-premises / air-gapped (inferred) | USB-stick deployment pattern in Step One (`Split-Path -parent $MyInvocation.MyCommand.Definition`); no cloud provider references anywhere in the repository |
| **Domain** | `temp.local` (source/lab domain) | All GPO backup metadata; `Install-HGS-StepTwo.ps1` uses `cool-name.net` as the HGS forest domain |
| **Domain Controller** | `WIN-6GTSCUPOSSN.temp.local` | All GPO `bkupInfo.xml` and `Backup.xml` files |
| **Ansible Target OS** | Windows Server 2019 | All scripts target Windows; no Linux components present |

---

## 3. Migration Approach

### 3.1 Key Dependencies to Address

1. **`HgsServer` PowerShell Module** — Required by Steps Two and Three. This module ships with the `HostGuardianServiceRole` Windows Feature installed in Step One. Ansible must ensure the feature is installed and the module is available before executing HGS cmdlets via `ansible.windows.win_powershell` or `community.windows.win_psmodule`.

2. **`ActiveDirectory` and `GroupPolicy` PowerShell Modules** — Required by `ImportGPOBulk.ps1`. These are part of RSAT (Remote Server Administration Tools) and must be installed on the Ansible control node's target Windows host (or the DC itself). Use `ansible.windows.win_feature` with `RSAT-AD-PowerShell` and `GPMC` feature names.

3. **`certutil.exe`** — Used in Step One to install `Root.pem`. This is a built-in Windows utility; no additional installation required. The Ansible equivalent is `ansible.windows.win_certificate_store`.

4. **Reboot Sequencing** — The HGS installation has two mandatory reboot points (end of Step One, end of Step Two via `-Restart` flag on `Install-HGSServer`). Ansible must use `ansible.builtin.reboot` (via `ansible.windows.win_reboot`) with appropriate `reboot_timeout` and post-reboot task sequencing. The three steps must be implemented as separate plays or as tasks with `when:` guards based on a persistent state variable or Windows registry check.

5. **`Import-GPO` Cmdlet** — Requires the `GroupPolicy` module and an active connection to a domain controller. The Ansible play executing GPO import must run in the context of a domain-joined machine with appropriate AD permissions (Domain Admins or Group Policy Creator Owners).

6. **COM Shell Object (`Shell.Application`)** — Used in `ImportGPOBulk.ps1` for interactive folder selection. This is incompatible with unattended Ansible execution and must be replaced with a hardcoded or variable-driven path to the GPO backup folder.

7. **Binary GPO Artifacts** — `registry.pol` and `GptTmpl.inf` files are UTF-16 LE encoded and cannot be templated. They must be distributed as binary file assets using `ansible.windows.win_copy` and must not pass through Ansible's Jinja2 template engine.

---

### 3.2 Security Considerations

#### Secrets Inventory

| Secret | Location | Current Handling | Count | Migration Action |
|---|---|---|---|---|
| DSRM Password | `Install-HGS-StepTwo.ps1` | `Read-Host -AsSecureString` (interactive, never stored) | 1 | Store as `ansible_vault`-encrypted variable: `hgs_dsrm_password` |
| PFX Certificate Password | `Install-HGS-StepThree.ps1` | `Read-Host -AsSecureString` (interactive, never stored) | 1 | Store as `ansible_vault`-encrypted variable: `hgs_pfx_password` |
| HGS Certificate (PFX file) | `host-guardian-service/HGS-Certificate.pfx` (referenced path, not in repo) | Stored at `C:\Admin\host-guardian-service\HGS-Certificate.pfx` on target | 1 | Encrypt with Ansible Vault (`ansible-vault encrypt_string` or vault file); deploy via `ansible.windows.win_copy` to a secured path; restrict NTFS ACL post-copy |
| Root CA Certificate (PEM) | `host-guardian-service/Root.pem` | Stored on USB stick alongside scripts | 1 | Store in Ansible role `files/` directory; not a secret but must be integrity-verified before deployment |

**Total secrets requiring Vault protection: 3** (DSRM password, PFX password, PFX file)

#### Additional Security Concerns

1. **HTTP-only HGS Communication:** `Initialize-HgsServer` is called with `-Http` flag (not `-Https`). This means HGS attestation traffic is unencrypted. The Ansible role should parameterize this flag and default to `-Https` with a proper TLS certificate. This is a **security regression** that must be flagged for the target environment owner.

2. **Single PFX for Dual Roles:** The same `HGS-Certificate.pfx` is used for both `-SigningCertificatePath` and `-EncryptionCertificatePath`. Microsoft's HGS documentation recommends separate certificates for signing and encryption. The migration is an opportunity to correct this.

3. **Hardcoded Domain Names:** `cool-name.net` (HGS forest) and `cool-name-here` (HGS service name) are placeholder values in the scripts. These must be replaced with real, parameterized values (`hgs_domain_name`, `hgs_service_name`) before production use.

4. **Domain SIDs in GPO Backups:** All `Backup.xml` files contain hardcoded SIDs from `temp.local` (e.g., `S-1-5-21-3520277273-3852588721-1217316554-519` for Enterprise Admins). When importing into a different domain, `Import-GPO` will attempt SID migration. If the target domain differs from `temp.local`, a SID mapping table must be provided or the GPOs must be re-ACLed post-import.

5. **WinRM Exposure:** Step One runs `winrm quickconfig -force` and `Enable-PSRemoting -force`. The Ansible role must ensure WinRM is configured with HTTPS (not HTTP) and that the WinRM listener is restricted to authorized management networks. This should be enforced via a pre-task or a separate hardening role applied before HGS installation.

6. **`certutil` Enterprise Store:** The Root PEM is installed with `-enterprise` flag, which writes to the machine's enterprise trust store. This is a domain-wide trust operation and should be gated behind a change approval process in the Ansible workflow.

7. **GPO Security Templates (GptTmpl.inf):** Four GPOs contain `GptTmpl.inf` security templates (Default Domain Policy, Default Domain Controllers Policy, Default Workstation Policy, Default Member Servers Policy). These define user rights assignments, account policies, and restricted groups. The binary encoding prevents direct inspection — the migration team must decode these files using `secedit.exe /export` or a UTF-16 reader before migration to understand the full security posture being applied.

---

### 3.3 Technical Challenges

| Challenge | Detail | Mitigation Strategy |
|---|---|---|
| **Multi-reboot HGS orchestration** | HGS installation requires two reboots with different module states before and after each reboot. Ansible plays are stateless across reboots by default. | Use `ansible.windows.win_reboot` with `test_command` to verify post-reboot state. Implement idempotency guards using `win_powershell` to check if HGS features/roles are already installed before re-running steps. Store installation phase state in a Windows registry key (e.g., `HKLM:\SOFTWARE\Ansible\HGS\InstallPhase`) that Ansible reads at play start. |
| **Interactive `Read-Host` prompts** | All three HGS scripts and the GPO import script use interactive prompts that block unattended execution. | Replace all `Read-Host` calls with Ansible variables sourced from `group_vars`, `host_vars`, or Ansible Vault. The computer rename prompt in Step One becomes `hgs_computer_name`; DSRM password becomes `hgs_dsrm_password`; PFX password becomes `hgs_pfx_password`. |
| **COM Shell folder browser** | `ImportGPOBulk.ps1` uses `Shell.Application.BrowseForFolder()` which requires an interactive desktop session and cannot run under WinRM. | Replace with a hardcoded or variable-driven path: `$GPOFolderName = "{{ gpo_backup_path }}"`. The Ansible role should copy the GPO backup folder to the target host before executing the import. |
| **Binary GPO artifacts** | `registry.pol`, `GptTmpl.inf`, and `gpreport.xml` files are UTF-16 LE encoded and will be corrupted if processed as text. | Use `ansible.windows.win_copy` with `force: yes` for all binary GPO files. Never use `ansible.builtin.template` for these files. Validate file integrity with `win_stat` checksum after copy. |
| **Hardcoded paths and names** | `C:\Admin`, `cool-name.net`, `cool-name-here`, `C:\Admin\host-guardian-service\HGS-Certificate.pfx` are all hardcoded. | Extract all hardcoded values into role variables with defaults: `hgs_log_dir` (default: `C:\Admin`), `hgs_domain_name`, `hgs_service_name`, `hgs_cert_path`. |
| **Source domain SID references** | GPO `Backup.xml` files contain SIDs specific to `temp.local`. Importing into a different domain requires SID remapping. | If the target domain differs from `temp.local`, create a SID migration mapping file and pass it to `Import-GPO` via `-MigrationTable`. The Ansible role should include a task to generate or apply a migration table using `New-GPMigrationTable`. |
| **No error handling in source scripts** | None of the four PowerShell scripts contain `try/catch`, `$ErrorActionPreference`, or exit code checks. | Wrap all `win_powershell` tasks with `failed_when` conditions checking `result.rc != 0` and `result.stderr`. Add `$ErrorActionPreference = 'Stop'` at the top of all inline PowerShell blocks. |
| **USB-relative path for Root.pem** | Step One uses `Split-Path -parent $MyInvocation.MyCommand.Definition` to locate `Root.pem` relative to the script. This pattern is incompatible with WinRM-executed scripts. | Pre-stage `Root.pem` to a known path on the target (e.g., `C:\Admin\certs\Root.pem`) using `ansible.windows.win_copy` before executing the certificate installation task. |
| **`Get-HGSTrace -RunDiagnostics` validation** | Step Three ends with a diagnostic trace that outputs to stdout. There is no pass/fail logic. | Capture the output of `Get-HGSTrace` in a `win_powershell` task, register the result, and use `assert` or `fail` to validate expected diagnostic output patterns. |
| **GPO target domain mismatch** | All GPO backups reference `temp.local` as the source domain. If the production target domain has a different name, `Import-GPO` may fail or produce incorrect results. | Parameterize the target domain as `gpo_target_domain` and pass it explicitly to `Import-GPO -TargetName`. Test import in a staging domain before production. |

---

### 3.4 Migration Order (Priority-Based)

#### Priority 1 — Foundation (Week 1)
1. **Ansible environment setup:** Configure WinRM with HTTPS on all target Windows hosts. Create inventory with HGS node group and AD/DC group. Install required Ansible collections: `ansible.windows`, `community.windows`.
2. **Vault scaffolding:** Create `group_vars/hgs/vault.yml` with `hgs_dsrm_password`, `hgs_pfx_password`. Create `group_vars/all/vault.yml` for shared secrets. Encrypt `HGS-Certificate.pfx` for Ansible distribution.
3. **Variable extraction:** Document all hardcoded values (`cool-name.net`, `cool-name-here`, `C:\Admin`, `temp.local`) and create corresponding role defaults in `roles/hgs/defaults/main.yml` and `roles/gpo_baseline/defaults/main.yml`.

#### Priority 2 — HGS Role Migration (Weeks 2–3)
4. **`roles/hgs/` role creation:**
   - `tasks/step_one.yml`: WinRM config check, Root.pem copy + certutil install (→ `win_certificate_store`), Windows Feature install (`HostGuardianServiceRole`, `DNS`), computer rename (→ `win_computer_name`), reboot handler.
   - `tasks/step_two.yml`: HGS server install with domain name and DSRM password variables, reboot handler.
   - `tasks/step_three.yml`: HGS server initialization with parameterized service name, cert paths, attestation mode, and protocol; diagnostic validation task.
   - `tasks/main.yml`: Phase-gated orchestration using registry-based state checks.
   - `files/Root.pem`: Root CA certificate asset.
   - `vars/main.yml` + `defaults/main.yml`: All extracted variables.

#### Priority 3 — GPO Baseline Role Migration (Weeks 3–4)
5. **`roles/gpo_baseline/` role creation:**
   - `tasks/main.yml`: RSAT feature install, GPO backup folder copy to target, bulk import loop replacing `ImportGPOBulk.ps1` logic using `community.windows.win_domain_group_policy` or `win_powershell` with `Import-GPO`.
   - `files/gpo_backups/`: All 7 GPO backup directories with their binary artifacts, copied verbatim.
   - `vars/main.yml`: `gpo_target_domain`, `gpo_backup_path`, `gpo_dc_name`.
   - Add idempotency: check if GPO already exists before importing (`Get-GPO -Name <name> -ErrorAction SilentlyContinue`).

#### Priority 4 — Testing and Validation (Weeks 5–6)
6. **Staging environment test:** Deploy HGS role against a test Windows Server 2019 VM. Validate all three phases complete without interactive prompts. Verify HGS diagnostics pass.
7. **GPO import test:** Deploy GPO baseline role against a test `temp.local` domain controller. Verify all 7 GPOs import with correct names and settings. Validate audit policies using `auditpol /get /category:*`.
8. **Documentation and handoff:** Update role READMEs with variable reference tables, secret management instructions, and rollback procedures.

---

### 3.5 Assumptions

The following ambiguities and unclarities were identified during analysis and must be resolved by the migration team before or during execution:

1. **`cool-name.net` and `cool-name-here` are placeholders.** The actual production HGS domain name and HGS service name are not present in the repository. These must be obtained from the infrastructure owner before the Ansible role can be used in production.

2. **`HGS-Certificate.pfx` is not present in the repository.** The file is referenced at `C:\Admin\host-guardian-service\HGS-Certificate.pfx` but does not exist in the repo. It is assumed to be provisioned out-of-band. The migration team must determine the certificate source, validity period, and whether separate signing and encryption certificates should be generated.

3. **`Root.pem` is present in the repository** (referenced as `.\Root.pem` relative to the script). It is assumed this file exists in the `host-guardian-service/` directory but was not directly readable during analysis. The migration team must confirm this file is present and valid.

4. **The target production domain is unknown.** All GPO backups reference `temp.local` as the source domain. It is unclear whether the production target is also `temp.local` or a different domain. If different, SID migration tables will be required for all 7 GPOs.

5. **The HGS node count is unknown.** The scripts provision a single HGS node. It is unclear whether this is a single-node deployment or the first node of a multi-node HGS cluster. If multi-node, additional `Add-HgsServer` steps will be needed and the Ansible role must be extended.

6. **The `-Http` flag in Step Three may be intentional for an isolated/air-gapped network.** The use of HTTP (not HTTPS) for HGS communication is a security concern but may be deliberate in an air-gapped environment. The migration team must confirm whether HTTPS is required and whether a TLS certificate for the HGS service endpoint is available.

7. **The `gpreport.xml` files are UTF-16 LE encoded and could not be read.** The full content of the GPO reports (which contain the complete list of configured settings) could not be inspected. The migration team should decode these files using a UTF-16 capable reader to fully document the GPO settings before migration.

8. **The `GptTmpl.inf` security templates are UTF-16 LE encoded and could not be read.** The user rights assignments, account policies, and restricted group memberships defined in these templates are unknown. This is the highest-risk unknown in the repository — these settings directly affect domain security posture. The team must decode and document these before migration.

9. **No existing Ansible inventory or control node is referenced.** It is assumed the migration team will provision a new Ansible control node with `ansible.windows` and `community.windows` collections installed, and that WinRM HTTPS connectivity to target Windows hosts will be established as a prerequisite.

10. **The USB-stick deployment model (Step One's `Split-Path` pattern) implies possible air-gapped or disconnected environments.** If the target environment has no network connectivity to an Ansible control node, the migration approach must account for `ansible-pull` or offline artifact staging rather than standard push-mode execution.

11. **No CI/CD pipeline or version control workflow is defined in the repository.** It is assumed the migration team will establish a Git-based workflow with branch protection, peer review for playbook changes, and a staging environment gate before production deployment.

12. **The `Default Domain Policy` and `Default Domain Controllers Policy` GPOs are the built-in AD GPOs** (their GUIDs `{31B2F340-...}` and `{6AC1786C-...}` are the well-known default GPO GUIDs). Importing over these with `Import-GPO -CreateIfNeeded` will overwrite the existing default policies. This is a high-impact, potentially irreversible operation that requires explicit change approval and a backup of the current production GPOs before execution.
