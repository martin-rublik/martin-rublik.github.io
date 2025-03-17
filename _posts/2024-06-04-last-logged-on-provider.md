---
layout: post
title: "determining last interactive logon method (password, Windows Hello/PIN etc.)"
date: 2024-06-04 08:00:00 +0100
image: lastLoggedOnProvider/lastLoggedOnProvider.jpg
tags: powershell authentication
---

This one will be short.

I always forget how to determine the way the current user logged to Windows interactively. 

This simple script does the trick:

{% include code-button.html %}
```powershell
$lastLogonUiKey = Get-ItemProperty HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Authentication\LogonUI\
$provider=Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Authentication\Credential Providers\$($lastLogonUiKey.LastLoggedOnProvider)"
$provider.'(default)'
```

This script provides you with the last used [credential provider](https://learn.microsoft.com/en-us/windows/win32/secauthn/credential-providers-in-windows).

For the list of providers you can run following:
{% include code-button.html %}
```powershell
$providers=ls "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Authentication\Credential Providers"
$providers | select @{n='GUID';e={ $_.Name.replace('HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Authentication\Credential Providers\','')}}, @{n='Name';e={Get-ItemProperty $_.Name.replace('HKEY_LOCAL_MACHINE','HKLM:\')|select -ExpandProperty "(default)"}}
```

For Windows 11 23H2 you'll see following.

|GUID|Name|
|:--|:--|
|{01A30791-40AE-4653-AB2E-FD210019AE88}|Automatic Redeployment Credential Provider|
|{1b283861-754f-4022-ad47-a5eaaa618894}|Smartcard Reader Selection Provider|
|{1ee7337f-85ac-45e2-a23c-37c753209769}|Smartcard WinRT Provider|
|{2135f72a-90b5-4ed3-a7f1-8bb705ac276a}|PicturePasswordLogonProvider|
|{25CBB996-92ED-457e-B28C-4774084BD562}|GenericProvider|
|{27FBDB57-B613-4AF2-9D7E-4FA7A66C21AD}|TrustedSignal Credential Provider|
|{3dd6bec0-8193-4ffe-ae25-e08e39ea4063}|NPProvider|
|{48B4E58D-2791-456C-9091-D524C6C706F2}|Secondary Authentication Factor Credential Provider|
|{600e7adb-da3e-41a4-9225-3c0399e88c0c}|CngCredUICredentialProvider|
|{60b78e88-ead8-445c-9cfd-0b87f74ea6cd}|PasswordProvider|
|{8AF662BF-65A0-4D0A-A540-A338A999D36F}|FaceCredentialProvider|
|{8FD7E19C-3BF7-489B-A72C-846AB3678C96}|Smartcard Credential Provider|
|{94596c7e-3744-41ce-893e-bbf09122f76a}|Smartcard Pin Provider|
|{BEC09223-B018-416D-A0AC-523971B639F5}|WinBio Credential Provider|
|{C5D7540A-CD51-453B-B22B-05305BA03F07}|Cloud Experience Credential Provider|
|{C885AA15-1764-4293-B82A-0586ADD46B35}|IrisCredentialProvider|
|{cb82ea12-9f71-446d-89e1-8d0924e1256e}|PINLogonProvider|
|{D6886603-9D2F-4EB2-B667-1971041FA96B}|NGC Credential Provider|
|{e74e57b0-6c6d-44d5-9cda-fb2df5ed7435}|CertCredProvider|
|{f64945df-4fa9-4068-a2fb-61af319edd33}|RdpCredentialProvider|
|{F8A0B131-5F68-486c-8040-7E8FC3C85BB6}|WLIDCredentialProvider|
|{F8A1793B-7873-4046-B2A7-1F318747F427}|FIDO Credential Provider|

Unfortunately, Microsoft does not provide much documentation for these providers. 

However, I searched the web and found some useful blogs. 
- ["Three ways of enforcing Security Key sign-in on Windows 10 & Windows 11"](https://swjm.blog/three-ways-of-enforcing-security-key-sign-in-on-windows-10-windows-11-4f0f27227372) by [Jonas Markström](https://swjm.blog/)
- ["AzureAD Passwordless Sign in, forcing Windows 10 to login with FIDO only - Part 3"](https://craigwilson.blog/post/2019/2019-09-29-azuread-passwordless-sign-in-forcing-windows-10-to-login-with-fido-only-part-3/) by [Craig Wilson](https://craigwilson.blog/)

But when you want to go deep, lookup this little gem: ["A primer on writing a credential provider in Windows."](https://dennisbabkin.com/blog/?t=primer-on-writing-credential-provider-in-windows#filter_class) by [Dennis A. Babkin](https://dennisbabkin.com/blog/author/?a=dab). 


