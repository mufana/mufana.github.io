---
title: "Automating WSUS Checks"
date: 2017-08-22
tags:
- PowerShell
categories: powershell
comments: true
author_profile: true
---

# Automating WSUS Checks

## Background

Anyone responsible for a Windows Server environment knows it is vital to keep the servers up to date. For a large part, that means; updating the servers on a regular basis. And it goes without saying that this could be done with PowerShell. But, where updating a server is one part of the job. Checking if everything still runs accordingly is another. 

### Check Norris

So, what do we check after a server has been updated. Well! in my case I check the following:

* Server still online
* All important Windows Services (Auto start) running
* Specific application Windows Services running
* Server still a member of the correct domain
* Windows Firewall running
* Specific application ports open

Imagine having to check all that manually on twenty or more servers? On that special evening you and your girl planned months in advance! 

### Happy wife - Happy life

In order to keep the wife (and life) happy and save some time, we have to do a little PowerShelling.

I always start coding in the PowerShell console. That allows me to play around a bit to see what works best.

#### Step 1 - Is this thing on

First step is to check if a server is still online. We can find out with a simple ping. But, we're not going to do that. To easy. Where is the fun in that? I need a 'ping' script. And preferably one that I can use to ping multiple servers at the same time.

Good news is that we have a cmdlet for that. ```Test-NetConnection``` 

![TestNet](https://codeinblue.files.wordpress.com/2017/08/ws1.png)

Ok, well that seems to deliver! But can I ping multiple computers with: ```Test-NetConnection``` 

![PingMore](https://codeinblue.files.wordpress.com/2017/08/ws3.png)

Apparently not. Let's check what the help file has to say about that.

```powershell
Get-Help Test-NetConnection
```

![help](https://codeinblue.files.wordpress.com/2017/08/ws2.png)

 The ```ComputerName``` switch is a string and doen not support the input of multiple ComputerNames. Otherwise it would look like this: ```<string[]>``` The square brackets indicate that the string supports multiple values.

It does however support input from the pipeline. So, If I have a list of computernames in a text file, I can pipe the content of that text file to ```Test-NetConnection```. 

![getcontent](https://codeinblue.files.wordpress.com/2017/08/ws4.png)

Nice! that works! However I don't like the output. Lots of information I don't need. I just want to know the status of: ```PingSucceeded```. 


```powershell
(Get-Content C:\temp\Servers.txt | Test-NetConnection).pingsucceeded
```

![Suc6](https://codeinblue.files.wordpress.com/2017/08/ws5.png)

That returns a ```True``` of ```False```. Which is exactly what I'm after.

#### Step 2 - Should I start or go

Next is to find out if all autostart Windows services are indeed started. 

It's important to know that PowerShell actually provides two ways to get information about Windows services.

* Get-Service
* Get-WmiObject win32_service

The difference __at least, that is how I see it__ is that ```Get-Service``` works best of you deal with one particulair service. ```Get-WmiObject``` on the other hand works well with multiple services and provides more detailed information.

I always tend to use ```Get-WmiObject```.

#### If you're new to PowerShell, play around with the ```Get-Service``` cmdlet. It's a great cmdlet to learn or enhance your scripting abilities.

To find out if all services that autostart are indeed running I use to following line of code.

```powershell
(Get-WmiObject win32_serivce -ComputerName mufana | where ({ $_.state -eq "stopped" -and $_.startmode -eq "auto"}))
```

This will give me the following output:

![Output](https://codeinblue.files.wordpress.com/2017/08/ws6.png)

### What about SQL or specific services

For SQL (both the agent and instance) and the Windows Firewall, I use a very similar line of code.

```powershell
(Get-WmiObject win32_service -ComputerName mufana | where {$_.name -like "SQLAgent$*"})
```

#### You do have to find out how your SQL agent / instance services are called

### Step 3 - Da Da Di Dom Domain

To be thorough I also want to know if a server still is a member of the correct domain.

As with ```Get-Service``` there are multiple ways to do this.

```(Get-NetConnectionProfile).name```

```(Get-WmiObject -class win32_computersystem -ComputerName localhost).domain```

Since I'm already querying wmi I decided to stick with that. However, there's no 'wrong' way to get this information.

### Step 4 - Loose ends

Now that I know what I want to query (and most importantly) how. It's time to figure out the best way to do this automatically. 

To accomplish this I'm going to place all my code in separate scripts. I named them all in a similar manner.

* chk_Firewall.ps1
* chk_AppSerivces.ps1
* chk_WinServices.ps1
* chk_Domain.ps1

Furthermore I make sure all these scripts are build the exact same way. This makes it easier to maintain them.

Let's take a look at the ```chk_WinServices.ps1``` script.

The first part in the script is a line of code that actually finds out from where on my Windows box it's running. That location is stored in a variable called ```$loc```

```powershell
# Get script location
$loc = [System.IO.Path]::GetDirectoryName($myInvocation.MyCommand.Definition)
```

Next is to dotsource a ```_Globals.ps1``` file. Dotsourcing means that everything in the ```_Globals.ps1``` file is available in this scripts. Whether that be variables or functions.

```powershell
# Get global vars
. $loc\_Globals.ps1
```

Finally something familiar! 

```powershell
$tstWinService = (Get-WmiObject win32_service -ComputerName $collectSrv | where ({ $_.state -eq "stopped" -and $_.startmode -eq "auto"}))
````

Last is to enumerate the contents of ```$tstWinService``` and write a message to a logfile.

```powershell
Foreach ( $Service in $tstWinService ) {
    
        If ($tstWinService.state -eq 'Running') {
            $Msg = "The Service: $($service.name) is running on: $($service.PSComputerName)"
            Write-Log "$Msg"
        } # end if

         If ($tstWinService.state -eq 'stopped') {
            $Msg = "The Service: $($service.name) is stopped on: $($service.PSComputerName)"
            Write-Log "$Msg"
        } # end if

} # end foreach
```

Now, in this script you might notice a few variables. For instance: ```$collectSrv``` More on that later on!

## More Background

First and foremost I want to be able to check multiple servers belonging to different application chains. For instance; A specific SQL application chain that contains:

* SQL Servers
* Application Servers

Or our Office365 chain that contains a totally different set of servers.

That means my script has to be dynamic.

Sigh. Well nobody said I have an easy job

### Jason the 13th

An other key aspect of my scripting is that is must be easy to maintain. Or; easy to add/remove application chains for someone with little or no knowledge of PowerShell. Therefore, I decided to setup a json data file that holds all relevant data. I.o. Servernames, application specific services, Domain names... and so on.

_For the following examples I use a chain called Tosca._

Creating a json file is easy.

```json
{ 
    "Tosca_Server": ["srv_001"],
    "Tosca_Domain": ["domain.com"],
    "Tosca_Service: ["Sentinal RMS"]
}
```

Pretty straight forward.

### Global warming

Remember I want my script to be as dynamic as possible. 

To achieve this I create a __global.ps1_ file. I'm going to dotsource this file in all the other scripts I'm about to create.
(more of that later on). Now, the __Globals_ file contains the following.

```powershell
$Date = (Get-Date -Format d).Replace("-","")

# The JSON container where all server names, service names are stored.
$arr = Get-Content "C:\Temp\Check\data.json" | Out-String | ConvertFrom-Json

# Location of the logpath
$Logpath = "C:\Temp\Check\Log\"

# Function that enables logging
Function Write-Log {

    Param (
        [Parameter(ValueFromPipeLine=$True)]
        [string]$Logstring)

    $time = get-date -Format T
    $logstring  = $time +" "+ $Logstring

    Add-Content $Logfile -Value $Logstring
}
```

Lots of things are going on here! Most important is the json file.

#### 1 - Import JSON file

I have to import to contents of the json file in a variable called ```$arr```.

#### 2 - A custom Write-Log function

I have added a ```Write-Log``` function to enable custom logging. 

### Ise Ise Baby

Ok, my plan is beginning to unfold. It's time to write the final script.
Check-Server.ps1.

```powershell
$loc = [System.IO.Path]::GetDirectoryName($myInvocation.MyCommand.Definition)

. $loc\_Globals.ps1`

Function Check-Server {

    [cmdletbinding()]

    Param ([String]$environment)

# Name and location of the logfile
$Logfile = $Logpath + $Date + "-WSUS-" + "$environment"  + ".log"

$Date1 = (Get-Date -format g)
If(-not($Logpath | Test-Path)){New-Item $Logpath -type directory}
New-Item $Logfile -type file -force -value "[$Date1] WSUS Check for: $environment" 
Add-Content $Logfile -Value ""

<#
The following vars need be set:

$CollectsrvAp   = The server where the application specific services are running on. 
$collectdom     = The domain where the server(s) are member off.
$collectSrv     = All servers needed for this application.
$collectService = The services that must be running for the application to run.

These vars are used in every script.
#>

$status_text = switch ($environment) {

        Tosca { 
            $CollectsrvAp = ($arr).Tosca_server
            $collectSrv = ($arr).Tosca_server
            $collectdom = ($arr).Tosca_domain
            $collectService = ($arr).Tosca_service
            Write-Log "Checking if server(s): $collectsrv are member of domain: $collectdom"
            Invoke-Expression C:\Temp\CheckPC2\chk-Domain.ps1
            Write-Log ("#"*80)
            Write-Log "Checking Tosca Services"
            Invoke-Expression $loc\chk-AppServices.ps1
            Write-Log ("#"*80)
            Write-Log "Checking firewall state on: $Collectsrv"
            Invoke-Expression C:\Temp\CheckPC2\chk-Firewall.ps1
            Write-Log ("#"*80)
            Write-Log "Checking Windows Services on: $collectSrv"
            Invoke-Expression C:\Temp\CheckPC2\chk-WinServices.ps1
            Write-Log ("#"*80)
            Add-Content $logfile -value "[$Date1] Completed all WSUS checks for: $environment"
            #SendTo-eMail
        } # End tosca
    } # end switch
} # end script
```

So, what is going on hee!? well, let me explain!

#### 1 - Location of script and dotsourcing the _globals.ps1

No explanation needed.

#### 2 - A parameter called ```$environment```

That corresponds with a switch statement. Remember I've created a json file and imported the contents in the __Globals.ps1_ file. I want to be able to check multiple servers / application chains. So, for every chain I will create the values in the json file. For instance; Tosca. 

In my script, I have created a switch statement called ```Tosca```. Within that statement I have set a few variables. For instance; the ```$CollectSrv``` variable. (Remember that one from the ```chk_WinServices.ps1``` script!?) 

```powershell
$CollectsrvAp = ($arr).Tosca_server
$collectSrv = ($arr).Tosca_server
$collectdom = ($arr).Tosca_domain
$collectService = ($arr).Tosca_service
```

The ```CollectSrv``` variable is set to a value of ```($arr).Tosca_server```. That's the servername for the Tosca application chain in the json file. Is that cool or what?

All I have to do to now is run the script. 

![run](https://codeinblue.files.wordpress.com/2017/08/ws7.png)

That will generate a nice logfile.

```log
[22-8-2017 22:39] WSUS Check for: tosca
22:39:22 Checking if server(s): srv_001 are member of domain: domain.com
22:39:23 Server srv_001 is member of the correct domain
22:39:23 ################################################################################
22:39:23 Checking Tosca Services
22:39:23 The Service: Sentinel RMS License Manager is running on: srv_001
22:39:23 ################################################################################
22:39:23 Checking firewall state on: srv_001
22:39:24 The firewall is running on: srv_001
22:39:24 ################################################################################
22:39:24 Checking Windows Services on: srv_001
22:39:24 The Service: RemoteRegistry is stopped on: srv_001
22:39:24 The Service: sppsvc is stopped on: srv_001
22:39:24 ################################################################################
[22-8-2017 22:39] Completed all WSUS checks for: tosca
```

## Wrapping things up

Looking back on my original goals, I think I've done pretty well. The script(s) are dynamic, it's easy to add more servers/environments. However, they are somewhat complex. Dotsourcing a __Globals_ file may not be the most ideal way. Last, I'm not to happy with my _switch_ statement so I will definitely make some improvements.

Good news is that I have saved myself lots of time. Next time I have to check the servers after WSUS updates, I will fire-up my scripts, sit back and relax.

Happy wife, Happy life!

[Go back](https://mufana.github.io/blog)


















