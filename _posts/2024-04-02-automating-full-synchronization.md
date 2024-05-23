---
layout: post
title: "automating full synchronization cycle start of an HR driven provisioning connector app"
date: 2024-04-02 08:00:00 +0100
image: eid-hr-full-sync/eid-hr-logo.jpg
tags: entra-id entra-id-governance
toc: include
---

Recently I had a task to start full synchronization cycle of HR driven provisioning enterprise application programmatically.

According to [Microsoft
documentation](https://learn.microsoft.com/en-us/graph/api/synchronization-synchronizationjob-restart?view=graph-rest-1.0&tabs=http)
you just need to run following:

{% include code-button.html %}
```powershell
Import-Module Microsoft.Graph.Applications

$params = @{
	criteria = @{
		resetScope = "Full"
	}
}
# we want to restart the synchronization job
Restart-MgServicePrincipalSynchronizationJob -ServicePrincipalId $servicePrincipalId -SynchronizationJobId $synchronizationJobId -BodyParameter $params
```

Even though it seems quite simple, in this post I'll elaborate on:
* parameters gathering, e.g.:
  - how do I get the ```$servicePrincipalId``` and ```$synchronizationJobId``` parameters?
* authorization part, how should I:
  - create a service principal under which context will the full sync scheduler / invocation script run
  - assign minimal rights to the app registration/service principal for automating the run
  - put the rights/permissions/grants to "configured permissions list" via MgGraph
* scripts, I'll provide:
  - full script to set-up all necessary prerequisites
  - a script to run the full synchronization

## Parameters gathering
You can get the ```$servicePrincipalId``` and ```$synchronizationJobId``` via different means:
1. [Entra ID portal](https://entra.microsoft.com/)
2. MgGraph PowerShell

### Gathering parameters via Entra ID portal

To get the ```$servicePrincipalId``` via Entra ID portal use following steps.

1. Open the [enterprise application blade in Entra ID portal](https://enapps.cmd.ms)
2. Find the respective HR driven provisioning connector application, choose *Properties*
3. Choose/Copy Object ID, this is the ```$servicePrincipalId```

![](/assets/pictures/eid-hr-full-sync/eid-hr-entra_servicePrincipalId.jpg)

To get the ```$synchronizationJobId``` via Entra ID portal use following steps.
1. Open the [enterprise application blade in Entra ID portal](https://enapps.cmd.ms)
2. Find the respective HR driven provisioning connector application, choose *Provisioning*
3. Select the *View technical information* and copy *Job ID*

![](/assets/pictures/eid-hr-full-sync/eid-hr-entra_servicePrincipalJobId.jpg)

### Gathering parameters via MgGraph
Run following script, you can filter the application by displayName, in this case it is **SuccessFactors to Active Directory User Provisioning**.

{% include code-button.html %}
```powershell
$servicePrincipalId =  Get-MgServicePrincipal -Filter "displayName eq 'SuccessFactors to Active Directory User Provisioning'" | select -ExpandProperty Id
$synchronizationJobId = Get-MgServicePrincipalSynchronizationJob -ServicePrincipalId $servicePrincipalId | Select -ExpandProperty Id
```

So now we have the necessary parameters to run the synchronization scheduler
script. We still need to assign correct rights. 

## Authorization part

### FullSyncScheduler service principal creation
In an ideal world I would use an
[azure
function](https://learn.microsoft.com/en-us/azure/azure-functions/functions-overview?pivots=programming-language-powershell)
and [managed
identity](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview).

Unfortunately I needed to run the scheduling script from on-premises
environment, so I needed to create an [app
registration](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app).

![](/assets/pictures/eid-hr-full-sync/eid-hr-app-registration.jpg)

I've also uploaded a certificate which can be used to authenticate as principal.

![](/assets/pictures/eid-hr-full-sync/eid-hr-cert-upload.jpg)

### Granting rights to FullSyncScheduler
Since we are running under service principal context - *FullSyncScheduler* service principal, we need
application level permissions. 


According to [documentation](https://learn.microsoft.com/en-us/graph/api/synchronization-synchronizationjob-restart?view=graph-rest-1.0&tabs=http#permissions) you need following rights.

| Permission type | Least privileged permissions | Higher privileged permissions |
| ----------- | ----------- | ----------- |
Application	| Application.ReadWrite.OwnedBy |	Synchronization.ReadWrite.All |

[Synchronization.ReadWrite.All](https://graphpermissions.merill.net/permission/Synchronization.ReadWrite.All)
is pretty powerful as it allows the application to configure the Azure AD
synchronization service. 

This basically means that the service principal with such rights could configure
any provisioning application within your Entra ID tenant (including SCIM
provisioning, excluding on-premises hybrid synchronizations such Entra Connect
Sync or Entra Cloud Sync). 

If we don't want to grant such powerful permissions, we need to make the *FullSyncScheduler*
service principal make an application owner. Please note that this cannot be done via GUI/browser and you'll need Graph for the assignment.

{% include code-button.html %}
```powershell
# Connect to MgGraph, we'll need to write to the application
Connect-MgGraph -Scopes Application.ReadWrite.All

# get the FullSyncScheduler service principal Id
$fullsyncServicePrincipalId =  Get-MgServicePrincipal -Filter "displayName eq 'FullSyncScheduler'" | select -ExpandProperty Id

# assign owner of the SuccessFactors to "Active Directory User Provisioning" application
$params = @{
	"@odata.id" = "https://graph.microsoft.com/v1.0/directoryObjects/$fullsyncServicePrincipalId"
}
New-MgServicePrincipalOwnerByRef -ServicePrincipalId $servicePrincipalId -BodyParameter $params
```

Finally we will need to grant ```Application.ReadWrite.OwnedBy``` application admin consent. We do this again by PowerShell
{% include code-button.html %}
```powershell
Connect-MgGraph -Scopes "Application.ReadWrite.All AppRoleAssignment.ReadWrite.All"
# get Graph Resource
$graphServicePrincipal = Get-MgServicePrincipal -Filter "displayName eq 'Microsoft Graph'" 
# find role 
 $appRoleRWOwnedBy=$GraphServicePrincipal.AppRoles | ?{$_.Value -eq 'Application.ReadWrite.OwnedBy'}

$params = @{
	principalId = "$fullsyncServicePrincipalId"
	resourceId = "$($graphServicePrincipal.Id)"
	appRoleId = "$($appRoleRWOwnedBy.Id)"
}

# grant the consent
New-MgServicePrincipalAppRoleAssignedTo -ServicePrincipalId $fullsyncServicePrincipalId -BodyParameter $params
``` 
You can check that permissions were granted successfully in App registrations. You'll see that the permissions are granted, however they are not part of configured permissions list. 

![](/assets/pictures/eid-hr-full-sync/eid-hr-entra_app-permissions.jpg)

I could not find any way how to put the permissions on "configured permissions
list", so I used [Merill's Graph X-Ray
extension](https://graphxray.merill.net/) to sniff out how it is being done in
Entra ID portal. Unfortunately no direct command was fetched by Graph X-Ray,
though you can clearly see that a ```PATCH``` verb was used against an
application registration.

I checked the network trace to reverse engineer the necessary call. The result
is included in following code snippet.

![](/assets/pictures/eid-hr-full-sync/eid-hr-entra_app-configured-permissions-list.jpg)

:exclamation:**BEWARE**:exclamation: 

The code snippet does not check if there
are any other permissions assigned to the FullSyncScheduler. So if you have
other permissions assigned in "configured permissions list" it will overwrite
those. To accommodate existing permissions, you need to combine the existing
```requiredResourceAccess``` with the newly added one. This exercise is out-of-scope and is left to the reader as homework :smiley:.

{% include code-button.html %}
```powershell
# Connect to MgGraph, we'll need to write to the application
Connect-MgGraph -Scopes Application.ReadWrite.All

$fullSyncEntrepriseApp=Get-MgApplication -Filter "displayName eq 'FullSyncScheduler'" 
$graphAppId=$graphServicePrincipal.AppId

$params = @{
	requiredResourceAccess = @(
    @{
        resourceAppId = "$graphAppId"  
	      resourceAccess = @(
          @{
            id = "$($appRoleRWOwnedBy.Id)"
            type = "Role"
          }
        )	
    }
  )
}

# Update the application
Update-MgApplication -ApplicationId  $$fullSyncEntrepriseApp.Id -BodyParameter $params
```

## Scripts
The code snippets in this short post can be summarized into:
- setup script,
- full synchronization script.

### Setup script
{% include code-button.html %}
```powershell
<# PARAMETERS #>
$schedulerAppName="FullSyncScheduler"
$successFactorsProvisioningAppName="SuccessFactors to Active Directory User Provisioning"
$authCertificate=ls Cert:\CurrentUser\my\4312ad9dfb84c891aa3ef5bd137b083449e63931
<# END OF PARAMETERS #>

try
{
    Connect-MgGraph -Scopes "Application.ReadWrite.All AppRoleAssignment.ReadWrite.All"

    # Create Application Registration
    Write-Host "Creating application registration: $schedulerAppName ..."
    $schedulerAppRegistration=New-MgApplication -DisplayName $schedulerAppName 
    Write-Host " OK!"

    # Upload Certificate to Application Registration
    Write-Host "Creating uploading certificate for certificate authentication to: $schedulerAppName ..."
    $params = @{
	    keyCredentials = @(
		    @{
			    endDateTime = $authCertificate.NotAfter
			    startDateTime = $authCertificate.NotBefore
			    type = "AsymmetricX509Cert"
			    usage = "Verify"
			    key = [System.Text.Encoding]::ASCII.GetBytes("$([System.Convert]::ToBase64String($authCertificate.RawData))")
			    displayName = $authCertificate.GetNameInfo("Simple",$false)
		    }
	    )
    }
    $mgResult=Update-MgApplication -ApplicationId $schedulerAppRegistration.Id -BodyParameter $params
    Write-Host " OK!"


    # assign owner rights of the application SuccessFactorsProvisioningServicePrincipal to the FullSyncSchedulerServicePrincipal
    Write-Host "Assigning owner rights on SuccessFactorsProvisioningServicePrincipal to FullSyncSchedulerServicePrincipal..."

    # Creating schedulerAppServicePrincipal
    $schedulerAppServicePrincipal= New-MgServicePrincipal -AppId $schedulerAppRegistration.AppId

    # Searching for SuccessFactorsProvisioning Service Principal
    $successFactorsProvisioningServicePrincipal =  Get-MgServicePrincipal -Filter "displayName eq '$successFactorsProvisioningAppName'"
    
    $params = @{
	    "@odata.id" = "https://graph.microsoft.com/v1.0/directoryObjects/$($schedulerAppServicePrincipal.Id)"
    }
    $mgResult=New-MgServicePrincipalOwnerByRef -ServicePrincipalId $successFactorsProvisioningServicePrincipal.Id -BodyParameter $params
    Write-Host " OK!"

    # Grant the consent for Application.ReadWrite.OwnedBy
    Write-Host "Granting admin consent for  MSGraph/Application.ReadWrite.OwnedBy to FullSyncSchedulerServicePrincipal..."
    # get Graph Resource
    $graphServicePrincipal = Get-MgServicePrincipal -Filter "displayName eq 'Microsoft Graph'" 
    # find role 
    $appRoleRWOwnedBy=$GraphServicePrincipal.AppRoles | ?{$_.Value -eq 'Application.ReadWrite.OwnedBy'}

    $params = @{
	    principalId = "$($schedulerAppServicePrincipal.Id)"
	    resourceId = "$($graphServicePrincipal.Id)"
	    appRoleId = "$($appRoleRWOwnedBy.Id)"
    }
    $mgResult=New-MgServicePrincipalAppRoleAssignedTo -ServicePrincipalId $($schedulerAppServicePrincipal.Id) -BodyParameter $params
    Write-Host " OK!"

    # Add permissions to configured permissions list
    Write-Host "Adding admin consent/OAuth scopes FullSyncScheduler application to configured permissions list..."
    $params = @{
	    requiredResourceAccess = @(
        @{
            resourceAppId = "$($graphServicePrincipal.AppId)"  
	          resourceAccess = @(
              @{
                id = "$($appRoleRWOwnedBy.Id)"
                type = "Role"
              }
            )	
        }
      )
    }

    # Update the application
    $mgResult=Update-MgApplication -ApplicationId  $schedulerAppRegistration.Id -BodyParameter $params
    Write-Host " OK!"

}catch
{
    throw $_
}
```

### Full synchronization script
{% include code-button.html %}
```powershell
<# PARAMETERS #>
# FullSyncScheduler clientId, e.g.: Get-MgApplication -filter "displayName eq 'FullSyncScheduler'" | select -ExpandProperty AppId
$clientId='e6f278e4-8527-435b-8164-cf09359e9a6d' 
# authentication certificate uploaded during setup
$authCertificate=ls Cert:\CurrentUser\my\4312ad9dfb84c891aa3ef5bd137b083449e63931
# tenantId, pretty self-explanatory
$tenantId='46d764b9-0122-4319-8731-df37b7b4fcb8' # put your tenant Id here
$successFactorsProvisioningAppName="SuccessFactors to Active Directory User Provisioning"
<# END OF PARAMETERS #>

try
{
    
    $params = @{
	    criteria = @{
		    resetScope = "Full"
	    }
    }
    # we are automating in service context
    Connect-MgGraph -ClientId $clientId -Certificate $authCertificate -TenantId $tenantId 

    $servicePrincipalId =  Get-MgServicePrincipal -Filter "displayName eq '$successFactorsProvisioningAppName'" | select -ExpandProperty Id
    $synchronizationJobId = Get-MgServicePrincipalSynchronizationJob -ServicePrincipalId $servicePrincipalId | Select -ExpandProperty Id

    # we want to restart the synchronization job
    Restart-MgServicePrincipalSynchronizationJob -ServicePrincipalId $servicePrincipalId -SynchronizationJobId $synchronizationJobId -BodyParameter $params
    
}catch
{
    throw $_
}
```

That's it for today [have fun and snacks :smiley:](https://www.youtube.com/watch?v=HEMjF2n3-SQ).