Now I'll create a comprehensive migration plan for the ActiveDirectoryHub.ps1 PowerShell DSC configuration to Ansible.

# Migration Plan: ActiveDirectoryHub.ps1

**TLDR**: This PowerShell DSC script configures an Active Directory Domain Controller in an existing domain. It installs necessary Windows features, joins an existing domain as a domain controller, configures base OS settings like UAC and timezone, and monitors the NTDS service. The script is designed to add additional domain controllers to an existing Active Directory forest.

## Service Type and Configuration

**Service Type**: Active Directory Domain Controller

**Key Operations**:
- Configures base OS settings (UAC, timezone, PowerShell execution policy)
- Creates and removes specific directories
- Installs Active Directory Domain Services and related features
- Joins an existing domain as a domain controller
- Monitors the NTDS service

## File Structure

**Scripts:**
- ActiveDirectoryHub.ps1

**DSC Configurations:**
- ActiveDirectoryHub.ps1 (contains Configuration block)

**Related Files:**
- ActiveDirectoryBuild.ps1 (separate configuration for initial domain controller setup)

## Module Explanation

The script performs operations in this order:

1. **ActiveDirectoryHub** (`ActiveDirectoryHub.ps1`):
   - Imports required DSC resources from various modules
   - Sets variables from Azure Automation variables
   - Retrieves credentials from Azure Automation
   - Defines Windows features to install
   - Configures base OS settings (UAC, timezone, execution policy)
   - Creates admin folder and removes API folder
   - Installs Windows features for Active Directory
   - Waits for domain availability
   - Joins an existing domain as a domain controller
   - Monitors the NTDS service
   - Ansible equivalent: Multiple Ansible roles and tasks

## PowerShell to Ansible Mapping

| PowerShell Operation | Ansible Module | Notes |
|---|---|---|
| Import-DscResource | N/A | Ansible uses modules directly without imports |
| Get-AutomationVariable | ansible.builtin.set_fact | Store variables in inventory or vars files |
| Get-AutomationPSCredential | ansible.builtin.set_fact with ansible-vault | Store encrypted credentials |
| xUAC (UAC setting) | community.windows.win_security_policy | Configure UAC settings |
| TimeZone | community.windows.win_timezone | Set timezone |
| PowerShellExecutionPolicy | community.windows.win_shell | Set execution policy |
| File (Directory) | ansible.windows.win_file | Create/remove directories |
| WindowsFeatureSet | ansible.windows.win_feature | Install Windows features |
| WaitForADDomain | community.windows.win_domain_membership with retry | Wait for domain availability |
| ADDomainController | community.windows.win_domain_controller | Join domain as DC |
| Service | ansible.windows.win_service | Monitor and manage services |

## Dependencies

**PowerShell Module dependencies**:
- PSDesiredStateConfiguration
- xPSDesiredStateConfiguration
- ComputerManagementDSC
- xSystemSecurity
- ActiveDirectoryDsc

**Windows Features**:
- AD-Domain-Services
- DNS
- RSAT-AD-PowerShell
- RSAT-ADDS
- RSAT-DNS-Server
- BitLocker
- RSAT-Feature-Tools-BitLocker-BdeAducExt

**Service dependencies**:
- NTDS (Active Directory Domain Services)

## Checks for the Migration

**Files to verify**:
- Admin folder path (from $ADMIN_PATH variable)
- API folder path (from $API_FOLDER_PATH variable)

**Services to check**:
- NTDS service (should be running and set to automatic startup)

## Pre-flight checks:
- Verify domain connectivity: `ansible.windows.win_shell: "nltest /dsgetdc:{{ domain_name }}"`
- Verify required features are available: `ansible.windows.win_feature_info:`
- Check if machine is already a domain controller: `ansible.windows.win_shell: "dcdiag /test:replications"`

## Ansible Playbook Structure

```yaml
---
- name: Configure Active Directory Domain Controller
  hosts: windows_servers
  gather_facts: true
  vars_files:
    - vars/ad_vars.yml
    - vars/credentials.yml  # encrypted with ansible-vault

  tasks:
    # Base OS Settings
    - name: Configure UAC settings
      community.windows.win_security_policy:
        area: System Access
        key: EnableLUA
        value: 1
      
    - name: Set timezone
      community.windows.win_timezone:
        timezone: "Pacific Standard Time"
      
    - name: Set PowerShell execution policy
      community.windows.win_shell: Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope LocalMachine -Force
      
    - name: Create admin folder
      ansible.windows.win_file:
        path: "{{ admin_path }}"
        state: directory
      
    - name: Remove API folder
      ansible.windows.win_file:
        path: "{{ api_folder_path }}"
        state: absent
        
    # Install Windows Features
    - name: Install Active Directory and related features
      ansible.windows.win_feature:
        name:
          - AD-Domain-Services
          - DNS
          - RSAT-AD-PowerShell
          - RSAT-ADDS
          - RSAT-DNS-Server
          - BitLocker
          - RSAT-Feature-Tools-BitLocker-BdeAducExt
        include_sub_features: true
        include_management_tools: true
        state: present
      register: feature_installation
      
    - name: Reboot if required after feature installation
      ansible.windows.win_reboot:
      when: feature_installation.reboot_required
      
    # Wait for domain availability
    - name: Wait for domain to be available
      ansible.windows.win_shell: |
        try {
          $domain = Get-ADDomain -Identity "{{ domain_name }}" -Credential $using:credential
          return 0
        } catch {
          return 1
        }
      vars:
        credential:
          username: "{{ domain_join_user }}"
          password: "{{ domain_join_password }}"
      register: domain_check
      until: domain_check.rc == 0
      retries: 30
      delay: 10
      
    # Join domain as domain controller
    - name: Promote server to domain controller
      community.windows.win_domain_controller:
        dns_domain_name: "{{ domain_name }}"
        domain_admin_user: "{{ domain_controller_join_user }}"
        domain_admin_password: "{{ domain_controller_join_password }}"
        safe_mode_password: "{{ domain_controller_join_password }}"
        state: domain_controller
      register: dc_promotion
      
    - name: Reboot after DC promotion
      ansible.windows.win_reboot:
      when: dc_promotion.changed
      
    # Monitor services
    - name: Ensure NTDS service is running and set to automatic
      ansible.windows.win_service:
        name: NTDS
        start_mode: auto
        state: started
```

## Variables File Structure (vars/ad_vars.yml)

```yaml
---
# Domain settings
domain_name: "your_domain.com"
admin_path: "C:\\Admin"
api_folder_path: "C:\\APIRegistration"

# Feature list
ad_features:
  - AD-Domain-Services
  - DNS
  - RSAT-AD-PowerShell
  - RSAT-ADDS
  - RSAT-DNS-Server
  - BitLocker
  - RSAT-Feature-Tools-BitLocker-BdeAducExt
```

## Credentials File (vars/credentials.yml - encrypt with ansible-vault)

```yaml
---
# Domain join credentials
domain_join_user: "domain\\user"
domain_join_password: "SecurePassword1"

# Domain controller join credentials
domain_controller_join_user: "domain\\admin"
domain_controller_join_password: "SecurePassword2"
```