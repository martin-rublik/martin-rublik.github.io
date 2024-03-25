---
layout: post
title: "comparing files, directories and binaries with powershell"
date: 2024-03-22 08:00:00 +0100
image: fs-cmp/fs-cmp-logo.jpg
tags: one-liner powershell
---

Yesterday, I needed to compare binaries deployed in different environments. I
could not copy the files, and I could use only native platform tools. 

Here is the one-liner :smiling_imp: I prepared to get the information. The steps
how I prepared and used the command are described next :information_desk_person:.

{% include code-button.html %}
```powershell
ls . -Recurse | ?{$_.Extension -in ".dll",".exe"} | foreach {Get-FileHash $_.FullName | select *,@{n='RelativePath';e={$_.Path.Replace("$($pwd)","")}} -ExcludeProperty Path | sort -property RelativePath} | ConvertTo-Json | Set-Clipboard
```

I decided to use PowerShell, and thought of two ways how to get the task done:
1. Compare DLLs and EXEs versions via
   ```[System.Diagnostics.FileVersionInfo]::GetVersionInfo``` function.
2. Use a ```Get-FileHash``` function to compare the hashes of the files.

I decided to use the second approach. Version information is not so reliable and
in some cases might not be updated. On the other hand, the hash is changing each
time a file changes. So I started building a one-liner which could be used for comparison.

First, I needed to list all files within a directory. Simple:
```powershell
ls . -Recurse
```

I was not really interested in configuration files, nor other non-executable
files. This was an ASP.NET web service containing only DLL executables. So my filter was
again simple:
```powershell
ls . -Recurse -Include *.dll
```

If I would need to include ```*.exe``` as well, it would be problematic.
According to [superuser: How do I get get-childitem to filter on multiple file
types?](https://superuser.com/questions/318197/how-do-i-get-get-childitem-to-filter-on-multiple-file-types),
I found that filtering files might simpler and faster after enumeration, thus I used this approach.
```powershell
ls . -Recurse | ?{$_.Extension -in ".dll",".exe"} 
```

So, now we had all DLL and EXE files. I've piped them to the ```Get-FileHash```, easy.
```powershell
ls . -Recurse | ?{$_.Extension -in ".dll",".exe"} | foreach {Get-FileHash $_.fullname}
```

We could just, pipe the output to the clipboard or to a file and we'd be done.
Afterwards we could use our favorite text editor and compare the hashes between the environments.

Sadly, in my case the files were not in the same path (in different
environments), thus it was not so easy. Still, the directory structure was the
same. So I decided to get a relative path instead of full path. To achieve that
I used:
```powershell
select *,@{n='RelativePath';e={$_.Path.Replace("$($pwd)","")}} -ExcludeProperty Path | sort -property RelativePath}
```

And after that I've converted output to JSON which could be easily compared. 

You can check the output here (bogus, but you'll get the message).

![](/assets/pictures/fs-cmp/fs-cmp-winmerge.jpg)