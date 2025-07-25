---
layout: post
title: "searching entra id users via mg-graph on non-empty values in extension attributes"
date: 2025-07-25 08:00:00 +0100
tags: powershell entra-id mg-graph
---

Hi, long time no see 🥲. 

Recently, I needed to search MG Graph for users  with a specific [on premises
extension
attribute](https://learn.microsoft.com/en-us/graph/api/resources/onpremisesextensionattributes?view=graph-rest-1.0)
set.

Even though the documentation states it is
[supported](https://learn.microsoft.com/en-us/graph/api/resources/user?view=graph-rest-1.0#properties):

|*Supports $filter (eq, ne, not, in).* |

I was not able to perform a successful search for non-null values, or, in fact for no values 🤷.

To search for values via Microsoft Graph you need to use two important parameters:
* `ConsistencyLevel` - set to `Eventual`
* `Count` - must be defined

I was aware of the `ConsistencyLevel`, which is explained in detail here:
[https://ourcloudnetwork.com/understanding-consistencylevel-eventual-with-microsoft-graph-powershell/](https://ourcloudnetwork.com/understanding-consistencylevel-eventual-with-microsoft-graph-powershell/).

Unfortunately I had forgotten about the `Count` variable 🤦. You can find more
details about it in the official [Microsoft
documentation](https://learn.microsoft.com/en-us/graph/query-parameters?tabs=http#count).

Without the `Count` variable or if you omit the `ConsistencyLevel` parameter/header you'll encounter following error:

```
Get-MgUser : Filter operator 'NotEqualsMatch' is not supported.
Status: 400 (BadRequest)
ErrorCode: Request_UnsupportedQuery
```

{% include gimage.html uri="/assets/pictures/202507-mg-graph-search/202507-searching-error.png" %}

To successfully execute the query you'll need specify both:

{% include code-button.html %}
```powershell
Get-MgUser -Filter "onPremisesExtensionAttributes/extensionAttribute1 ne null" -ConsistencyLevel Eventual -CountVariable countvar -All
```

If you prefer calling MG Graph via HTTP, use this is the equivalent:
```
ConsistencyLevel: Eventual
GET /v1.0/users?$filter=onPremisesExtensionAttributes%2FextensionAttribute1%20ne%20null&$count=true HTTP/1.1
```

or
{% include code-button.html %}
```powershell
 Invoke-WebRequest -Headers @{ 
                                "ConsistencyLevel"="Eventual" 
                                "Authorization"="Bearer <access-token-here>" 
                            } `
    -Uri "https://graph.microsoft.com/v1.0/users?`$filter=$([uri]::EscapeDataString('onPremisesExtensionAttributes/extensionAttribute1 ne null'))&`$count=true"
```

Still, with the HTTP approach, you will need to implement your own paging. 

That's all folks, see you next time 💙.
