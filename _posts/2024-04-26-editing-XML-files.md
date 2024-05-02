---
layout: post
title: "editing XML config files with powershell"
date: 2024-04-22 08:00:00 +0100
image: editing-xml/editing-xml.jpg
toc: include
tags: powershell
---

A long, long time ago in a server room far, far away I needed to automate
deployment of web services in IIS server and parametrize the web services via
script.

This lead me to creation of Edit-Config.ps1 which I published also in [Technet
Gallery (which was retired
12/2020)](https://learn.microsoft.com/en-us/teamblog/technet-gallery-retirement). 

Recently, I needed to use that script again, so I decided to publish it in
PowerShell gallery. In this post, I will provide:
- Steps to install the module/script.
- Common scenarios where the script can be useful.
- Notes on changes made since the original version prepared in 2015.

:exclamation:**BEWARE**:exclamation: 
Please be cautious when using this script and thoroughly test it before implementation.

Nevertheless, I hope it proves helpful to someone :smiley:.


## Set-Config module installation
Simply:

{% include code-button.html %}
```powershell
Install-Module Set-Config
```



## Usage
To edit an XML config file, follow these steps:
1. **Backup the Original File:** Before making any changes, create a backup of
   the original XML config file to ensure you can revert back if needed.
2. **Determine the XPath:** Identify the
   [XPath](https://en.wikipedia.org/wiki/XPath) expression that targets the
   specific attribute, value, or element text you want to modify within the XML
   file.
3. **Run Set-Config:** Utilize the ```Set-Config``` (formerly ```Edit-Config```)
   script to programmatically update the XML config file based on the specified
   XPath and desired changes.
4. **Verify the Contents:** After running the script, verify the updated
   contents of the XML file. You can use tools like
   [WinMerge](https://winmerge.org/?lang=en) or
   [VSCode](https://code.visualstudio.com/).
   
   

For XPath determination, consider using XML-specific tools available for text
editors like Visual Studio Code with [XML Tools
plugin](https://marketplace.visualstudio.com/items?itemName=DotJoshJohnson.xml), or
Notepad++ with [XML Tools](https://github.com/morbac/xmltools).

![](/assets/pictures/editing-xml/editing-xml_xpath.jpg)

Determining XPath is 99% of the magic you need to know. Here are some
[resources](https://www.w3schools.com/xml/xpath_syntax.asp) for a
[better](https://www.geeksforgeeks.org/introduction-to-xpath/)
[start](https://devhints.io/xpath).

It also covers 50% of your problems with editing the XML config. The remaining
50% is that you did not backup the original XML config. So don't forget that
:innocent:.

```Set-Config``` cmd-let requires these parameters: 

| ParameterName | Description |
| ------------- | ----------- |
|```ConfigFileName```| This is the path to the Xml config you want to edit (and you have backed up for sure in the first place). |
|```XPath```|XPath determining the node you want to edit.|


When you know the file and the XML path you also want to usually set something
:smiley:. 

For this purpose, you can use one or more following parameter(s):

| ParameterName | Description |
| ------------- | ----------- |
|```Attribute```| Name of attribute you want to set in an element/node. |
|```Value```|Value of attribute you want to set.|
|```InnerText```|Inner text to set in an element/node.|

To preview changes, you can run the ```Set-Config``` with ```-WhatIf```
parameter. 

Enough, let's see some examples!

### Example 1: Setting a certificate thumbprint in Web.config
One of common scenarios is modifying web service certificate thumbprint
parameter in the ```Web.config``` configuration file. This is typically found in
[serviceCertificate](https://learn.microsoft.com/en-us/dotnet/framework/configure-apps/file-schema/wcf/servicecertificate-of-servicecredentials)
element in attribute ```findValue```.

To preview the changes first, run following PowerShell command. 

{% include code-button.html %}
```powershell
Set-Config -WhatIf -ConfigFileName web.config -XPath "/configuration/system.serviceModel/behaviors/serviceBehaviors/behavior/serviceCredentials/serviceCertificate" -Attribute "findValue" -Value "E483FA9FFA42F000A366773DD124CE532C31BC68" 
```

The cmdlet will attempt to replace the value in the configuration. In the
screenshot, you can see a warning indicating that a missing element in the
web.config file is causing this issue. 

You can safely ignore the warning if you want to create a new XML element in the
config file.

![](/assets/pictures/editing-xml/editing-xml_whatif-illustration.png)

To actually make a change run the same code but without ```WhatIf``` directive.

{% include code-button.html %}
```powershell
Set-Config -ConfigFileName web.config -XPath "/configuration/system.serviceModel/behaviors/serviceBehaviors/behavior/serviceCredentials/serviceCertificate" -Attribute "findValue" -Value "E483FA9FFA42F000A366773DD124CE532C31BC68"
```

The cmdlet can be a little more chatty. To obtain additional information, use the ```-Verbose``` parameter.

### Example 2: Updating / setting metadata URL in application configuration section
In some cases, you may encounter identical elements that differ only in the
attribute values section. This scenario is typical for key-value application
configuration sections.

Consider following application configuration section:
```xml
<appSettings>
   <add key="LogFile" value="C:\TEMP\application.log" />
   <add key="EventLogLevel" value="3" />
   <add key="FileLogLevel" value="3" />
</appSettings>
```
To change the ```FileLogLevel``` you can use this XPath:
```xpath
/configuration/appSettings/add[@key="FileLogLevel"]
```

The entire cmdlet would than look like this:

{% include code-button.html %}
```powershell
Set-Config -ConfigFileName web.config -XPath '/configuration/appSettings/add[@key="FileLogLevel"]' -Attribute "value" -Value "4"
```

### Example 3: Silent usage
When you want to modify the config file silently, you need to specify ```-Confirm:$false``` parameter. E.g.
{% include code-button.html %}
```powershell
Set-Config -Confirm:$false -ConfigFileName web.config -XPath '/configuration/appSettings/add[@key="FileLogLevel"]' -Attribute "value" -Value "4"
```


## Few notes on changes
```Set-Config``` differs from ```Edit-Config``` only in name. 

I've changed the name to better reflect what the cmd-let does and decided to use
one of the [approved verbs for PowerShell
commands](https://learn.microsoft.com/en-us/powershell/scripting/developer/cmdlet/approved-verbs-for-windows-powershell-commands?view=powershell-7.4).

I've included alias ```Edit-Config``` for backwards compatibility.

Note to self: [naming things is
hard](https://www.martinfowler.com/bliki/TwoHardThings.html).
