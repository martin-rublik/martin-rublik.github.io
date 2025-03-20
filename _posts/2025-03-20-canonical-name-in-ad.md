---
layout: post
title: "getting canonical name in Active Directory via PowerShell  "
date: 2025-03-20 08:00:00 +0100
tags: powershell active-directory
toc: include
---

Some time ago I wrote a blog post about [distinguished names and how to compute
a parent in Active Directory using
powershell](https://martin.rublik.eu/2024/10/09/parent-ou-in-active-directory.html)).

Today I'm following-up with a simple PowerShell function for translating
distinguished name into a [canonical
name](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/b645c125-a7da-4097-84a1-2fa7cea07714#gt_79ab9d86-0d30-41c3-b7da-153ad41bdfd8).

{% include code-button.html %}
```powershell
function Get-CanonicalName
{
    [cmdletbinding(ConfirmImpact = 'Low', SupportsShouldProcess=$true)]
    param(
        [Parameter(Mandatory=$true,ValueFromPipeline = $true)][string]$DistinguishedName
    )
    begin
    {
        # nothing to do here we don't need to setup anything if 
        # multiple DNs are at the pipeline
    }
    process
    {
        # process each DistinhuishedName from the pipeline
        if ($DistinguishedName -match "^dc=")
        {
            ((($DistinguishedName -split ",DC=") -join ".").TrimStart("DC="))+"/"
        }else
        {
            $dcregex=New-Object Regex("(?<!\\),DC=")
            $dcprefix=$dcregex.Split($DistinguishedName,2)[-1].Replace(",DC=",".")
            $objDN=$dcregex.Split($DistinguishedName,2)[0]
            $regex=New-Object Regex("(?<!\\),")
            $components=$regex.Split($objDn)
            [array]::Reverse($components)
            $retval="$dcprefix"
            foreach($rdn in $components)
            {
                $name=$rdn.Split('=',2)[-1]
                $retval+="/$name"
            }
            $retval
        }
    }
    end
    {
        # Nothing to do here, we don't need any specific cleanup
    }
}
```

The PowerShell script isn't easy to read, so I’ll try to explain it step by step.

The first condition handles a corner case.
```powershell
if ($DistinguishedName -match "^dc=")
{
    ((($DistinguishedName -split ",DC=") -join ".").TrimStart("DC="))+"/"
}
```
In this case we have only domain naming context without any
objects nor OUs, thus we'll process only the domain context. It is rather simple: 
1. Split the string by domain components ```DC=```
2. Join the parts with a ```.``` (dot)
3. Remove ```DC=``` from the start of the string
4. Append a trailing slash ```/```

This way, ```DC=ad,DC=domain,DC=local``` will be converted to ```ad.domain.local/```

In the second case we have two parts:
- Domain context part
- oOject relative part (within domain context).

The domain context part processed the same way as in the first case. 

The object relative part is processed this way:
1. Split the object by ```,``` using a [negative
   lookbehind](https://martin.rublik.eu/2024/10/09/parent-ou-in-active-directory.html)
   to ignore ```\,``` which is used as escape character for comma in DN.
2. Reverse the result. 
3. Parse each [relative distinguished name
   (rdn)](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/b645c125-a7da-4097-84a1-2fa7cea07714#gt_22198321-b40b-4c24-b8a2-29e44d9d92b9)
   and remove the ```CN=``` and ```OU=``` components, keeping only the names of the
   [containers](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/b645c125-a7da-4097-84a1-2fa7cea07714#gt_c3143e71-2ada-417e-83f4-3ef10eff2c56),
   not their objectClass equivalents.
4. Concatenate the strings 

That's it, happy canonical naming.

