---
layout: post
title: "getting parent OU for an object in Active Directory via powershell"
date: 2024-10-09 08:00:00 +0100
tags: powershell active-directory
---


As you all surely know, in Active Directory, X.500, and LDAP, distinguished
names (DNs) are delimited by commas (`,`).

Event though it's not recommended, it is
[possible](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ldap/distinguished-names)
to use comma in relative distinguished names (e.g., in `cn` or `ou` attributes).
I tend to avoid this possibility. Many application vendors do not expect it,
and as a result, the distinguished names may be parsed incorrectly.

Still, many customers have commas in their Active Directory DNs. To parse and
get parent container for these, I came with a simple regex parser/cmd-let.

{% include code-button.html %}
``` powershell
function Get-LdapHelperParentDN
{
    [cmdletbinding(ConfirmImpact = 'Low', SupportsShouldProcess=$true)]
    param(
        [Parameter(Mandatory=$true,ValueFromPipeline = $true)][string]$DistinguishedName
    )
    $regex=New-Object Regex("(?<!\\),")
    $result=$regex.Split($DistinguishedName,2)
    if ($result.Count -gt 1)
    {
        return $result[-1]
    }
    throw "Error splitting distinguished name"
}
```

The most important part of the cmd-let is the regex. We need to match the first
comma (`,`) but the comma can't be prefixed with backslash (`\`).

For that to work we need something called [negative
lookbehind](https://www.regular-expressions.info/lookaround.html).

Quoting:
>
> Lookbehind has the same effect, but works backwards. It tells the regex engine
> to temporarily step backwards in the string, to check if the text inside the
> lookbehind can be matched there. (?<!a)b matches a “b” that is not preceded by
> an “a”

So we need to match comma, not preceded by backslash. Since backslash is also
reserved in regular expressions we will need it twice :grin: 

Thus regex `(?<!\\),`.