# Migration Plan: Host Guardian Service Installation

**TLDR**: This PowerShell code installs and configures Microsoft's Host Guardian Service (HGS) in a three-step process. It installs required Windows features, configures WinRM, sets up certificates, creates an HGS domain, and initializes the HGS server with signing and encryption certificates.

## Service Type and Configuration

**Service Type**: Host Guardian Service (Security/Attestation Service)

**Key Operations**:
- Enable WinRM and PowerShell Remoting
- Install Root Certificate for proxy access
- Install Host Guardian Service Role and DNS features
- Rename the computer and restart
- Create a new HGS domain
- Initialize the HGS server with signing and encryption certificates
- Run HGS diagnostics

## File Structure

**Scripts:**
- host-guardian-service/Install-HGS-StepOne.ps1
- host-guardian-service/Install-HGS-StepTwo.ps1
- host-guardian-service/Install-HGS-StepThree.ps1

**Modules:**
- None (uses built-in HgsServer module)

**Data Files:**
- Root.pem (referenced but not listed in directory)
- HGS-Certificate.pfx (referenced but not listed in directory)

## Module Explanation

The scripts perform operations in this order:

1. **Install-HGS-StepOne.ps1** (`host-guardian-service/Install-HGS-StepOne.ps1`):
   - Gets the current script directory path
   - Enables WinRM and PowerShell Remoting
   - Installs a root certificate for proxy access
   - Installs the Host Guardian Service Role and DNS Windows features
   - Prompts for a new computer name and renames the computer
   - Restarts the computer
   - Ansible equivalent: Use win_shell for WinRM configuration, win_certificate_store for certificate installation, win_feature for role installation, and win_hostname for computer renaming

2. **Install-HGS-StepTwo.ps1** (`host-guardian-service/Install-HGS-StepTwo.ps1`):
   - Imports the HgsServer module
   - Prompts for DSRM (Directory Services Restore Mode) password
   - Installs the first node in the HGS cluster with a specified domain name
   - Restarts the computer
   - Ansible equivalent: Use win_shell to run the HgsServer module commands

3. **Install-HGS-StepThree.ps1** (`host-guardian-service/Install-HGS-StepThree.ps1`):
   - Imports the HgsServer module
   - Prompts for PFX certificate password
   - Initializes the HGS server with signing and encryption certificates
   - Runs HGS diagnostics
   - Ansible equivalent: Use win_shell to run the HgsServer module commands

## PowerShell to Ansible Mapping

| PowerShell Operation | Ansible Module | Notes |
|---|---|---|
| winrm quickconfig -force | ansible.windows.win_shell | Run WinRM configuration command |
| enable-psremoting -force | ansible.windows.win_shell | Enable PowerShell remoting |
| certutil -addstore | community.windows.win_certificate_store | Add certificate to certificate store |
| Install-WindowsFeature | ansible.windows.win_feature | Install Windows features |
| Rename-Computer | ansible.windows.win_hostname | Change computer name |
| Restart-Computer | ansible.windows.win_reboot | Restart the Windows server |
| Import-Module HgsServer | ansible.windows.win_shell | Import PowerShell module |
| Read-Host with -AsSecureString | ansible.builtin.pause or ansible.builtin.vars_prompt | Prompt for sensitive input |
| Install-HGSServer | ansible.windows.win_shell | Run HGS server installation |
| Initialize-HgsServer | ansible.windows.win_shell | Initialize HGS server |
| Get-HGSTrace | ansible.windows.win_shell | Run HGS diagnostics |

## Dependencies

**PowerShell Module dependencies**: HgsServer
**Windows Features**: HostGuardianServiceRole, DNS
**External packages**: None
**Service dependencies**: WinRM, Host Guardian Service

## Checks for the Migration

**Files to verify**: 
- C:\Admin\host-guardian-service\HGS-Certificate.pfx
- Root.pem in script directory

**Registry keys**: None explicitly modified

**Services to check**: 
- Host Guardian Service
- DNS Service

**Firewall rules**: None explicitly created (though HGS installation may create some)

## Pre-flight checks:
```powershell
# Check if WinRM is configured
Test-WSMan

# Check if required Windows features are installed
Get-WindowsFeature HostGuardianServiceRole, DNS

# Check HGS service status
Get-Service GuardianService -ErrorAction SilentlyContinue

# Verify HGS configuration
Import-Module HgsServer
Get-HgsServer
```