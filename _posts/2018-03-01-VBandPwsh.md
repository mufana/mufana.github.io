---
title: "VB, DS, Ldifde and pwsh"
date: 2018-03-01
---


# VBscript, DSquery, LDIFde and PowerShell

It's almost impossible to imagine but, there was a time we 'admins' had to do our job without PowerShell. 

In the early days of IT, if you wanted to extract, change or add information from/to Active Directory, we had to create the most dreadful _I'm having trouble writing this!_ __VBScripts__, download tools with awful GUI's, use: __dsquery__ or __LDIFde__.

If there's one thing all these tools have in common, it's the fact that they're all complex. And you _(or at least I when I wrote the 'code')_ had no idea if it was going to work. Most of the times, I had to try number of times before I got something useful. Or; worst case, you screw-up your environment.

## DSquery (The Enigma Machine)

Take a look at _dsquery_ to extract the: ```sAMAcountName``` of all members in a specifc group.

```cmd
for /F "delims=*" %i IN ('dsquery * "ou=groups,ou=corp,dc=contoso,dc=com" -filter "(&(objectClass=group)(name=*All_Employees*))" -l -d contoso.com -attr member') DO @dsquery * "ou=users,ou=corp,dc=contoso,dc=com" -filter "(distinguishedName=%i)" -attr sAMAcountName
```
![CMD](https://codeinblue.files.wordpress.com/2018/03/1.png)

Really? I'm gobsmacked. I mean, c'mon! ```for /F "delims*" %I``` is this writen in Dothraki? is it a secret enigma code? I have no idea.

Okay! for someone __familiar with coding__ it's pretty obvious. But, for people who aren't familiar with coding, it makes no sense at all.

## LDIFde

So, dsquery is complex. What about LDIFde? Well, to export the: _sAMAcountName_ of all userobjects within the: _ou=users,ou=corp_ organizational unit, We had to type something like:

```cmd
ldifde -s contoso.com -f c:\temp\export.ldf -d "ou=users,ou=corp,dc=contoso,dc=com" -r "(objectclass=User)" -l sAMAcountName
```

![Export](https://codeinblue.files.wordpress.com/2018/03/4.png) 

The export generates an ```*.LDF``` file. Which basically is just a text file. You can open and edit LDF files with notepad. The output of the LDF file is as follows:

![LDF](https://codeinblue.files.wordpress.com/2018/03/5.png) 

```ldf
dn: CN=092,OU=Users,OU=Corp,DC=contoso,DC=com
changetype: add
cn: 092

dn: CN=093,OU=Users,OU=Corp,DC=contoso,DC=com
changetype: add
cn: 093
```

Owkay! well, LDIFde is useless. 

## Those were the days

Now a days we can do all this with a simple PowerShell cmdlet. 

```powershell
Get-ADGroupMember -Identity 'all_employees' | select sAMAcountName
```

![PWSH](https://codeinblue.files.wordpress.com/2018/03/2.png) 

Gobsmacked. Again.
This is understandable. Even for someone with no knowledge of PowerShell. Heck! even old Nan can understand this one. And it took me just 5 seconds to type and get the information I want.  

## Make a change

But what if you had to change lots of records within Active Directory?
Again you could use: _VB*@&*A#@, nope, not writing it again!_ A third party tool or LDIFde.

### LDAP

So, I think everyone knows something about LDAP right!? or at least, at some point in your career you might have heard about it.

Let's be clear about one thing! __LDAP is not Active Directory__. Both LDAP and Active Directory are different things all together. However, Active Directory is based on LDAP. Which means that you can use LDAP query's to gather information from Active Directory. 

### Classes

The Active Directory contains lots of objects. _User,Computer,Organizationl Unit, Server, Group_ etc... Every object uses a different set of attributes. For instance; a userobject has the attributes: ```CN```, ```FirstName```, ```LastName``` and so on. 

These attributes are defined in a class. A userobjects's class is: ```user```. 

A _class_ is like a template or blueprint with a set of rules to define how a particuliar object looks and which attributes are mandatory. This information is stored in the Active Directory Schema.

### Browse

Take a look at a userobject with an LDAP browser...
For this example I've used: [Softerra LdapBrowser](http://www.ldapbrowser.com/)

![Browser](https://codeinblue.files.wordpress.com/2018/03/3.png) 

As you can see in the image above, the objectclass has a value of: ```user```. The: ```cn``` with a value of: ```Moses``` is an attribute. 

## B2Ldifde

Remember my exported file with LDIFde? 

LDIF is actually the same information as you see in the LDAPbrowser. Represented in plaintext. 

If I had exported all the attributes with LDIFde, the output would look like this:

```ldf
dn: CN=092,OU=Users,OU=Corp,DC=contoso,DC=com
changetype: add
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: user
cn: 092
sn: Moses
description: Automatic Import
givenName: Frank
distinguishedName: CN=092,OU=Users,OU=Corp,DC=contoso,DC=com
instanceType: 4
whenCreated: 20171117102419.0Z
whenChanged: 20180302130840.0Z
displayName: 092
uSNCreated: 12798
memberOf: CN=All_Employees,OU=Groups,OU=Corp,DC=contoso,DC=com
uSNChanged: 57398
name: 092
objectGUID:: k6eCWmOp/UmhoArs7c7ufQ==
userAccountControl: 66050
badPwdCount: 0
codePage: 0
countryCode: 0
badPasswordTime: 0
lastLogoff: 0
lastLogon: 0
pwdLastSet: 131553878594042640
primaryGroupID: 513
objectSid:: AQUAAAAAAAUVAAAACSxXIoAiWp4NDVt1UAQAAA==
accountExpires: 9223372036854775807
logonCount: 0
sAMAcountName: 092
sAMAccountType: 805306368
userPrincipalName: Frank.Moses@Contoso.com
objectCategory: CN=Person,CN=Schema,CN=Configuration,DC=contoso,DC=com
dSCorePropagationData: 20171202162037.0Z
dSCorePropagationData: 16010101000001.0Z
```

Each of these attributes can be modified with LDIFde.  

Let's take a look at my original LDF file:

```ldf
dn: CN=092,OU=Users,OU=Corp,DC=contoso,DC=com
changetype: add
cn: 092
```

One of the first things I notice is the: __changetype: add__ 

Whenever I import this LDF file, a new user is created. (only to generate an error since the object already exists!)

Now, what if I don't want to add anything but instead, modify the ```sAMAcountName``` of an existing object!? 

First is to modify the LDF file to meet the 'modify' requirements. 

```cmd
dn: CN=092,OU=Users,OU=Corp,DC=contoso,DC=com

changetype: modify              ## Change and existing object
replace: sAMAcountName          ## Replace the sAMAcountName
sAMAcountName: 092_LDIFDE       ## Replace the value with: 092_LDIFDE
-                               ## Marks the end of the record
```

![Change](https://codeinblue.files.wordpress.com/2018/03/6.png) 

Now that the file is modified. It's time to import it in Active Directory.

```cmd
ldifde -i -f c:\Temp\export.ldf
```

![Import](https://codeinblue.files.wordpress.com/2018/03/7.png) 

And to verify if the sAMAcountName is indeed changed:

![Verify](https://codeinblue.files.wordpress.com/2018/03/8.png) 

It's clear that LDIFde works perfectly. It does the job and handles it pretty good. Only downside, the process is quite slow and requires a lot of typing and editing. Reasonable chance of making typo's.

## PWSH

 The PowerShell cmdlet to do the exact same thing:

```powershell
Set-ADUser -Identity "092_LDIFDE" -sAMAcountName "092"
```

![Change_PWSH](https://codeinblue.files.wordpress.com/2018/03/9.png) 

Same result. Less typing. 

![Verify](https://codeinblue.files.wordpress.com/2018/03/10.png)

## Wrapping up

Now a days; PowerShell is __the__ tool most sysadmins use _(should use)_ to get work done. And yes! also with PowerShell and coding, there's always a chance of making typo's. However, in contrary to LDIFde, VBscript or whatever, with PowerShell there's always a safety option. 
```-WhatIf```. 

[Go back](https://mufana.github.io/blog)