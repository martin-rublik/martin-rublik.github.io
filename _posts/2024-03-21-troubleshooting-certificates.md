---
layout: post
title: "troubleshooting TLS and certificates"
date: 2024-03-21 08:00:00 +0100
image: ts-tls/ts-tls-logo.jpg
tags: tls x509 one-liner
---
Quite a few of my customers had issues with certificates on their web farms,
Active Directory Federation Services or any other services using TLS
protocol. This post summarizes techniques for checking certificates I tend to
use when troubleshooting TLS.

When it comes to TLS troubleshooting there are several goals you have; questions
you need to answer:
1. Is the service accessible?
2. Which certificate is presented by the service?
3. Which certificates are present in the certificate chain?

In this post I'll outline the methods and tools which I use for answering
these questions. I'll try to point out these things:
- how to deal with load-balancers and TLS server name indication (SNI),
- how to get and save remote certificates presented by a service via TLS,
- how to see the entire certificate chain on various platforms.

## Service accessibility and SNI
This part is relevant for HTTP over TLS services. For LDAP over TLS, or other
non-HTTP services, you can jump directly to the certificate enumeration part.

So, the very first step is to check whether the web page available :smiley:.
Basically we need to know if we'll get: 

- something nice, like ```HTTP 200 OK```, 
- something expected, like ```HTTP 401 Unauthorized```
- something not so nice and maybe unexpected, like ```HTTP 500 Internal Server
  Error```, or in the worst case
- you will not be able to connect because of network or certificate errors.

Performing this step might be sometimes tricky. Especially in load-balanced
environments you may need to check a specific IP address, but you may need to
request a specific hostname as well. 

### Service accessibility via PowerShell
On machines with PowerShell installed the first you might like to test something
like this: 

```powershell
Invoke-WebRequest -Uri "https://<target-ip-addr>/<target-path>" `
    -UseBasicParsing
```
In modern environments you'll probably fail, since the web server will want
to know a server name / service DNS name as well.

This is due TLS extension called [server name indication
(SNI)](https://en.wikipedia.org/wiki/Server_Name_Indication). This an extension
which is similar to ```HTTP``` ```Host``` header. 

To tell the PowerShell you want to use SNI you can use the ```Headers```
parameter: 

{% include code-button.html %}
```powershell
Invoke-WebRequest -Uri "https://<target-ip-addr>/<target-path>" `
    -UseBasicParsing `
    -Headers @{"Host"="<target-dns-name>"} 
```

### Service accessibility in *nix environments
Now imagine for a moment you don't have a PowerShell available. You'll most
probably would have ```curl```. You can use following command to check
availability of a service 

{% include code-button.html %}
```bash
curl -vkx '' "https://<target-ip-addr>/<target-path>" -H "Host: <target-dns-name>"
```

With ```curl``` you can even set specific ciphers which should be used inside
the TLS session, e.g.: 

{% include code-button.html %}
```bash
curl -vkx '' "https://<target-ip-addr>/<target-path>" -H "Host: <target-dns-name>" --ciphers TLS_AES_128_CCM_SHA256
```

The list of ciphers can be found at
[https://ciphersuite.info/cs/](https://ciphersuite.info/cs/). You'll need to use
the ```OpenSSL name``` in ```curl```. This name might be different from ```IANA
name``` in some cases.

See following screenshot for more details:

![](/assets/pictures/ts-tls/openssl-name.jpg){: width="500"}


## Certificate enumeration
As a next step you may need to enumerate the certificates which are presented by
the service
* if you can enumerate the certificate, this typically means that:
    - service is running,
    - service is correctly listening on specific port,
    - service is accessible via network.
* if you can't, this typically means the opposite:
    - service is not running, or
    - service is not correctly listening on specific network port, or
    - service is not accessible via network.

To enumerate certificate you can use either openssl tool or a powershell script. 

After enumerating the certificates focus on:
- certificate validity/expiration (especially notBefore)
- certificate chains (incorrect or missing CA certificates in chain)

### Enumerating certificates via openssl
First, you want to get the certificates presented by a service. Following
command will help you (and it uses SNI). 

{% include code-button.html %}
```bash
openssl s_client -servername <target-dns-name> -connect <target-ip-address>:<target-port> </dev/null
```

For example when enumarating certificate on www.example.org web-site use
following:
```bash
openssl s_client -servername www.example.com -connect www.example.com:443 </dev/null
```

The output of the command is illustrated by next picture.

![](/assets/pictures/ts-tls/ts-tls-openssl_sclient_01.jpg)

I've highlighted four areas within the picture.

First highlighted part relates to the certificate chain, comprising of two
certificates. 
```
Certificate chain
 0 s:C = US, ST = California, L = Los Angeles, O = Internet Corporation for Assigned Names and Numbers, CN = www.example.org
   i:C = US, O = DigiCert Inc, CN = DigiCert Global G2 TLS RSA SHA256 2020 CA1
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Jan 30 00:00:00 2024 GMT; NotAfter: Mar  1 23:59:59 2025 GMT
 1 s:C = US, O = DigiCert Inc, CN = DigiCert Global G2 TLS RSA SHA256 2020 CA1
   i:C = US, O = DigiCert Inc, OU = www.digicert.com, CN = DigiCert Global Root G2
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Mar 30 00:00:00 2021 GMT; NotAfter: Mar 29 23:59:59 2031 GMT
```
The first certificate (index 0) has subject ```s: ``` 
   
```C=US, ST=California, L=Los Angeles, O=Internet Corporation for Assigned Names
and Numbers, CN=www.example.org```. 

It was issued ```i: ``` by a CA with subject: 

```C=US, O=DigiCert Inc, CN=DigiCert Global G2 TLS RSA SHA256 2020 CA1``` 

And it is valid ```v: ``` until: 

```NotBefore: Jan 30 00:00:00 2024 GMT; NotAfter: Mar  1 23:59:59 2025 GMT```

Similarly the second certificate in the certificate chain is the issuing CA
certificate with respective ```s```, ```i``` and ```v``` fields. Make sure that
you are seeing correct certificates here. 

Second highlighted part contains the
[https://en.wikipedia.org/wiki/Base64](base64 encoded) server certificate.
```
-----BEGIN CERTIFICATE-----
MIIHbjCCBlagAwIBAgIQB1vO8waJyK3fE+Ua9K/hhzANBgkqhkiG9w0BAQsFADBZ
MQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMTMwMQYDVQQDEypE
aWdpQ2VydCBHbG9iYWwgRzIgVExTIFJTQSBTSEEyNTYgMjAyMCBDQTEwHhcNMjQw
MTMwMDAwMDAwWhcNMjUwMzAxMjM1OTU5WjCBljELMAkGA1UEBhMCVVMxEzARBgNV
BAgTCkNhbGlmb3JuaWExFDASBgNVBAcTC0xvcyBBbmdlbGVzMUIwQAYDVQQKDDlJ
bnRlcm5ldMK...
-----END CERTIFICATE-----
```
You can copy the text and save it to a file and inspect either via ```openssl```
or other tools. E.g. if you save the file as certificate.crt and open it you'll
see an standard certificate.

![](/assets/pictures/ts-tls/ts-tls-saved-certificate.jpg)

Or you can save the certificate directly with a following one-liner

{% include code-button.html %}
```bash
openssl s_client -servername www.example.com -connect www.example.com:443 </dev/null 2>/dev/null | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > certificate.pem
```

If you want to use X.509 certificates for  client authentication, the third highlighted
part is important. www.example.org currently does not support certificate
authentication thus "No client certificate CA names are sent". 
```
No client certificate CA names sent
```

Finally, ```openssl``` outputs TLS version and algorithm it used during
connection set-up.
```
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 2048 bit
Secure Renegotiation IS NOT supported
```

### Enumerating certificates with PowerShell
Enumerating certificates with PowerShell is not straight forward. You need to
create your own function for that, and still, you can't easily change
cipher-suites as these are selected by [SCHANNEL](https://learn.microsoft.com/en-us/windows/win32/secauthn/cipher-suites-in-schannel).

Following function uses native .NET Framework libraries and can be used for
certificate enumeration and testing.

{% include code-button.html %}
```powershell
function Get-RemoteCertificate
{
param(
    [Parameter(Mandatory=$true)][string]$HostName,
    [string]$ServerName=$HostName,
    [int]$Port=443,
    [System.Security.Authentication.SslProtocols] $TlsVersion = [System.Security.Authentication.SslProtocols]::Tls12,
    [string]$OutTlsCertFile
)
    $tcpsocket = New-Object Net.Sockets.TcpClient($HostName, $port)
    try
    {
        $tcpstream = $tcpsocket.GetStream()
        $sslStream = New-Object System.Net.Security.SslStream($tcpstream,$false,{param($sender, $certificate, $chain, $sslPolicyErrors)return $true})
        $sslStream.AuthenticateAsClient($ServerName,$null, $TlsVersion, $false);        
        if ($sslStream.RemoteCertificate)
        {
            $cert=New-Object system.security.cryptography.x509certificates.x509certificate2($sslStream.RemoteCertificate)
            if ($OutFile)
            {
                $content = @(
                '-----BEGIN CERTIFICATE-----'
                [System.Convert]::ToBase64String($cert.RawData, 'InsertLineBreaks')
                '-----END CERTIFICATE-----')
                $content | Out-File -FilePath $OutFile -Encoding ascii
            }
            return $cert
        }else
        {
            throw "Error connecting"
        } 
    }catch
    {
        throw $_
    }finally
    {
        $tcpsocket.Close();
    }
}
```

I like to put this function into PowerShell ```profile.ps1```. This way it is
available each time I start the PowerShell. 

If you wish you can check mine ```profile.ps1``` on 
[GitHub](https://github.com/martin-rublik/psprofile).

To use the function just use the output as a certificate object, e.g.
```powershell
$cert=Get-RemoteCertificate -HostName www.example.org
$cert | fl 
```
will output following:
```
Subject      : CN=www.example.org, O=Internet Corporation for Assigned Names and Numbers, L=Los Angeles, S=California, C=US
Issuer       : CN=DigiCert Global G2 TLS RSA SHA256 2020 CA1, O=DigiCert Inc, C=US
Thumbprint   : 4DA25A6D5EF62C5F95C7BD0A73EA3C177B36999D
FriendlyName :
NotBefore    : 30. 1. 2024 1:00:00
NotAfter     : 2. 3. 2025 0:59:59
Extensions   : {System.Security.Cryptography.Oid, ...}
```

To save the certificate to filesystem and use SNI you can use this command:
```powershell
Get-RemoteCertificate -HostName 93.184.216.34 -ServerName www.example.org -OutFile C:\temp\example.crt
```

## Certificate chain enumeration 
Within TLS, a certificate chain is used. A certificate chain is collection of
certificates from the end-entity certificate (server certificate) to the trusted
root certificate present locally on the device.

Typically the chain starts with the server certificate (as first certificate)
and ends with last intermediate CA certificate (see [Filling in the Gaps:
HTTPS/TLS
Certificates](https://blog.bityard.net/articles/2023/December/filling-in-the-gaps-httpstls-certificates))

Normally, the chain (except the root certificate) should be sent within the TLS
protocol. This provides less burden on the client side, since the client only
needs to match the root certificate in its trust store.

For Windows clients, this is more complicated. Chains are built at run-time
based on local configuration of the machine / Windows stores (see [Certificate
Chaining Engine — how it
works](https://www.sysadmins.lv/blog-en/certificate-chaining-engine-how-this-works.aspx).).

On other devices and platforms e.g. browsers not using CryptoAPI, Android/iOS
devices, etc. The correct chain which is sent by the server is extremely
important.

### Certificate chain enumeration via openssl
To save the certificate chain via openssl you can use following command. 

{%include code-button.html %}
```bash
 openssl s_client -showcerts -servername www.example.com -connect www.example.com:443 </dev/null 2>/dev/null | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > chain.pem
```

To inspect the chain you can use 
{% include code-button.html %}
```bash
openssl storeutl -noout -text -certs chain.pem
```

Or if you like unreadable one-liners as much as I do :smiling_imp:, use this one

{% include code-button.html %}
```bash
openssl s_client -showcerts -servername www.example.com -connect www.example.com:443 </dev/null 2>/dev/null | openssl storeutl -noout -text -certs /dev/stdin
```
Now I'll burn in hell :smiling_imp:.

### Certificate chain enumeration via PowerShell
To get the certificate chain locally on Windows you can use following PowerShell
commands (and cmd-let mentioned previously).

{%include code-button.html %}
```powershell
$cert=Get-RemoteCertificate -HostName sts-zpr.infra.npz.sk
# build the chain in machine context
$chain = New-Object -TypeName System.Security.Cryptography.X509Certificates.X509Chain($true)

$chainIsValid=$chain.Build($cert)
foreach($chainCert in $chain.ChainElements)
{
    $chainCert.Certificate
}
```

The output includes also root CA certificate, but besides that, it contains (or at
least should contain) pretty much the same certificates as in ```openssl```
case. 
```
Thumbprint                                Subject
----------                                -------
4DA25A6D5EF62C5F95C7BD0A73EA3C177B36999D  CN=www.example.org, O=Internet Corporation for Assigned Names and Numbers,...
1B511ABEAD59C6CE207077C0BF0E0043B1382612  CN=DigiCert Global G2 TLS RSA SHA256 2020 CA1, O=DigiCert Inc, C=US
DF3C24F9BFD666761B268073FE06D1CC8D4F82A4  CN=DigiCert Global Root G2, OU=www.digicert.com, O=DigiCert Inc, C=US
```

If that is not the case, it is definitely something you **should investigate**. It
means that local chain building took precedence and the chain is different than
what service sent.

:exclamation:BEWARE:exclamation: 

There are quite a few points where you can get different results.

First, it depends whether you build the chain in _machine_ or _user_ context. If
the chain is built in _user_ context it might be different from _machine_
context. 

This is especially the case when you will see a correct certificate chain
through GUI, but services running in local _machine_ context cannot build
certificate chain correctly. 

Intermediate CAs can be identified from certificate extension called [authority
information access (AIA)](http://www.pkiglobe.org/auth_info_access.html) during
the chain building. AIA may contain an URL for the certificate of the issuing
CA. Using this URL the client can download the issuing CA certificate from the
Internet (a.k.a. [AIA chasing](https://www.thesslstore.com/blog/aia-fetching/)). 

In the _user_ context this typically succeeds because the user has in most cases
access to the internet (directly or through the proxy). 

On the other hand, when the access to the internet is governed via proxy, the
download in _machine_ context might fail. Therefore it is good to test the chain
building in machine context.

Second, if the chain leads to an untrusted root or it cannot be verified due
lack of revocation context, you' won't get an error. Instead you can find more information
in the ```ChainStatus``` property.

For example:
{%include code-button.html %}
```powershell
$cert=Get-RemoteCertificate -HostName untrusted-root.badssl.com
# build the chain in machine context
$chain = New-Object -TypeName System.Security.Cryptography.X509Certificates.X509Chain($true)

$chainIsValid=$chain.Build($cert)
foreach($chainCert in $chain.ChainElements)
{
    $chainCert.Certificate
}

if (-not $chainIsValid)
{
    $chain.ChainStatus
}
```

Outputs following:

|Status|StatusInformation|
|:--|:--|
|PartialChain|A certificate chain could not be built to a trusted root authority.|
|RevocationStatusUnknown|The revocation function was unable to check revocation for the certificate.|
|OfflineRevocation|The revocation function was unable to check revocation because the revocation server was offline.|

And that's it. Have fun!

For the brave, that made it to the end: here is a video of [Jon Callas](https://en.wikipedia.org/wiki/Jon_Callas) perfectly summarizing the hype on [blockchain](https://www.youtube.com/watch?v=zhWCnQoRyPo). I hope someone makes a similar video on AI soon. It would be well deserved.

