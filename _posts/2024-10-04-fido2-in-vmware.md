---
layout: post
title: "using FIDO2 keys in VMWare Workstation"
date: 2024-10-03 19:00:00 +0100
image: fido2-vmware/fido2-vmware.jpg
tags: entra-id authentication mfa
---

In the past few months, I've been testing some cool FIDO2 keys. Since I do a lot
of testing and provisioning to Entra ID (Hybrid Joined, Entra Joined) and
Windows Hello for Business, I decided to use a virtual machine for these tasks.

Unfortunately, VMware does not bridge USB FIDO2 keys by default. To enable this,
you need to edit the VM's ```.vmx``` file directly. This file is usually located in
the same directory as the rest of the virtual machine's files. 

To edit this file you should:

1. Poweroff the machine and close VMWare Workstation 
2. Backup ```vmx``` file :smile: 
3. Add these lines to ```vmx``` configuration
{% include code-button.html %}
```
usb.generic.allowHID = "TRUE"
usb.generic.allowLastHID = "TRUE"
```

After you have modified the ```vmx``` file, you can open VMWare Workstation,
start the machine and connect the FIDO2 key. You'll be able to bridge it now. 

{% include gimage.html uri="/assets/pictures/fido2-vmware/fido2-vmware-connect-key.png" %}