---
layout: post
title: "how to programmatically sign out a user from new teams"
date: 2024-03-27 08:00:00 +0100
image: teams-signout/teams-signout-logo.jpg
tags: powershell ms-teams
---

Recently, together with my friends and competitors from
[SOFTIP](https://www.softip.sk/), we had a task to programmatically sign-out a
user from MS Teams desktop application on Windows.

We already had a script that signs-out a user from the **old** MS Teams. Since
Microsoft is phasing out the old Teams app, the customer had also **new** MS
Teams app deployed.

Unfortunately, our old script did not really work with the new MS Teams app and
needed to be modified. There were not so many sources online (support forums nor
blog posts), outlining how to do that. So we needed to find our own way.

It took some time, lot of process monitor debugging, troubleshooting, rebooting
and VM snapshots reverting, but finally we had some hypothesis.

It seems the new MS Teams app does not use credential manager at all for caching
sign-in information. Instead it uses AAD Token Broker plugin and stores sign-in
information in a local Teams cache. It also seems that some login related
information is stored in the Edge profile.

Thus, to successfully sign out a user from the **new** MS Teams app we used
following script. 

:exclamation:**BEWARE**:exclamation: 

* **The script is provided under no warranty and MUST be tested before
deployment in any environment.**
* **The script deletes local Edge profile including all bookmarks and other
  customizations.** 
* **The script MUST run under current user context.**
* **After running the script you need to restart the computer, do not run Teams
  app before restarting.**

{% include code-button.html %}
```powershell
# kill new teams and updater
taskkill /IM ms-teams.exe /F
taskkill /IM ms-teamsupdate.exe /F
# kill the broker plugin
taskkill /IM "Microsoft.AAD.BrokerPlugin.exe" /F  
# close the edge
taskkill /IM "msedge.exe" /F  
taskkill /IM "msedgewebview2.exe" /F
sleep 5
# 
# delete local teams cache
rmdir "$($env:LocalAppData)\Packages\MSTeams_8wekyb3d8bbwe\LocalCache\Microsoft\MSTeams" -Force -Recurse
rmdir "$($env:LocalAppData)\Packages\MSTeams_8wekyb3d8bbwe\LocalCache\Local\Microsoft\IdentityCache" -Force -Recurse
rmdir "$($env:LocalAppData)\Packages\MSTeams_8wekyb3d8bbwe\LocalCache\Local\Microsoft\OneAuth" -Force -Recurse
rmdir "$($env:LocalAppData)\Packages\MSTeams_8wekyb3d8bbwe\AC\INetCache\" -Force -Recurse
rmdir "$($env:LocalAppData)\Packages\Microsoft.AAD.BrokerPlugin_cw5n1h2txyewy" -Force -Recurse
# delete token broker cache
rmdir "$($env:LocalAppData)\Microsoft\OneAuth" -Force -Recurse
rmdir "$($env:LocalAppData)\Microsoft\TokenBroker" -Force -Recurse
rmdir "$($env:LocalAppData)\Microsoft\IdentityCache" -Force -Recurse
# delete Edge profile
rmdir "$($env:LocalAppData)\Microsoft\Edge\User Data" -Force -Recurse
# delete registry related infomration
rmdir 'HKCU:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\TokenBroker\ProviderInfo\Microsoft.AAD.BrokerPlugin_cw5n1h2txyewy\' -Force -Recurse
rmdir 'HKCU:\SOFTWARE\Microsoft\IdentityCRL\' -Force -Recurse
rmdir 'HKCU:\SOFTWARE\Microsoft\Teams\' -Force -Recurse
rmdir 'HKCU:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\WorkplaceJoin' -force -recurse
# 
# restart the computer either manually or via shutdown /r /t 120 command
# display the restart pop-up window
$wshell = New-Object -ComObject Wscript.Shell
$wshell.Popup("Restart the computer in order to apply changes.`nDo not run Teams before restarting.",0,"OK",0x0)

```

