---
layout: post
title: "tracking oauth scopes in sign‑in logs and upcoming change in conditional access"
date: 2026-03-12 08:00:00 +0100
tags: entra-id kusto conditional-access
toc: include
---

# Motivation and upcoming enforcement for conditional access policies with resource exclusions
Microsoft announced a change to Conditional Access policies in January, with
enforcement scheduled to begin on March 27. The change was communivated via several channels:
* [blog post](https://techcommunity.microsoft.com/blog/microsoft-entra-blog/upcoming-conditional-access-change-improved-enforcement-for-policies-with-resour/4488925)
* [official documentation](https://aka.ms/CAforLowValueScopes)
* [Message
Center](https://admin.cloud.microsoft/#/MessageCenter/:/messages/MC1223829).

You can find it also in the [Merills message center archive](https://mc.merill.net/message/MC1223829).

Quoting from the blog post:
>Today, when a user signs in through a client application that requests only
>OIDC scopes or a limited set of directory scopes, Conditional Access policies
>that target All resources are not enforced if the policy has one or more
>resource exclusions.
>
>After this change, Conditional Access policies that target
>All resources will be enforced for these sign-ins, even when resource exclusions
>are present. This ensures that policies are consistently applied regardless of
>the scope set requested by the application.

To illustrate the change, consider having two applications: `App` and
`AppExcluded` and a conditional access policy `CA Policy 01`. `CA Policy 01` applies to all
resources except `AppExcluded`.

If the user requests an **access token** via client `App` with low priviledged scopes, for example with:
`User.Read`, `openid`, `profile`, `mail`, the conditional access policy
`CA Policy 01` will not be enforced currently.

After the change the `CA Policy 01` will be enforced as well.

# Identifying the affected applications

The following Kusto query can be used to identify affected applications:

{% include code-button.html %}
```kql
SigninLogs
| where TimeGenerated > ago(120d)
| extend APD=todynamic(AuthenticationProcessingDetails)
| mv-expand APD
| where APD.key == "Oauth Scope Info"
| summarize SignInCount=count() by AppDisplayName,ResourceDisplayName,tostring(OAuthScopeInfo=APD.value)
| sort by AppDisplayName
```

Finally, you can export the results and can check which applications might be
affected by the conditional access policy change. 

I was too lazy to come with a KQL solution, so I wrote a simple PS script
that parses the results and checks for requests with low priviledged OAuth
scopes.

{% include code-button.html %}
```powershell
$queryData=Import-Csv ~\downloads\query_data.csv

$nativeClientsLowPriviledgedScopes=@(
    'email', 'offline_access', 'openid', 'profile', 'User.Read', 'People.Read'
)
$confidentialClientsLowPriviledgedScopes=@(
    'email', 'offline_access', 'openid', 'profile', 'User.Read', 
    'User.Read.All', 'User.ReadBasic.All', 'People.Read', 
    'People.Read.All', 'GroupMember.Read.All', 'Member.Read.Hidden'    
)

$retval=@()
foreach($app in $queryData)
{
    $nativeException=$true
    $confidentialException=$true
    $scopes=$app.OAuthScopeInfo | ConvertFrom-Json
    foreach($scp in $scopes)
    {
        if ($scp -notin $nativeClientsLowPriviledgedScopes)
        {
            $nativeException=$false
        }
        if ($scp -notin $confidentialClientsLowPriviledgedScopes)
        {
            $confidentialException=$false
        }
    }
    $retval+=$app | select -Property *,`
        @{n='nativeException';e={$nativeException}},`
        @{n='confidentialException';e={$confidentialException}}
}

$retval | ?{$_.confidentialException -or $_.nativeException} 
```