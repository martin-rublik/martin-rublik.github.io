---
layout: post
title: "troubleshooting log analytcis queries from az-cli"
date: 2024-06-11 08:00:00 +0100
image: az-cli-and-loganalytics/az-cli-and-loganalytics.jpg
tags: powershell troubleshooting log-analytics tls
toc: include
---

Today I needed to query the Log Analytics workspace from a script to retrieve logs from both cloud and on-premises environments for double-checking.

Everything worked nicely until I noticed strange things happening :stuck_out_tongue_closed_eyes:. I've noticed that the KQL query returns the same number of results and does not include extended properties / columns 😖.

In this post I'll outline:
1. The issue with az-cli
2. How to use Fiddler to debug the az-cli network communications
3. A workaround for the issue

# The issue with az-cli
There's an [old issue](https://stackoverflow.com/questions/65585260/how-can-i-run-a-multi-line-query-while-using-az-monitor-app-insights-query) in [az cli tool](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) when it comes to querying azure log analytics workspace. I didn't know about that :disappointed:. As far as I can tell, the bug is present at least from 2.42.0 until now (version: 2.61.0).

The issue is that you really can't use a multiline analytics query inside the [```az monitor log-analytics query```](https://learn.microsoft.com/en-us/cli/azure/monitor/log-analytics?view=azure-cli-latest#az-monitor-log-analytics-query) command. 

What's worse, ***you may not see any errors***, if you put a multiline query there. It just won't return expected results.

Imagine a following script (does not make much sense, but for the purpose of this blog-post, it is sufficient).
```powershell
$query = "SigninLogs | where TimeGenerated > ago(30minutes) | distinct UserPrincipalName,tostring(DeviceDetail.displayName) | limit 10"
$signInLogs=$(az monitor log-analytics query -w $sentinelLA.customerId --analytics-query $query) | ConvertFrom-Json
```

Everything works fine 👍.

Now since I like reading my queries formated I decided to rewrite it a little. It is still the same query, just different formatting.
```powershell
$query = @"
SigninLogs 
| where TimeGenerated > ago(30minutes) 
| distinct UserPrincipalName,tostring(DeviceDetail.displayName) 
| limit 10
"@
$signInLogs=$(az monitor log-analytics query -w $sentinelLA.customerId --analytics-query $query) | ConvertFrom-Json
```

Ufortunatelly, this is a bummer. The query executes successfully, but the
results are not filtered and contain all the columns from SignInLogs table.

# How to use Fiddler to debug the az-cli network communications
First, I tried to use the built-in capabilities for troubleshooting and determining the root cause. Therefore, I ran the following command to get more verbose output:

```powershell
az monitor log-analytics query -w $sentinelLA.customerId --analytics-query $query --verbose --debug
```

Though I got more information it didn't help much. Fiddler to the rescue :ambulance:.

I opened Fiddler, [enabled TLS inspection](https://docs.telerik.com/fiddler/configure-fiddler/tasks/decrypthttps) and started automatic network traffic capturing (F12). Still I had not seen any powershell related traffic there 😕. 

So I googled on [how to enable proxy for az cli tool](https://docs.telerik.com/fiddler/configure-fiddler/tasks/decrypthttps).
```powershell
#fiddler listens by default on 127.0.0.1:8888
$env:HTTP_PROXY="http://127.0.0.1:8888"
$env:HTTPS_PROXY="http://127.0.0.1:8888"
```

Using environment variables to set a proxy should already ring a bell, but I was
impaitent and ran the query from powershell. I've got an error that the
certificate is not trusted. 

If you check the image below you will see that: 
- az cli is implemented via python (thus the environment variables for setting up a proxy). 
- Microsoft nicely included [configuration steps necessary](https://docs.microsoft.com/cli/azure/use-cli-effectively#work-behind-a-proxy) to trust the TLS inspection certificate.

![](/assets/pictures/az-cli-and-loganalytics/az-cli-tls-cert-error.png)

So after exporting the Fiddler CA base64 encoded certificate and adding it to ```C:\Program Files (x86)\Microsoft SDKs\Azure\CLI2\Lib\site-packages\certifi\cacert.pem``` I was able to inspect the network communications finaly.

The first image corresponds to a single-line query.

![](/assets/pictures/az-cli-and-loganalytics/az-cli-single-line.png)

For the sake of clarity, the request body in the first case is:
```json
{"query": "SigninLogs | distinct UserPrincipalName,tostring(DeviceDetail.displayName),TimeGenerated | top 10 by TimeGenerated"}
```

The second image corresponds to a multi-line query. 

![](/assets/pictures/az-cli-and-loganalytics/az-cli-multi-line.png)

The body in the second case is:
```json
{"query": "SigninLogs"}
```
It is apparent that only first line is included thus you'll get all the SigninLogs data, no filters, no projections.

# Workaround
To workaround this issue you need to replace all newlines. This can be done easily using the PowerShell ```-replace``` operator just after the string definition:
```powershell
$query = @"
SigninLogs 
| where TimeGenerated > ago(30minutes) 
| distinct UserPrincipalName,tostring(DeviceDetail.displayName) 
| limit 10
"@ -replace [System.Environment]::NewLine
$signInLogs=$(az monitor log-analytics query -w $sentinelLA.customerId --analytics-query $query) | ConvertFrom-Json
```

Now the az cli tool will get a single line query which is correctly formated and I can see all the query formated as I wish.

Case closed.



