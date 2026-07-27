---
layout: post
title: "using OData any() to filter proxyAddresses in Graph API"
date: 2026-07-24 08:00:00 +0100
tags: entra-id powershell mg-graph
toc: include
---

# Notes & motivation
I constantly forget why I did that thing or the other 😆. Since I already had
notes on querying [non‑empty extension
attributes](https://martin.rublik.eu/2025/07/25/searching-graph-for-extensionAttr.html),
I figured I’d extend that topic. The next thing I needed to solve was searching
for proxyAddresses on synchronized objects.

# The filter
For on‑premises synchronized objects, this part is easy; just filter on
```onPremisesSyncEnabled eq true```.

For proxyAddresses the [documentation](https://learn.microsoft.com/en-us/graph/api/resources/user?view=graph-rest-1.0#properties) states:
>Requires $select to retrieve.
>Supports $filter (eq, not, ge, le, startsWith, endsWith, /$count eq 0, /$count
>ne 0).

The tricky part is that proxyAddresses is multivalued. This is where the [Graph OData documentation](https://learn.microsoft.com/en-us/graph/filter-query-parameter?tabs=http#any-operator) helps.

>The any operator iteratively applies a Boolean expression to each item of a
>collection and returns true if the expression is true for at least one item of
>the collection, otherwise it returns false. 

The documentation includes also some examples. For example, the imAddresses property of the
user resource contains a collection of String primitive types. The following
query retrieves only users with at least one imAddress of admin@contoso.com.
```powershell
Import-Module Microsoft.Graph.Users

Get-MgUser -Filter "imAddresses/any(i:i eq 'admin@contoso.com')"
```

And there is even an example for proxyAddresses [with advanced query capabilities](https://learn.microsoft.com/en-us/graph/aad-advanced-queries?tabs=http)
```
~/users?$filter=proxyAddresses/any(p:endswith(p,'contoso.com'))
```

In practice, this filter matches an object if at least one value in proxyAddresses ends with contoso.com. The p variable represents a single item from the multi‑value collection and is used inside the endsWith() function.

# The command
The resulting command is then
```powershell
Get-MgUser -Filter "proxyAddresses/any(p:endsWith(p,'$domainToRemove')) and onPremisesSyncEnabled eq true" `
-ConsistencyLevel eventual `
-CountVariable countvar `
-All
```

The ```ConsistencyLevel``` and ```CountVariable``` parameters are required by [advanced query capabilities](https://learn.microsoft.com/en-us/graph/aad-advanced-queries?tabs=http).
