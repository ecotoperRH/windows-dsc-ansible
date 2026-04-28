# Migration Plan: Office365-TrustedSites.ps1

**TLDR**: This script configures Internet Explorer trusted sites for Office 365 by reading a list of domains from an XML file and adding them to the Windows registry. It sets both HTTP and HTTPS protocols as trusted (value 2) for each domain in the user's registry.

## Service Type and Configuration

**Service Type**: Other (Internet Explorer Security Configuration)

**Key Operations**:
- Reads trusted site domains from an XML file
- Creates registry keys for each domain under HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings\ZoneMap\Domains
- Sets HTTP and HTTPS protocols as trusted (value 2) for each domain
- Handles primary domains and subdomains separately in the registry structure

## File Structure

**Scripts:**
- one-time-config/special-use-case/Office365-TrustedSites.ps1

**Data Files:**
- one-time-config/special-use-case/Office365-TrustedSites.xml

## Module Explanation

The scripts perform operations in this order:

1. **Office365-TrustedSites.ps1** (`one-time-config/special-use-case/Office365-TrustedSites.ps1`):
   - Determines the script's directory path and reads an XML file containing trusted sites
   - Defines the registry path for Internet Explorer trusted sites
   - Defines functions to create registry keys and set registry values
   - Extracts the list of trusted sites from the XML
   - For each trusted site, splits the domain into primary domain and subdomain
   - Creates registry keys for each domain and subdomain
   - Sets HTTP and HTTPS protocols as trusted (value 2) for each domain
   - Ansible equivalent: Use `community.windows.win_regedit` module to create registry keys and set values

## PowerShell to Ansible Mapping

| PowerShell Operation | Ansible Module | Notes |
|---|---|---|
| Get-Content (XML) | ansible.builtin.copy | Copy XML file to target and use with win_template |
| New-Item (Registry) | community.windows.win_regedit | Create registry keys |
| Set-ItemProperty | community.windows.win_regedit | Set registry values |
| Split-Path | ansible.builtin.set_fact | Handle path operations in Ansible |
| String manipulation | ansible.builtin.set_fact | Handle string operations in Ansible |

## Dependencies

**PowerShell Module dependencies**: None  
**Windows Features**: None  
**External packages**: None  
**Service dependencies**: None  

## Checks for the Migration

**Files to verify**:  
- Office365-TrustedSites.xml (should be copied to target)

**Registry keys**:  
- HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings\ZoneMap\Domains\[domain]\[subdomain]
- Values to check: http=2, https=2 for each domain

**Services to check**: None  
**Firewall rules**: None  

## Pre-flight checks:
```powershell
# Check if registry keys exist for a sample domain
Get-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings\ZoneMap\Domains\office365.com\*" -Name http,https -ErrorAction SilentlyContinue
```