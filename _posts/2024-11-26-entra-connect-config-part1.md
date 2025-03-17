---
layout: post
title: "Entra Connect Configuration (part 1) - Key Components of Entra Connect Configuration"
date: 2024-11-26 08:00:00 +0100
tags: entra-id active-directory
image: entra-id-custom-claims-testing/entra-id-claims.jpg
toc: include
---

>
> Yikes, I've hit git push too soon, please come back on Tuesday, hopefully 
> this will be completed until then.
> Sorry 😲
> Until then
>  UNDER CONSTRUCTION 

I’ve worked with Entra Connect in various scenarios. As the core component of a Hybrid Identity setup, it offers significant flexibility in its configuration.

In this blog post series, I’ll cover:
- Part 1: Key components of Entra Connect configuration.
- Part 2: Using AADConnectConfigDocumenter to compare Entra Connect configurations.
- Part 3: Identifying and transferring custom rules across different environments.

# Introduction
You may need to align the configuration of Entra Connect in different scenarios:

- Clean installation and migration scenarios within existing tenant and Active Directory infrastructure:
    - Migrating Entra Connect to a **new** server
    - Setting up a **new** staging [Entra Connect server](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-sync-staging-server)
- Existing installations and configuration synchronization and scenarios, e.g.:
    - Aligning configurations between existing servers within the same Active
  Directory forest (e.g., aligning staging and active Entra Connect servers).
    - Synchronizing configurations across development, testing, and production environments.

The easiest way to transfer configurations between Entra Connect instances within
the same Active Directory forest is to [export and import the
configuration](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-import-export-config#import-microsoft-entra-connect-settings).

This approach seamlessly transfers the entire configuration from the source
installation to the new installation, with the exception of user accounts
utilized within the Entra Connect setup.

>:exclamation:BEWARE:exclamation:
>
> While it works perfectly for clean installations, it is not suitable for
> existing deployments.

In scenarios where you need to synchronize configuration between existing
deployments or across different Active Directory forests and Entra ID tenants, a
deeper understanding of the components within the Entra Connect configuration is
essential.

Entra Connect (built on Microsoft Identity Manager) has a configuration
comprising several components. 

These configuration components, ranked from most frequently modified to least, include:
1. **Connectors** (management agents) configuration, especially the: 
    - Included Active Directory partitions and respective organization units.
    - Selected attributes and Active Directory schema.
2. **Custom Rules**,
3. other **Entra Connect specific configuration** such as:
    - **Authentication Policy**: e.g. password hash sync, federated, pass-through authentication, etc.,
    - **Identity Mapping Policy**: anchor selection, join rules for users based on email/SID, etc.,
    - **Exchange Online settings**, such as enabling/disabling hybrid mode or public folder support.
4. **Metaverse** configuration, especially the:
    - configuration of custom attributes.

95% of changes I've seen are comprised of **changing organization units in scope**.
4% of changes in configuration affect **custom rules**.

# Connectors configuration
Even though Microsoft Identity Manager allows many changes in connector
configuration, in Entra Connect the configuration options are limited. In
practice you will mostly find following changes useful:
- Active Directory partitions e.g. (child) domains inclusion and exclusion,
- Organizational units inclusions and exclusions,
- Active Directory attributes inclusions,
- Cached Active Directory schema.

Even though you could change these in Synchronization Service Manager directly you
**SHOULD NOT!**. Use Azure AD Connect wizard instead.

To review the current configuration in Active you can either:
- Run Azure AD Connect wizard and review the configuration manually, or
- Use tools to gather this information from backup:
    - official Microsoft: [Azure AD Connect Configuration Documenter](https://github.com/microsoft/AADConnectConfigDocumenter)
    - my unofficial [Azure AD Backup Tools PowerShell module](fixme)

I'll cover the [Azure AD Connect Configuration
Documenter](https://github.com/microsoft/AADConnectConfigDocumenter) in the next
blog post, today I will go briefly through the [Azure AD Backup Tools PowerShell
module](fixme).

To get the module just run
```PowerShell
Install-Module AADC-BackupTools
```

# Custom rules
> Will be completed hopefully until Tuesday 4th Feb. 25


# Metaverse configuration 
> Will be completed hopefully until Tuesday 4th Feb. 25

# Other Entra Connect configuration
> Will be completed hopefully until Tuesday 4th Feb. 25

