---
title: "How to PowerShell (Part2)"
date: 2017-08-10
excerpt: Learn PowerShell PowerShell
tags:
- PowerShell
categories: powershell
comments: true
author_profile: true
---

# How to PowerShell (Part 2)

In the first part I covered a few basic concepts. What is Powershell, what is a cmdlet and how to find all cmdlets on your Windows Box. In this post we will dig a little deeper into the world of PowerShell. Play around with a few cmdlets and change the security settings. This is part 2 of the: _How to PowerShell_ series.

### Secure by default

PowerShell is secure by default. When you first start a PowerShell console (or the ISE for that matter) you won't be able to run scripts.  

For this example I've created a simple script.

```powershell
$Execution = Get-ExecutionPolicy
Write-Host $Execution
```

I saved this file to: ```C:\Temp\Posh\Test.ps1```

To run this script: ```.\Test.ps1```

That gives me the following output:

![Output](https://codeinblue.files.wordpress.com/2017/08/learn1.png)

It says that: _The script cannot be loaded because running scripts is disabled_. 

So, that clearly that is not working at all! 

To solve this, I have to change the execution policy. This policy actually changes a registry key. ```HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\PowerShell\1\ShellIds\Microsoft.PowerShell```

The way to change this, is through a cmdlet. ```Set-ExecutionPolicy``` There are multiple settings to choose from.

* RemoteSigned (Donwloaded scripts must be signed with a certificate, local or network scripts can run)
* AllSigned (All scripts must be signed with a certificate)
* Unrestricted (All scripts _local and downloaded_ can run)
* Restricted (No scripts will run)
* Bypass (This will bypass the current setting of the ExecutionPolicy)

My advice is to always use: ```RemoteSigned``` But for this example I will use: ```Unrestricted```. 

To see the current setting of the ExecutionPolicy, Enter: ```Get-ExecutionPolicy```

![Current](https://codeinblue.files.wordpress.com/2017/08/learn2.png)

As you can see it is set to _Restricted_. 

To change this, I have to open a new PowerShell Console. (as Administrator) You can only make changes to the ExecutionPolicy in administrator mode.

![Change](https://codeinblue.files.wordpress.com/2017/08/learn3.png)

To change the policy I enter: ```Set-ExecutionPolicy Unrestricted```

![Output](https://codeinblue.files.wordpress.com/2017/08/learn4.png)

And I have to enter ```Y``` to confirm the new setting.

In theory I can run scripts. So, lets find out!

I change the directory location to: ```C:\Temp\Posh\Test.ps1```
And enter: ```.\Test.ps1```

![Works](https://codeinblue.files.wordpress.com/2017/08/learn6.png)

Nice! that works! I can run scripts without errors. 

### The PowerShell mantra

PowerShell actually has a mantra. And it's a simple one. 

Think -> Type -> Get

![Mantra](https://codeinblue.files.wordpress.com/2017/08/learn7.png)

That's it. Remember this and soon you will rock PowerShell.

### If I could offer you only one tip for the future

No, not sunscreen! Well that too obviously! But I have another tip!

Get to know one cmdlet very, very well! 

The long term benefits of knowing one cmdlet are not only proved by scientists, but advised by everybody who uses PowerShell in her/his career.

A default Windows 10 installation comes with around 1500 PowerShell cmdlets. Then; there are the cmdlets for: Sharepoint, Office365, Azure, SCM, Intune, SQL, Hyper V and so on. 

It's impossible to know them all. And there's no need. Because every cmdlet uses the same verb-noun principle. (Or, with other words, the same mantra).

So pick an easy cmdlet. For instance: ```Get-Service``` or ```Get-Process``` play with it. Build functions around them. Make them your own. Fall in love with the magic of PowerShell. Experience the true power and always remember to just _have fun_. 

### Get-Service 

```Get-Service``` acutally gets the services running on your Windows box.

![service](https://codeinblue.files.wordpress.com/2017/08/learn8.png)

I will not go into details. Instead; let's focus on variables.

### Variables

A variable (sounds frightening) is nothing more then a storage room. Or; drawer. You can put something in that drawer. (And no; I don't mean your dirty knickers!)

You declare a variable with the: ```$``` character. 

#### you can copy the examples below in your ISE to try them out for yourself!

Example 1 - Standard variable:

```powershell
$variable1 = A
$variable2 = 'Variable'
Write-Output $variable1, $variable2
```

Example 2 - extend a variable:

```powershell
$variable3 = 'This is a v'
$variable3 += 'ariable
Write-Output $variable3
```

Example 3 - Variables in variables:

```powershell
$variable4 = 'This is a variable'
$variable5 = $variable4
Write-Output $variable5
```

Example 5 - Variables with quotes

```powershell
$variable6 = 'This is PowerShell'

'Single quote, dit is $variable6' # No variable resolving
"double quote, dit is $variable6" # variable resolving
``` 

I've shown you the ```Get-Service``` cmdlet and the basics of a variable. Now it's time to start coding yourself. You won't learn PowerShell just by reading a blog. Go away and start coding!

[Go back](https://mufana.github.io/blog)


