---
layout: post
title: "how to test NPS MFA using radclient"
date: 2024-10-02 08:00:00 +0100
image: entra-id-MFA-radius/entra-id-MFA-radius.jpg
tags: entra-id authentication mfa
toc: include
---

In an Entra ID tenant-to-tenant migration project, we needed to test the
behavior of [Microsoft Network Policy Server
(NPS)](https://learn.microsoft.com/en-us/windows-server/networking/technologies/nps/nps-top),
which was used as a RADIUS server to support [MFA for VPN
access](https://learn.microsoft.com/en-us/entra/identity/authentication/howto-mfa-nps-extension).
We wanted to: 
- understand what it takes to change the NPS MFA adapter configuration, 
- identify the key aspects of this change, and most importantly,
- test it thoroughly before deploying it in production on the day of migration (day-D).

In this post I'll cover, how we approached this task. I'll focus especially on:
- using [freeradius-utils](https://www.freeradius.org/radiusd/man/radclient.html) in [Windows Subsystem for Linux (WSL)](https://learn.microsoft.com/en-us/windows/wsl/install) for testing NPS,
- NPS MFA Adapter configuration/migration aspects.

I'll omit things such as installation and configuration of NPS and WSL. Ping me if you need further info, or check the resources at the bottom. NPS/MFA setup is pretty well documented.

## Installing freeradius-utils
Installing `radclient` is straight forward in Ubuntu (in my case, running in WSL). 

Just execute:

{% include code-button.html %}
```bash
sudo apt-get install freeradius-utils
```

## Initial RADIUS testing with freeradius-utils
To test the NPS configuration, we need:
1. New radius client in your NPS server, generate a secret for the client 
and 

2. Test NPS configuration with a test account using ```radclient``` command. 

The first part is [simple](https://learn.microsoft.com/en-us/windows-server/networking/technologies/nps/nps-radius-clients-configure) with NPS. It is illustrated by next picture.

![](/assets/pictures/entra-id-MFA-radius/entra-id-radius_client-setup.png)

To test using ```radclient``` command,  we need to setup our variables such as ```USERNAME```, ```PASSWORD``` and shared ```SECRET``` first:

{% include code-button.html %}
```bash
# setup username, password and secret
# beware when copying / pasting into terminal as the commands
# perform a read operation :)
echo "Reading one-time settings"
read -e -p " Provide username: " USERNAME
read -s -e -p " Provide user password: " PASSWORD ; echo
read -s -e -p " Provide shared RADIUS secret: " SECRET ; echo
```

Next, we can test authentication:

{% include code-button.html %}
``` bash
CLIENTIP=`ip a s eth0 | grep -E -o 'inet [0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' | cut -d' ' -f2`
echo "Radius Client IP: $CLIENTIP"
echo "Trying to authenticate/authorize $USERNAME"
# authenticate with username/password
echo "User-Name=$USERNAME,User-Password=$PASSWORD,Framed-Protocol=PPP,NAS-IP-Address=$CLIENTIP"  | /usr/bin/radclient -t 60 -x target.server.address:1812 auth "$SECRET"
```

If you are lucky enough, you'll see ```Access-Accept```. This means access was granted.

{% include gimage.html uri="/assets/pictures/entra-id-MFA-radius/entra-id-radius_client-access_accept.png" %}

If you are not so lucky, you'll get ```Access-Reject```, you should
[troubleshoot](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/troubleshoot-network-policy-server)
the NPS then.

{% include gimage.html uri="/assets/pictures/entra-id-MFA-radius/entra-id-radius_client-access_reject.png" %}


## Initial NPS MFA Adapter configuration
Initial NPS MFA Adapter configuration is also simple:
1. Download [NpsExtnForAzureMfaInstaller.exe](https://www.microsoft.com/en-us/download/details.aspx?id=54688) from Microsoft site.
2. Run PowerShell script ```AzureMfaNpsExtnConfigSetup.ps1``` from ```C:\Program Files\Microsoft\AzureMfa\Config```

This script: 
1. Installs entire Microsoft.Graph PS module :hushed:
2. Asks for ```tenantId``` (or reads it from registry)
3. Generates self-signed certificate that is used for communication / authentication between NPS and the Entra ID tenant
4. Grants necessary permissions on the endpoints called by NPS for authenticating/authorizing users via MFA
5. Updates local configuration so that the new certificate is used in the NPS <=> Entra ID communications
5. Grants permissions to the appropriate private key to ```NETWORK SERVICE```
6. Restarts the NPS service (```ias```)

Typical output of the script is illustrated by next picture.

{% include gimage.html uri="/assets/pictures/entra-id-MFA-radius/entra-id-radius_nps_extension_config.png"%}

## MFA RADIUS testing with freeradius-utils
Once the extension is installed and configured, the NPS server will call Entra ID
for MFA for all NPS clients (e.g. VPN gateways) and *by default* will require MFA
for *all users*.

If you *do not have* OTP authentication method enabled, the authenticator will
prompt you for approve sign-in request (without number matching). After
approval, you’ll receive an ```Access-Accept``` or ```Access-Reject``` response, just like
in previous testing scenarios.

On the other hand, if you have enabled OTP authentication method, you'll need to
authenticate "twice":

1. First, you authenticate with username and password, as in the first case. 
The NPS server will respond with an ```Access-Challenge``` and provides ```State``` parameter.
2. Second, you need to specify the ```OTP``` from the MS Authenticator app along
   with a ```State``` challenge provided by the NPS server to complete the
   authentication.


Following snippet will help you with the testing via ```radclient```.

{% include code-button.html %}
``` bash
# authenticate with username/password, 
# get the State parameter from response
STATE=`echo "User-Name=$USERNAME,User-Password=$PASSWORD,Framed-Protocol=PPP,NAS-IP-Address=$CLIENTIP" | /usr/bin/radclient -t 60 -x target.server.address:1812 auth "$SECRET" | grep State | cut -d = -f 2 | tr -d '[:blank:]'`
# read OTP from authenticator
read -e -p " Provide OTP from MS Authenticator: " OTP
# authenticate with OTP, pass the STATE
echo "User-Name=$USERNAME,User-Password=$OTP,State=$STATE,Framed-Protocol=PPP,NAS-IP-Address=$CLIENTIP" | /usr/bin/radclient -t 60 -x target.server.address:1812 auth "$SECRET" 
```

And the testing is illustrated by next picture.

{% include gimage.html uri="/assets/pictures/entra-id-MFA-radius/entra-id-radius_client-radius_two-way-auth.png" %}

## NPS MFA Adapter migration considerations and reconfiguration during migration day 
The reconfiguration of the NPS MFA adapter is straightforward. You simply need
to rerun the PowerShell script ```AzureMfaNpsExtnConfigSetup.ps1```. During the
process, you'll be prompted to select the new ```tenantId```. The rest of the
setup is simple and easy to follow.

> :exclamation: WARNING :exclamation: 
>
> Though reconfiguring NPS is simple, be aware of unintended consequences.
> Previously, all of your users might have been enrolled in MFA. However, during
> tenant-to-tenant migrations, it's common for users to enroll for MFA
> gradually. Fortunately, NPS supports requiring MFA authentication only for
> users who are already enrolled. 

To take advantage of requiring MFA authentication only for users who are MFA
enabled, you must reconfigure NPS server via [registry](https://learn.microsoft.com/en-us/entra/identity/authentication/howto-mfa-nps-extension#prepare-for-users-that-arent-enrolled-for-mfa).

{% include code-button.html %}
``` batch
reg add "HKLM\SOFTWARE\Microsoft\AzureMfa" /v "REQUIRE_USER_MATCH" /t REG_SZ /d "FALSE" /f
```

To revert the configuration either set the value to ```TRUE``` or delete the registry setting.

> :exclamation: WARNING :exclamation: 
>
> Even with ```REQUIRE_USER_MATCH``` set to ```FALSE``` the NPS still queries
> the Entra ID for the user authentication methods. If the user is not found,
> the authentication will fail. Therefore, ensure that all users utilizing the
> NPS/VPN are synchronized with Entra ID.

For users who are not present in Entra ID you'll see these errors:

| EventID | Log                                                    | Details                                                                                                                                                                                                                                                                                                                                                                                                                                   |
|----------|--------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 6273     | Security                                               | Reason Code: 21<br/>Reason: An NPS extension dynamic link library (DLL) that is installed on the NPS server rejected the connection request.|
| 1        | Application and Services Logs\Microsoft\AzureMfa\AuthZ | NPS Extension for Azure MFA: CID: 430ef057-7941-44ff-a0b1-240e7d74cedc : Access Rejected for user <USERNAME> with Azure MFA response: UserNotFound and message: MSODS Graph call returned user not found error. GraphError,SAS.Shared.GraphProvider.Exceptions.GraphUserNotFoundException: [Code:Request_ResourceNotFound] [Message:Resource <USERNAME>' does not exist or one of its queried reference-property objects are not present.] |

The errors are shown below:

{% include gimage.html uri="/assets/pictures/entra-id-MFA-radius/entra-id-radius_missing-user-1.png" %}

{% include gimage.html uri="/assets/pictures/entra-id-MFA-radius/entra-id-radius_missing-user-2.png" %}

## Resources
- [https://learn.microsoft.com/en-us/entra/identity/authentication/howto-mfa-nps-extension](https://learn.microsoft.com/en-us/entra/identity/authentication/howto-mfa-nps-extension)
- [https://learn.microsoft.com/en-us/entra/identity/authentication/howto-mfa-nps-extension-advanced](https://learn.microsoft.com/en-us/entra/identity/authentication/howto-mfa-nps-extension-advanced)
- [https://learn.microsoft.com/en-us/entra/identity/authentication/howto-mfa-nps-extension-vpn](https://learn.microsoft.com/en-us/entra/identity/authentication/howto-mfa-nps-extension-vpn)
- [https://wiki.freeradius.org/config/Radclient](https://wiki.freeradius.org/config/Radclient)


