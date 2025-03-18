---
layout: post
title: "MSOL decommisioning and emergency Entra ID federated domain trust repair"
date: 2025-03-17 08:00:00 +0100
tags: entra-id authentication
image: 202503-entra-id-fed-repair/202503-entra-id-fed-repair.jpg
toc: include
---

Microsoft typically monitors ADFS federation metadata [35 days before the token
signing certificate
expires](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-fed-o365-certs#renewal-notification-from-the-microsoft-365-admin-center-or-an-email)
and updates the federation domain settings automatically.

However, there are cases where Microsoft does not update the token signing
certificate automatically. This can happen if:
- The ADFS metadata is not correctly published on the internet,
- You urgently updated the token signing certificate, or
- You updated the certificate more than 35 days before its expiration.

In these situations, after switching the primary token signing certificate,
users may be unable to sign in to the tenant, which is obviously problematic.

To resolve this, you must manually repair the federation trust as soon as
possible. While Microsoft provides [documentation for such
cases](https://learn.microsoft.com/en-us/microsoft-365/troubleshoot/active-directory/update-federated-domain-office-365),
the procedure relies on the MSOL module.

As you may know, Microsoft is phasing out the MSOL module, with complete
[decommissioning planned for April to late May
2025](https://techcommunity.microsoft.com/blog/microsoft-entra-blog/action-required-msonline-and-azuread-powershell-retirement---2025-info-and-resou/4364991).

Today, I'll show you how to use MgGraph to fix the federation settings. It
involves two steps:
1. Gathering token signing certificate
2. Repairing federation trust.

## Gathering token signing certificate
You can retrieve the token signing certificate in two ways:
- **Directly:** from the ADFS server.
- **Indirectly:** from the federation metadata.

### Getting token signing certificate from ADFS server
This is the easiest and preferred method. You can retrieve the primary (and
secondary) token signing certificate via
[Get-AdfsCertificate](https://learn.microsoft.com/en-us/powershell/module/adfs/get-adfscertificate?view=windowsserver2025-ps)
cmd-let. Next, we take the binary value and Base64 encode it for further use.

{% include code-button.html %}
```powershell
$adfsCerts=Get-AdfsCertificate -CertificateType Token-Signing 
$adfsPrimaryCert=$adfsCerts | ?{$_.IsPrimary}
$adfsNextCert=$adfsCerts | ?{-not $_.IsPrimary}

$b64AdfsPrimaryCert=[System.Convert]::ToBase64String($adfsPrimaryCert.Certificate.RawData)
if ($adfsNextCert)
{
    $b64AdfsNextCert=[System.Convert]::ToBase64String($adfsNextCert.Certificate.RawData)
}
```

### Gathering the token signing certificate from federation metadata
If you don’t have direct access to the ADFS server, you can retrieve the
certificate from the federation metadata instead. 

On a Windows machine with .NET installed, you can use the [FedMetadata
PowerShell module](https://www.powershellgallery.com/packages/FedMetadata/) to
download and parse the federation metadata.

{% include code-button.html %}
```powershell
$samlMetadata=Get-SAMLMetadata -ADFSHostname adfs.company.com -Verbose
$adfsPrimaryCert=$samlMetadata.SigningCertificates[0]
if ($samlMetadata.SigningCertificates.Count -gt 1)
{
    $adfsNextCert=$samlMetadata.SigningCertificates[1]
}

$b64AdfsPrimaryCert=[System.Convert]::ToBase64String($adfsPrimaryCert.Certificate.RawData)
if ($adfsNextCert)
{
    $b64AdfsNextCert=[System.Convert]::ToBase64String($adfsNextCert.Certificate.RawData)
}
```

Or you can just download the federation metadata from URL ```https://ADFS.COMPANY.COM/FederationMetadata/2007-06/FederationMetadata.xml```
and search for ```X509Certificate``` which ```KeyDescriptor``` is set to ```signing```.

{% include gimage.html uri="/assets/pictures/202503-entra-id-fed-repair/metadata-extraction.png" %}

You need to put the values into the ```$b64AdfsPrimaryCert``` and 
```$b64AdfsNextCert``` variables.

## Repairing federation trust
Now that we have the B64 encoded token signing certificates we can repair our
federation trust.

> :bulb: NOTE: Though Microsoft documentation states Hybrid Identity
> Administrator should be sufficient to modify the domain. Unfortunatelly, for
> unknown reason I had to be global admin, don't ask. :confused:

So when updating the federation trust via MgGraph you need at least:
- Hybrid Identity Administrator
- Domain.ReadWrite.All
- and Directory.AccessAsUser.All in some cases.

In my cases it worked with:
- Global Administrator
- Domain.ReadWrite.All
- Directory.AccessAsUser.All in some cases.

> :exclamation: BEWARE :exclamation:
>
> Before running following cmd-lets I strongly suggest backing up the federation
> trust first. This is done for example by serializing the $graphDomain variable
> contents into a json ...

So once you have permissions set-up and roles assigned following command will do
the trick: 

{% include code-button.html %}
```powershell
Connect-MgGraph -Scopes "Domain.ReadWrite.All","Directory.AccessAsUser.All"
$domainName="<your-federated-domain-here>" # e.g. $domainName='rublik.eu'
$graphDomain = Get-MgDomainFederationConfiguration -domainId $domainName
# BACKUP for example as JSON
$graphDomain | ConvertTo-JSON | Out-File "domain-settings-backup.json"

$params = @{
	signingCertificate = $b64AdfsPrimaryCert
}

if ($b64AdfsNextCert)
{
    $params.Add('$nextSigningCertificate',$b64AdfsNextCert)
}

Update-MgDomainFederationConfiguration -DomainId $domainName -BodyParameter $params -InternalDomainFederationId $graphDomain.Id
```

That's it and the federation trust should be updated now.