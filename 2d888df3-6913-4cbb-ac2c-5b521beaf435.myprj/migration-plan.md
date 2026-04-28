# MIGRATION FROM POWERSHELL DSC TO ANSIBLE

## Executive Summary

This repository contains a comprehensive PowerShell Desired State Configuration (DSC) implementation for an Enhanced Security Administrative Forest in a Windows environment. The migration to Ansible will involve converting PowerShell DSC configurations, scripts, and group policies to Ansible roles, playbooks, and modules. The repository is focused on high-security Windows Server environments with Active Directory, Host Guardian Service, and various security configurations.

**Scope**: Migration of 8 primary PowerShell DSC configurations, multiple runbooks, security baselines, and supporting scripts to Ansible.

**Complexity**: High - This migration involves complex Windows domain configurations, security hardening, and specialized Windows features that will require careful mapping to Ansible modules and potentially custom modules.

**Timeline Estimate**: 12-16 weeks
- Analysis & Planning: 2-3 weeks
- Core Infrastructure Migration: 4-6 weeks
- Security Configurations: 3-4 weeks
- Testing & Validation: 2-3 weeks
- Documentation & Knowledge Transfer: 1-2 weeks

## Module Migration Plan

This repository contains PowerShell DSC configurations and scripts that need individual migration planning:

### MODULE INVENTORY

- **ActiveDirectoryBuild**:
    - Description: Configures the first domain controller in a new forest with comprehensive AD structure, security groups, OUs, DNS settings, and user accounts
    - Path: ActiveDirectoryBuild.ps1
    - Technology: PowerShell DSC
    - Key Features: Forest creation, AD structure setup, DNS configuration, security group creation, OU structure, user management

- **ActiveDirectoryHub**:
    - Description: Configures additional domain controllers and hub infrastructure for the Active Directory forest
    - Path: ActiveDirectoryHub.ps1
    - Technology: PowerShell DSC
    - Key Features: Domain controller promotion, replication configuration, site management

- **MemberServer**:
    - Description: Base configuration for standard member servers joining the domain with security settings
    - Path: MemberServer.ps1
    - Technology: PowerShell DSC
    - Key Features: Domain join, BitLocker configuration, security settings, PowerShell execution policy

- **MemberServerSQL**:
    - Description: SQL Server specific configuration for member servers with additional security and performance settings
    - Path: MemberServerSQL.ps1
    - Technology: PowerShell DSC
    - Key Features: SQL Server prerequisites, security configurations, performance optimizations

- **RedForestBuild**:
    - Description: Specialized configuration for building a Red Forest (Enhanced Security Administrative Forest)
    - Path: RedForestBuild.ps1
    - Technology: PowerShell DSC
    - Key Features: Tiered administrative model, privileged access workstation configuration, enhanced security settings

- **S2DHypervisorDell**:
    - Description: Storage Spaces Direct (S2D) configuration for Dell hypervisors
    - Path: S2DHypervisorDell.ps1
    - Technology: PowerShell DSC
    - Key Features: Storage configuration, hypervisor settings, Dell-specific optimizations

- **StandAloneHypervisorDell**:
    - Description: Configuration for standalone Dell hypervisors without S2D
    - Path: StandAloneHypervisorDell.ps1
    - Technology: PowerShell DSC
    - Key Features: Hypervisor configuration, VM host settings, Dell-specific optimizations

- **Host Guardian Service**:
    - Description: Three-step process to configure Host Guardian Service for Shielded VMs
    - Path: host-guardian-service/
    - Technology: PowerShell Scripts
    - Key Features: TPM attestation, key protection, certificate management

- **Group Policy Baseline**:
    - Description: Collection of Group Policy Objects for security hardening and configuration
    - Path: group-policy-baseline/
    - Technology: PowerShell and Group Policy
    - Key Features: Security baselines, administrative templates, Windows settings

- **Code Integrity**:
    - Description: Windows Defender Application Control policies for audit and enforcement
    - Path: code-integrity/
    - Technology: XML Configuration
    - Key Features: Application allowlisting, PowerShell script blocking, Microsoft-only execution

- **One-Time Configuration**:
    - Description: Scripts and configurations for initial setup and special use cases
    - Path: one-time-config/
    - Technology: PowerShell Scripts, Registry Settings
    - Key Features: DSC meta configurations, Office 365 trusted sites, registry modifications

- **Runbooks**:
    - Description: Collection of automation scripts for various administrative tasks
    - Path: runbooks/
    - Technology: PowerShell Scripts
    - Key Features: Guarded host capture, Windows feature management, DSC configuration generation

### Infrastructure Files

- `group-policy-baseline/ImportGPOBulk.ps1`: PowerShell script to import Group Policy Objects in bulk - will need conversion to Ansible tasks for GPO import
- `group-policy-baseline/{GUID}/`: Multiple Group Policy Object backups that need to be converted to Ansible-managed GPOs
- `code-integrity/Audit/AllowMicrosoft_DenyByPassApps_Audit.xml`: Windows Defender Application Control policy in audit mode - will need conversion to Ansible-deployed configuration
- `code-integrity/Enforce/AllowMicrosoft_DenyByPassApps_Enforce.xml`: Windows Defender Application Control policy in enforce mode - will need conversion to Ansible-deployed configuration
- `one-time-config/special-use-case/Office365-TrustedSites.ps1`: Script to configure Office 365 trusted sites - will need conversion to Ansible registry tasks
- `one-time-config/special-use-case/SetStrongCrypto.reg`: Registry settings for TLS configuration - will need conversion to Ansible win_regedit tasks

### Target Details

Based on the source repository analysis:

- **Operating System**: Windows Server 2019 (minimum supported OS mentioned in README.md)
- **Virtual Machine Technology**: Hyper-V with Shielded VMs (based on Host Guardian Service configurations)
- **Cloud Platform**: Microsoft Azure (specifically Azure Automation for DSC Pull Server)

## Migration Approach

### Key Dependencies to Address

- **PowerShell DSC Resources**: The configurations use multiple DSC modules that need Ansible equivalents:
  - **PSDesiredStateConfiguration**: Replace with Ansible Windows modules
  - **xPSDesiredStateConfiguration**: Replace with Ansible Windows modules
  - **ComputerManagementDSC**: Replace with win_hostname, win_timezone, win_powershell modules
  - **xSystemSecurity**: Replace with win_security_policy module
  - **ActiveDirectoryDsc**: Replace with win_domain, win_domain_controller, win_domain_group, win_domain_user modules
  - **xDnsServer**: Replace with win_dns_server module
  - **xDSCDomainjoin**: Replace with win_domain_membership module

- **Azure Automation**: The current solution uses Azure Automation for DSC Pull Server:
  - Replace with Ansible AWX/Tower for orchestration
  - Use Ansible Vault for credential management
  - Implement dynamic inventory for Azure resources

- **Group Policy Objects**: Multiple GPOs need conversion:
  - Use win_group_policy module for simple GPO settings
  - Use PowerShell remoting via Ansible for complex GPO operations
  - Consider using community.windows.win_gpo module for more advanced GPO management

### Security Considerations

- **Credential Management**: The current solution uses Azure Automation credentials:
  - Migrate all credentials to Ansible Vault
  - Implement role-based access control in Ansible Tower/AWX
  - Detected credentials (7 PS Credential objects):
    - DEFAULT_DC_CRED: Domain controller admin credential
    - DOMAIN_CONTROLLER_JOIN: Domain controller join credential
    - DOMAIN_JOIN: Domain join credential
    - TEMP_PASSWORD: Temporary password for user creation
    - Plus 3 additional implied credentials in various scripts

- **Certificate Management**: 
  - Implement certificate deployment using win_certificate_store module
  - Handle Root CA certificates with win_certificate_store
  - Manage TPM attestation certificates with custom Ansible tasks

- **Security Baselines**:
  - Convert Group Policy baselines to Ansible security roles
  - Implement Windows Defender Application Control policies via Ansible
  - Maintain tiered administrative model through Ansible RBAC

- **BitLocker Configuration**:
  - Use win_bitlocker module to manage BitLocker encryption
  - Ensure key management is handled securely through Ansible Vault

### Technical Challenges

- **Host Guardian Service**: The HGS configuration is complex and specialized:
  - Develop custom Ansible roles for HGS configuration
  - Create multi-stage playbooks to handle the three-step HGS setup process
  - Implement TPM attestation handling through custom modules

- **Storage Spaces Direct**: S2D configuration is Windows-specific:
  - Create custom Ansible modules or use win_shell with idempotent checks
  - Develop testing framework for S2D configurations
  - Implement proper error handling for storage configuration

- **Group Policy Management**: Converting GPOs to Ansible is complex:
  - Analyze each GPO to determine settings
  - Create equivalent Ansible tasks for each setting
  - Develop validation tests to ensure equivalent security posture

- **Azure Integration**: The current solution is tightly integrated with Azure:
  - Develop Ansible dynamic inventory for Azure
  - Create playbooks for Azure resource management
  - Implement proper authentication flow for Azure resources

- **Windows Defender Application Control**: WDAC policies are complex XML:
  - Create templates for WDAC policies
  - Implement proper deployment and testing mechanisms
  - Ensure audit mode works correctly before enforcement

### Migration Order

1. **Base Infrastructure**:
   - MemberServer (low complexity, foundation for other configurations)
   - ActiveDirectoryBuild (high value, core infrastructure)

2. **Security Components**:
   - Group Policy Baseline (critical for security posture)
   - Code Integrity (important security component)

3. **Specialized Configurations**:
   - MemberServerSQL (depends on MemberServer)
   - S2DHypervisorDell and StandAloneHypervisorDell (complex storage configurations)

4. **Advanced Features**:
   - Host Guardian Service (complex, depends on core infrastructure)
   - RedForestBuild (most complex, depends on multiple components)

5. **Automation Components**:
   - Runbooks (convert to Ansible playbooks)
   - One-time configurations (convert to Ansible tasks)

### Assumptions

1. The target environment will continue to be Windows Server 2019 or newer.
2. Azure will remain the cloud platform for orchestration and automation.
3. The security posture and tiered administrative model must be maintained.
4. The migration will be done in phases, with parallel running of both systems during transition.
5. The current DSC configurations are working correctly and can be used as functional specifications.
6. Domain structure and naming conventions will remain the same.
7. The team has expertise in both PowerShell DSC and Ansible or will receive training.
8. The Ansible control node will have appropriate network access to manage Windows servers.
9. The target environment supports WinRM for Ansible management.
10. Group Policy Objects are standard and don't contain custom ADMX templates.
11. The Host Guardian Service configuration doesn't have environment-specific customizations not visible in the code.
12. Storage Spaces Direct configurations are standard and don't require vendor-specific drivers beyond what's in the scripts.