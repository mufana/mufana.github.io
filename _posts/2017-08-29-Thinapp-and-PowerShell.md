---
title: "Thinapp and PowerShell (Part 1)"
date: 2017-08-29
tags:
- PowerShell
categories: powershell
comments: true
author_profile: true
---

# Thinapp and PowerShell (Part 1)

## Background

One of the tasks in my job as a system engineer is to package / virtualize applications. And, with application updates coming out on a monthly base, this also involves updating those applications. And, I'm still doing this with VMware Thinapp. But sometimes that's a slow process, not intuitive and requires specific knowledge. E.g. Thinapp, Windows, concepts of virtualization.

I'm always looking for ways to make my work a little less annoying and 'in that process' educate my colleague's. Ok, truth be told, that pretty much means, assigning that task to a colleague. But, not before I made it easier for them and more fun for me. 

## PowerShell ![love](https://maryrefugeofholylove.files.wordpress.com/2017/02/heart-icon.gif) Thinapp

PowerShell and Thinapp are not a marriage made in heaven. It's more like getting trapped on a desserted hellish Island trying to catch fish with a bow and arrow. Or is it?

### SDK

Because there actually is an SDK available for Thinapp. And although it's meant for use with (I almost can't type this...) VBscript (disgusting) and C#, it could also be used with PowerShell. 

#### Installing the SDK

Well, we can't install anything without downloading it first. So, [Click here to download the SDK](
https://code.vmware.com/web/sdk/5.1.1/thinapp).

Installing the SDK in easy. Just unblock the files in the zip, save them to a sensible location and register the ```ThinAppSDK.dll``` In this example I saved them to ```C:\Temp\SDK```

![installsdk](https://codeinblue.files.wordpress.com/2017/08/12.png)

To register the ```*.dll```

Run a PowerShell Console or cmdprompt as ```Administrator```.

![regsdk](https://codeinblue.files.wordpress.com/2017/08/22.png)

That's it for the installation. 

#### Leveraging the SDK with PowerShell

To leverage the SDK with PowerShell:

1. Open a new PowerShell ISE or Console.

2. Create a connection to the ThinApp SDK COM object.

```powershell
$Thinapp = New-Object -ComObject ThinApp.Management
```

![regsdk](https://codeinblue.files.wordpress.com/2017/08/31.png)

So, I have my connection to the SDK. But what can we do with it? Let's find out by typing: ```$Thinapp | gm``` 

This will show us all properties and methods of the object. In this case, the ThinApp COM management object.

![regsdk](https://codeinblue.files.wordpress.com/2017/08/31.png)

In this case there are three.

* GetThinAppType
* OpenPackage
* RefreshDesktop

_OpenPackage_ seems like an interesting one. Let's try it wih my _Softerra LDAP Browser_ thinapp.

![regsdk](https://codeinblue.files.wordpress.com/2017/08/41.png)

Ok, So that shows me the:

* inventoryName
* MainDataContainer
* MSIVersion
* ThinAppType
* ThinAppVersion

This is all very cool. But let's dig al little deeper. What can we do with a Thinapp package itself?

Again, it's time to call ```GM```.

![regsdk](https://codeinblue.files.wordpress.com/2017/08/51.png)

Nice! First thing I noticed is ```GetOptions```. That retrieves the contents of the ```package.ini```

![regsdk](https://codeinblue.files.wordpress.com/2017/08/71.png)

There are lots cases where this tool could prove it's worth using. Like; forcing an AppSync update. Or, check whether a specific file exists in the package.
I will do another separate post about that.

So, while I was alone, trying to catch fish with a bow and arrow, Just a few feet away from me there was a party boat with tons of good looking bikini girls, loads of fruit,drinks and food. Heaven on earth!

### On that Bombshell

We're not there yet! True, the SDK is very nice. I personally think it's quite cool. Like, level 8 cool on a scale from 0-10. But, c'mon! we can do better.

Imagine building a Thinapp with PowerShell? Or, imagine having a few colleague's who aren't familiar with Thinapp but occasionally need to update a Thinapp. How do we provide them with the tools to do just that? 

### And the correct answer is

Well there isn't any. Though, there's no wrong answer either. It's simply a matter of preference. In my case; two.

* Leverage the build with PowerShell on a PowerShell console.
* Leverage the build with a GUI.

I'm not a fan of GUI's. But, I love PowerShell Studio. I bought it a few years ago and I still use it today. And I have to admit (though reluctantly) I do like to create GUI's. Nothing beats the feeling I have when creating a few GUI's before breakfast. 

### Trigger fast

When it comes to building GUI's there's one very important thing to remember. 

_**Actions trigger events.**_

All actions are initiated through user input. Whether that be a mouseclick, press of a button, or whatever.

Why this is so important? 
Well, I want my GUI to be responsive. I **hate** GUI's freezing up. I like to move my GUI around a bit while I'm waiting for something. 

Imagine having a GUI app with a button and a textbox. Press that button and all processes running on your Windows box will be displayed in the textbox. Because the GUI acts like a Console underneath, you can't give any input until the script has finished. If you were to run the ```Get-Process``` command on the console, you can't do anything until the command finished. 

The way to work around this is by using:

* Jobs
* Runspaces

Jobs are exactly that. There are four types of jobs in PowerShell.

* Background Jobs
* Scheduled Jobs
* WMI Jobs
* Remote Jobs

A runspace is something else entirely. Runspaces are actually separate instances of a PowerShell Console. Think of a runspace as a new _PowerShell Tab_ in the ISE.  

Creating new runspaces are however somewhat complicated and require us to take a plunge into .NET.  

The scripting guys have an awesome blog series about it. So, be sure to [check it out.](https://blogs.technet.microsoft.com/heyscriptingguy/2015/11/26/beginning-use-of-powershell-runspaces-part-1/)

### Fun time

But before I'm going to design a GUI, I need to write my PowerShell code. In this case I need to write a script that updates a Thinapp. 

Updating a Thinapp is process that requires multiple steps. 

1. Clean the sandbox / thinstall directory in profile directory.

2. Create a backup of the Thinapp build capture. 

3. Download the update for the application and store in the Build capture. **

4. Rebuild the Thinapp (with the downloaded update).

5. Start the application form the build capture. 

6. Update the application. This will actually update the sandbox in the profile directory.

7. Use SBmerge to merge the contents of the sandbox and the build capture.

#### The Code (or guidelines)

Steps 1-2-3-4-5-7 can be initiated from PowerShell so I'm going  to write a little bit of Powershell code for these steps.

_**Always start building/rebuilding Thinapps on a freshly installed machine!**_

Having a freshly installed machine also means that VMware Thinapp is not installed. Instead, I start it from a directory on the network.

### The first line of code

```powershell
$Loc = $PSScriptRoot
```

```PSScriptroot``` is the location of the script currently executed. It is a built-in variable.

##### Empty by default
It's important to know that this variable is empty by default. It's only gets a value when a script is actually running. As soon as the script has finished, the variable will be set to empty.

##### Only in PowerShell v3 and higher

Another import thing to remember is that this variable is only available in PowerShell version 3 and higher. 

##### The PowerShell 2 way:

```powershell
$Loc = [System.IO.Path]::GetDirectoryName($myInvocation.MyCommand.Definition)
```

##### Example

1. Create a script called ```Pathtest.ps1```.

2. Save it to ```C:\Temp```


```powershell
$Loc = [System.IO.Path]::GetDirectoryName($myInvocation.MyCommand.Definition) 
Write-Host '$loc is' $Loc
Write-host 'PSScriptRoot is' $PSScriptRoot
```

3. Run the script.

![Result](https://codeinblue.files.wordpress.com/2017/08/21.png)

Both return a value of: ```C:\Temp```

### Second line of code

Since the installation of my VMware Thinapp software resides on the network; I have to add an environmental variable. ``` Thinstall_Bin``` This is the location the ```build.bat``` needs in order to build the Thinapp.

```powershell
[Environment]::SetEnvironmentVariable("THINSTALL_BIN", "$Loc\VMware", "User")
```

Remember: ```$Loc``` is the location of the script I'm currently writing. ```\vmware``` is the directory of my VMware Thinapp software.

### Function: Clean the sandbox

This merely a safety measure. In case I already have a Thinstall sandbox in my profile, I want it to be removed. This prevents crap from getting in the build. 

```powershell
Function clean-Thinstall {

param (
    [string]$Application
    )

$User = $env:USERNAME
$AppTst = (Test-Path "C:\Users\$User\appdata\roaming\thinstall\$Appliction")

If ($AppTst -eq $true) {
    Remove-Item -Recurse "C:\Users\$User\appdata\roaming\thinstall\$Appliction" -force
    Write-Host "Thinstall Directory $Application is deleted...`n"}
    
Else {
    Write-Host "Thinstall Directory $Application does not exist in the Roaming Profile...`n"
    }
}
```

#### Example

I have an Thinapp application called: ```Application1''' in my profile folder. I want to remove that particulair thinstall directory.

![App1](https://codeinblue.files.wordpress.com/2017/08/11.png)

Simply type ```Clean-Thinstall -application Application1```. And all your problems will be gone.

### Function: Backup-Thinapp

Since I'm updating an existing capture, do want to create a backup. Just in case something goes wrong.

For this I always use Robocopy. It's pretty straigtforward and a reliable tool to copy data.

```powershell
Function Backup-Thinapp {

$SrcFolder = "$loc\Capture"
$DstFlder= "$loc\_Backup\$Date"
$Logfile = "$loc\_Logging\$Date-BackupCapture.log"
$Proces = "Robocopy"
$Args = "$SrcFolder $DstFlder /MIR /X /V /r:0 /Tee /Log:$Logfile"

New-Item -ItemType Directory -path $DstFlder

Start-Process $Proces $Args -Wait

}
```

No rocketscience here!

### Function: Dowload-Update

There are three ways to download files with PowerShell. All three have there pros and cons. 

* Bits transfer
* .NET (System.Net.Webclient)
* WebRequest

I don't which way is best. Of fastest. In this case I'll go with ```webrequest```. 

```powershell
$url = "URL to download\file.exe"
$output = "$PSScriptRoot\Capture\Downloads\File.exe"

Invoke-WebRequest -Uri $url -OutFile $output
```

Also, pretty straightforward and best, a small footprint of code.

### Function: Build-Thinapp

```powershell
Function Build-Thinapp {

$Runspace = [runspacefactory]::CreateRunspace()
$PowerShell = [powershell]::Create()
$PowerShell.runspace = $Runspace
$Runspace.Open()

# Add the $loc var to the runspace
$Runspace.SessionStateProxy.SetVariable('loc',$Loc)

    [void]$PowerShell.AddScript({

        Function init2 { 

            $Build = "$loc\Capture\build.bat"
            $Logfile = "$loc\_Logging\$Date-Build.log"

            Try { 
                Start-Process -wait $build -ErrorAction stop -ErrorVariable currenterror 
       
            } Catch { 
                Write-ConsoleWarning "Error detected. See Logfile for information" $currenterror |out-file $LogFile -append }
        }

        init2

    })

    $AsyncObject = $PowerShell.BeginInvoke()

}
```

This is where things are getting a little complicated. Because here I have created a runspace for the _Build-Thinapp_ function to run in. This will guarantee my GUI stays responsive.  

__*Remember*__ I created the $Loc variable as first line of code in my script. This is the location from where the script will be executed. When creating a new runspace and thus, creating a new instance of a PowerShell Console, the variable ```$Loc``` is not available. Makes sense. The way to solve this is parse the variable ```$loc``` with the new runspace. The string is as follows:

```powershell
$Runspace.SessionStateProxy.SetVariable('loc',$Loc)
```

### Function: Merge-Thinapp

Remember we are updating an existing Thinapp. That means, updating the sandbox and merging it's contents with the Thinapp capture. The way to do is is by using ```SBmerge.```

The Function is the almost same is with ```Build-Thinapp```.

```powershell
Function Merge-Thinapp {

    $Runspace = [runspacefactory]::CreateRunspace()
    $PowerShell = [powershell]::Create()
    $PowerShell.runspace = $Runspace
    $Runspace.Open()

    # Add the $loc var to the runspace
    $Runspace.SessionStateProxy.SetVariable('loc',$Loc)

    [void]$PowerShell.AddScript({

        Function init3 { 

            $Capture = "$loc\Capture"
            $Default = "cd .."
	        $Date = Get-Date
            $Date = $Date.ToString("dd-MM-yy")
	        $Exe = "$loc\VMware\SBmerge.exe"
	        $Args = "Apply"

	        Set-Location $Capture
	        Start-Process $Exe -Args $Args
	        $Default 
        }

        init3

    })

    $AsyncObject = $PowerShell.BeginInvoke()

}
```

### Funtion: Start-Thinapp

Well, that an easy one.

```powershell
Start-Process "PathToExecutable"
```

#### Building a GUI

Now that I have al my functions, it's time to build to GUI.

This is not a step-by-step instruction. There's a separate blogpost about PowerShell GUI's in the making. So make sure to come back!

For this example I've created a simple but fancy GUI. 

![GUI](https://codeinblue.files.wordpress.com/2017/09/10.png)

Now, all the settings that are required to build a Thinapp are controlled by an XML config file. I made sure that all the settings could be changed through the GUI.

![Settings](https://codeinblue.files.wordpress.com/2017/09/11.png)

#### Is this thing one

Now, Testing time...

I will test my new form with a simple build of an application. 

![TestBuild](https://codeinblue.files.wordpress.com/2017/09/test2.gif)

As you can see in the gif above, the GUI freezes during the build. It displays ```Not responding``` in the title bar. When the build is finished, I'm free to use the GUI and the output is displayed. Cleary that doesn't work the way I want it to.

To solve this I have to make a few adjustments in the function ```Build-Thinapp```. 

Original my ```Build-Thinapp``` function uses the default ```build.bat``` you can find in the application capture. But I don't need the *.bat file. We have PowerShell and anything you can do with ```bat``` you can do better with PowerShell.

In the ```build.bat``` you will find a few lines looking similair to this:

```bat
"%THINSTALL_BIN%\vregtool" "%TARGET_DIR%\Package.ro.tvr" ImportDir "%PROJECT_DIR%"
IF ERRORLEVEL 1 GOTO failed

"%THINSTALL_BIN%\vftool" "%TARGET_DIR%\Package.ro.tvr" ImportDir "%PROJECT_DIR%"
IF ERRORLEVEL 1 GOTO failed

"%THINSTALL_BIN%\tlink" "%PROJECT_DIR%\Package.ini" -OutDir "%TARGET_DIR%"
IF ERRORLEVEL 1 GOTO failed
```

And that can be transformed to a PowerShell function in under 2 minutes.

```powershell
Function Build-Thinapp {

    [cmdletbinding()]

    Param (
        [string]$ThinappDir = "C:\Temp\SDK\_Thinapp\VMware\",
        [string]$Project = "C:\Temp\SDK\_Thinapp\Capture\",
        [string]$OutputBin = "C:\Temp\SDK\_Thinapp\Capture\bin"
    )

$THINSTALL_BIN = $ThinappDir
$PROJECT_PATH = $Project
$TARGET_DIR = $OutputBin
$PROJECT_DIR = $PROJECT_PATH

.$THINSTALL_BIN\vregtool.exe $TARGET_DIR\Package.ro.tvr ImportDir $PROJECT_DIR
.$THINSTALL_BIN\vftool.exe $TARGET_DIR\Package.ro.tvr ImportDir $PROJECT_DIR
.$THINSTALL_BIN\tlink.exe $PROJECT_DIR\Package.ini -OutDir $TARGET_DIR
}
```
 
To stop my GUI from freezing, I'm going to create a job.

```powershell
start-job -name build -ScriptBlock { commandhere }
````

To get the status and output of a job simply type: 

```powershell
Get-Job -name build | receive-job -keep
```

My new function is as follows:

```powershell
start-job -name build -ScriptBlock {

    Function Build-Thinapp {

        [cmdletbinding()]

        Param (
            [string]$ThinappDir = "C:\Temp\SDK\_Thinapp\VMware\",
            [string]$Project = "C:\Temp\SDK\_Thinapp\Capture\",
            [string]$OutputBin = "C:\Temp\SDK\_Thinapp\Capture\bin"
        )


    $THINSTALL_BIN = $ThinappDir
    $PROJECT_PATH = $Project
    $TARGET_DIR = $OutputBin
    $PROJECT_DIR = $PROJECT_PATH

    .$THINSTALL_BIN\vregtool.exe $TARGET_DIR\Package.ro.tvr ImportDir $PROJECT_DIR
    .$THINSTALL_BIN\vftool.exe $TARGET_DIR\Package.ro.tvr ImportDir $PROJECT_DIR
    .$THINSTALL_BIN\tlink.exe $PROJECT_DIR\Package.ini -OutDir $TARGET_DIR

    }

    Build-Thinapp
}
```

I placed my function ```Build-Thinapp``` inside the scriptblock I'm sending with the ```Start-Job``` cmdlet.

![manually](https://codeinblue.files.wordpress.com/2017/09/test3.gif)

It works! But I still have to manually check for the job status. 
To solve this I've created a simple do/while loop. 

```powershell
$job = Get-job -name build
while($job.state -eq 'running') 
{
    start-sleep -Seconds 1
    receive-job -name build -Keep
    }
```

![automatic](https://codeinblue.files.wordpress.com/2017/09/test7.gif)

To do this manual in the GUI I've added a ```Get-Status``` button. That buttons runs the ```Get-Job``` cmdlet.

![ManualGUI](https://codeinblue.files.wordpress.com/2017/09/test8.gif)

It's nice but I still have to click a button to get the status and output of the job. And since I'm lazy that needs to be automated.

![AutoGUI](https://codeinblue.files.wordpress.com/2017/09/test9.gif)

## Wrapping Up

This GUI is still far from complete. It's just an example to show you the possibilities when it comes to PowerShell and VMware Thinapp.

In the next part of this series I will add more functions to the GUI and really make it a usefull tool.

The logo's used in this GUI from are created with the help of [Free Logo Design](https://www.freelogodesign.org/). 

[Go back](https://mufana.github.io/blog)