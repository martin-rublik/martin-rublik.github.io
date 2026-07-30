---
layout: post
title: "manipulating proxyAddresses via inbound Entra Connect rules"
date: 2026-07-30 08:00:00 +0100
tags: entra-id entra-connect
image: 202607-proxyAddr-manipulation/proxyAddress-removal.jpg
toc: include
---

## Notes & motivation
During tenant‑to‑tenant migrations, it’s common to modify `userPrincipalName`,
`mail`, `proxyAddresses`, and other Exchange Online attributes. 

Updating `userPrincipalName` and `mail` is
[well‑documented](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-create-custom-sync-rule)
and simple, but `proxyAddresses` tends to require a more nuanced approach.

With Microsoft Identity Manager, this used to be simple; a few lines of C# in an
extension and you could manipulate proxyAddresses however you needed. With
[Entra Sync
functions](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-sync-functions-reference),
the flexibility is much more limited.

In [tenant‑to‑tenant MRS
migrations](https://learn.microsoft.com/en-us/microsoft-365/migration/cross-tenant-mailbox-migration?view=o365-worldwide),
we typically need a mail user object in the target tenant to serve as the
destination for cross‑tenant moves. When hybrid identity is involved (which is
common) and the source Active Directory remains authoritative, the object must
be synchronized into the target tenant with specific attribute transformations.
This becomes particularly tricky when the source primary email domain is being
transferred as part of the migration.

We identified the following requirements for custom transformations:
- user object must be a mail user (involves transformations for
  `msExchRecipientDisplayType`, `msExchRecipientTypeDetails`,
  `msExchRemoteRecipientType` attributes),
- `targetAddress` should be correctly set to the source e-mail address prior
  mailbox finalization,
- and most importantly `proxyAddresses` should have following transformations
  applied:
    - Primary SMTP address needs to be rewritten to temporary e-mail domain
      until domain is transfered,
    - If MOERA smtp alias from the target tenant is set on the source tenant it
      should be kept as well,
    - All existing (expecially x500) non-smtp addresses should be kept.

## Approach to proxyAddresses manipulation
We can either modify the outbound rules from the metaverse to AAD or adjust the inbound rule from AD to the metaverse.

The outbound‑rule approach is easier in terms of attribute transformations, but significantly more complex for the overall Entra Connect configuration. It would require modifying multiple (five or more) default rules and changing their [merge type behavior](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/concept-azure-ad-connect-sync-declarative-provisioning#merging-attribute-values) to `Merge`, especially if the transformation should apply only to a subset of objects or specific connectors (source Active Directories).

The inbound‑rule approach seems more ligthweight when it comes to overall configuration. You only need to modify a single inbound rule, and it can be cleanly scoped or filtered; whether through a controlling attribute, an OU, or a connector.

`proxyAddresses` can be rewritten with this rule:

{% include code-button.html %}
```
RemoveDuplicates(
    Where
    (
        $address,
        Select
        (
            $item,
            ImportedValue("proxyAddresses"),
            IIF
            (
                Left($item,5)="SMTP:",
                IIF(
                    InStr(" @src-domain-to-rewrite1.com @src-domain-to-rewrite2.com ", " @" & Word($item,2,"@") &" ") > 0,
                    Word($item,1,"@") & "@dst-domain-to-rewrite.com",
                    Error("Unexpected primary mail")
                ),
                IIF(
                    Left($item,5)="smtp:",
                    IIF(
                        LCase(Right($item,Len("@dsttenantname.mail.onmicrosoft.com"))) = "@dsttenantname.mail.onmicrosoft.com",
                        $item,
                        ""
                    ),
                    $item
                )
            )
        ),
        False=IsNullOrEmpty($address)
    )
)
```
❗BEWARE❗
> DO NOT USE THIS RULE UNLESS YOU TEST IT FIRST AND UNDERSTAND WHAT IT DOES

## Explanation
Original proxyAddresses rule is much simpler:
```
RemoveDuplicates(Trim(ImportedValue("proxyAddresses")))
```

We decided to keep both [`RemoveDuplicates`](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-sync-functions-reference#removeduplicates) for final processing and [`ImportedValue`](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/concept-azure-ad-connect-sync-declarative-provisioning#importedvalue) as the input.

Next, the [`Select`](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-sync-functions-reference#select) iterates over all proxyAddresses values and evaluates following function:
```
IIF
(
    Left($item,5)="SMTP:",
    IIF(
        InStr(" @src-domain-to-rewrite1.com @src-domain-to-rewrite2.com ", " @" & Word($item,2,"@") &" ") > 0,
        Word($item,1,"@") & "@dst-domain-to-rewrite.com",
        Error("Unexpected primary mail")
    ),
    IIF(
        Left($item,5)="smtp:",
        IIF(
            LCase(Right($item,Len("@dsttenantname.mail.onmicrosoft.com"))) = "@dsttenantname.mail.onmicrosoft.com",
            $item,
            ""
        ),
        $item
    )
)
```

The
[`IIF`](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-sync-functions-reference#iif) function checks whether each address in proxyAddresses starts with `SMTP:`,
i.e., whether it represents the primary email address.


### Rewrite primary e-mail address

If the first `IIF` evaluates to true, we are indeed processing a primary e-mail
address, and we will rewrite its domain to temporary domain. Before doing that,
we need to verify that the primary address’s DNS suffix is one we can and want
to rewrite. That’s where the second `IIF` block comes in.

```
IIF(
    InStr(" @src-domain-to-rewrite1.com @src-domain-to-rewrite2.com ", " @" & Word($item,2,"@") &" ") > 0,
    Word($item,1,"@") & "@dst-domain-to-rewrite.com",
    Error("Unexpected primary mail")
)
```

The
[`InStr`](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-sync-functions-reference#instr)
function checks whether the first string contains the second string (substring).
If the substring is found, it returns the index at which the match occurs; if
not, it returns `0`.

We’re rewriting multiple domains here (*src-domain-to-rewrite1.com*,
*src-domain-to-rewrite2.com*) because we know the local parts (the prefixes) are
unique. This is a bold assumption and should not be taken lightly. 

❗BEWARE❗
> Relying on local-part uniqueness can easily backfire and cause duplicities in
> the synchronization. Do not rewrite multiple domains to a single domain unless
> you are 100% sure there are no conflicts possible.

The string we are checking is a space delimitted list of domains that should be
rewritten. 

The substring we search for is the domain extracted from the email address
obtained by splitting on "@" and then appending a trailing space " ". To extract
the domain portion, we use the
[`Word`](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-sync-functions-reference#word)
function.

This function splits the string using the specified delimiter (in our case "@")
and returns the second element, which corresponds to the domain part of the
email address.

For example, if the email address is `local.part@src-domain-to-rewrite2.com`
the `Word` function returns `src-domain-to-rewrite2.com`, which is then prefixed
by " @" and suffixed with a trailing space.

So we will be searching for: 
- substring: `" @src-domain-to-rewrite2.com "` 
- in the string: `" @src-domain-to-rewrite1.com @src-domain-to-rewrite2.com "`

This substring will match and the address will be rewritten. 

To rewrite the address, we again use the `Word` function to extract the local part
(the prefix) of the email address and then append the temporary domain used in
the target tenant, for example @dst-domain-to-rewrite.com:
```
Word($item,1,"@") & "@dst-domain-to-rewrite.com"
```

In case the primary email address is for example `local.part@some-other-domain.com` the
`InStr` function will return `0` and the synchronization engine will return an
error.
```
Error("Unexpected primary mail")
```

### Keep MOERA aliases from the target tenant
The second `IIF` block is used for aliases, especially the non-primary mail
addresses which start with lowercase `smtp:`. Event though the documentation
does not state it explicitely
[`Left`](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-sync-functions-reference#left)
is case-sensitive.

```
IIF(
    Left($item,5)="smtp:",
    IIF(
        LCase(Right($item,Len("@dsttenantname.mail.onmicrosoft.com"))) = "@dsttenantname.mail.onmicrosoft.com",
        $item,
        ""
    ),
    $item
)
```
The nested `IIF` block effectively acts as a case‑insensitive `endsWith`
function, which Entra Connect rules do not provide out of the box. 

It takes the length of the string `@dsttenantname.mail.onmicrosoft.com` using
[`Len`](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-sync-functions-reference#len),
then extracts that exact number of characters from the rightmost part of the
address. It compares this extracted substring to
`@dsttenantname.mail.onmicrosoft.com` to determine whether the address ends with
the target domain.

If it does it flows the value, otherwise it flows an empty string.

### Other aliases
Finally, if the address does not start with `SMTP:` nor with `smtp:` it is
passed without any transformations. This includes X500 and SIP addresses.

### Cleanup and final processing
After the `Select` function iterates over all proxyAddresses values, we can end up with a collection like this:
```
<empty-string>
SMTP:local.part@dst-domain-to-rewrite.com
<empty-string>
x500:/o=Contoso/ou=Exchange Administrative Group (FYDIBOHF23SPDLT)/cn=Recipients/cn=John.Doe
x500:/o=Contoso/ou=Exchange Administrative Group (FYDIBOHF23SPDLT)/cn=Recipients/cn=John.Doe2
smtp:alias@dsttenantname.mail.onmicrosoft.com
<empty-string>
```

This is almost the final result we want, the only remaining issue is the empty strings. To remove those, we use the [`Where`](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-sync-functions-reference#where) function.

`Where` iterates over all values produced by `Select`
expression and returns only those that satisfy the condition.

In our case, the condition is simply “non‑empty address”, e.g.:
```
False=IsNullOrEmpty($address)
```

This filters out the empty entries and leaves us with a clean, transformed proxyAddresses collection.

## Conclusion
So yes, using inbound rules to manipulate proxyAddresses is absolutely possible,
but it’s not exactly gentle on the brain. As mentioned in the initial warning,
do not use this rule unless you’ve thoroughly tested it and fully understand how
it behaves in your environment.

I'm a [basket case](https://www.youtube.com/watch?v=NUTGr5t3MoY) a
[loco](https://www.youtube.com/watch?v=zhP0AETwfXU) after all.