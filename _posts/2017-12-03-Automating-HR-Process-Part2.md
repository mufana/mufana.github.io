---
title: "Automating an HR process (Part 2)"
date: 2017-12-03
---

# Automating an HR process (Part 2)

Remember my previous post!

This was supposed to be part 2 of this blog series. But instead it turned out to be...

## Something completely different

A few days ago, I woke up in the middle of the night and pretty much decided to throw away the previous code and start over from scratch. The reason; well, to be honest, I didn't really like what I had built. It was complex, it lacked structure, it was a disaster. Okay, that's a little exaggerated but still...

## What I want to achieve (with automation)

__User accounts are automatically provisioned in the Active Directory by an IDM proces managed externally.__

* Create a home folder for the new user accounts in: ```\\contoso\Home```

* Add standard groups to any Active Directory user object within the ```users.contoso.com``` organizational unit

* Set folder permissions for the home folder

* Add an eMail address to any Active Directory user object within the ```users.contoso.com``` organizational unit

* Add a standard Office365 license for each user

## Jenkins

The first change I made was; __Jenkins.__ Jenkins is a true automation server. Mostly used in a _Continuous Integration / Testing and Deployment_ process. Cool, but I'm not developing software. Luckily Jenkins also comes with a PowerShell plugin.

There are more reasons to use Jenkins:

* Everybody _able to login to the Jenkins server_ can execute my scripts
* Support for Source Control. (Git, SVC)
* Include Pester tests and execute them from the same platform
* Support for _Slack_ integration to keep things transparent

## Setting up the Jenkins Server

Well, we can't setup anything without software so, click on your favorite browser and donwload the appropriate version for your environment from: <https://jenkins.io/download/>

_For this demo I'll be using the Windows version!_

1; Unzip the ```jenkins-version.zip``` file to a sensible location.

2; Run the Jenkins installer.

3; When the installion is completed, your browser will start automatically.

You will have to unlock Jenkins  with the pasword located in: ```C:\Program Files (x86)\Jenkins\secrets\InitialAdminPassword``` file. Open the file with Notepad and copy the string in the: ```Administrator Password``` field.

![Unlock](https://codeinblue.files.wordpress.com/2017/12/1-unlock.png)

Next you'll need to configure the Jenkins plugins. You can select all the plugins you want or go with the default selection.

4; Click on: ```Install Suggested Plugins``` or ```Select plugins to install```.

![plugins](https://codeinblue.files.wordpress.com/2017/12/2-pluginbasic.png)

5; The installation of plugins might take some time. So be patient.

![InstallPlugin](https://codeinblue.files.wordpress.com/2017/12/4-installing.png)

6; When the installation is finished, you will be asked to create the first Admin User.

![CreateAdmin](https://codeinblue.files.wordpress.com/2017/12/5-firstadmin.png)

7; Create the first admin user or continue as default Admin. (You'll need the password from: __step 3__).

![admin](https://codeinblue.files.wordpress.com/2017/12/6-continueasadmin.png)

8; That's it for the installation and basic setup of Jenkins.

![Welcome](https://codeinblue.files.wordpress.com/2017/12/7-welcome.png)

## Install plugings

Next we'll need to install a few plugins.

1; Go to: ```Manage Jenkins``` -> ```Manage Plugins```.

![ManagePlugin](https://codeinblue.files.wordpress.com/2017/12/1-manage-plugins.png)

2; Click on the: ```Available``` tab to show all available plugins.

![available](https://codeinblue.files.wordpress.com/2017/12/2-select-plugins.png)

3; In the: ```Filter box``` type: ```PowerShell``` and click the checkbox next to the PowerShell plugin.

4; In the: ```Filter box``` type: ```Environment Injector Plugin``` and click the checkbox.

5; When the plugins are installed, click the: ```Restart Jenkins when...``` checkbox to restart Jenkins.

![finishedinstall](https://codeinblue.files.wordpress.com/2017/12/3-install.png)

## A 64bit PowerShell instance

Default, the Jenkins server runs in 32bit mode. That also means that PowerShell scripts are executed from the 32bit PowerShell instance. Most PowerShell modules (including the Azure and MSonline modules are 64bit only), so they won't work with the 32bit instance. The solution is to run Jenkins in 64bit mode. __The 32bit Azure modules are deprecated__.

To change Jenkins from 32bit to 64bit:

1; Install Java JRE 64bit.

2; Edit the: ```C:\Program Files (x86)\Jenkins\Jenkins.xml``` file. 

Change the following line from:

```xml
<executable>%BASE%\jre\bin\java</executable>
```

To:

```xml
<executable>C:\Program Files\Oracle\javaVersion\Bin\Java</executable>
```

__You can find it on line 40__

Change _javaVersion_ to the version installed on your system.

3; Go to your Jenkins webpage. If installed locally: http://localhost:8080/

4; Change the url to: http://localhost:8080/restart

![RestartJenkins](https://codeinblue.files.wordpress.com/2017/12/restartjenkins.png)

5; Click ```Yes``` to restart the Jenkins server.

![Restarting](https://codeinblue.files.wordpress.com/2017/12/restarting.png)

## Slack notifications

Personally I love Slack. I use it daily and added lots of bots to make my work easier and more fun. [The Trello bot is one of them!](https://trello.com/platforms/slack)

Setting up the Slack notifications is easy. [I've written a separate blogpost for this. So be sure to check it out!](https://mufana.github.io/blog/2017/08/08/Jenkins-Slack)

## Add service account credentials

We all know those lazy admins who 'instead of using service accounts' are abusing their own credentials to do stuff or execute scripts on servers. And worse, store passwords plain text in scripts. Obviously we're not going to do that.

For this example I've created a Service Account in the Active Directory. ```SA-Jenkins```. This account has the right to:

* Create a folder in: ```\\contoso\Home```

* Set folder permissions for the home folder

* Add an eMail address to any Active Directory user object within the ```users.contoso.com``` organizational unit

* Add standard groups to any Active Directory user object within the ```users.contoso.com``` organizational unit

![Rights](https://codeinblue.files.wordpress.com/2017/12/rights.png)

The: ```SA-VIDUM``` account is member of: ```domain users``` so no admin rights there. For Active Directory rights I used the standard _Delegation of control_ feature.

The folder rights are __Special Rights_. I removed the: ```Delete folder``` right.

## Add credentials to Jenkins

To use the ```SA-VIDUM``` credentials in Jenkins we'll have to add them first. This is where the: ```Environment Injector Plugin``` is for!

1; On the Jenkins server, go to: ```Manage Jenkins``` -> ```Manage Plugins```.

2; Scroll down to: ```Global Passwords```

![Global](https://codeinblue.files.wordpress.com/2017/12/password.png)

3; Enter a name for the credentials. __This is not the username!__

4; Enter the password and click: ```Save```

__The credentials are actually stored as a variable within Jenkins. You can use these variables in the build scripts.__

## Time to code

And now, finally it's fun time!

### Check for newly created users

The first step is to check for newly created users in the Active Directory.

Since I already discussed this _in depth_ I won't go in to details __AGAIN__.

```powershell
# The searchdate (yesterday) for the Get-ADUser search in the Contoso Active Directory
$WhenCreated = "-1"
$SearchDate = ((Get-Date).AddDays($WhenCreated)).Date

# Location of the output CSV file    
$Data = "C:\Scripts\PowerShell\VIDUM\Data"
$Day = (Get-Date).ToString("dd-MM-yy")

$CollUsers = (Get-ADUser `
                -Filter "*" `
                -Server "contoso.com" `
                -SearchBase "ou=users,ou=corp,dc=contoso,dc=com" `
                -Properties "*" | Select samaccountname, sn, givenname 
                )

$CollUsers | export-csv "$Data\$Day-Users.csv" -NoTypeInformation
Write-Output "$($CollUsers.Count) new contoso users"
```

### Create a home folder

#### PowerShell

My code runs on a management server. Which means. The fileserver is a seperate server that I have to access remotely.

```powershell
Function New-Home {

    # Use the Jenkins global password: SAVIDUM.
    $Pass = $env:SAVIDUM | ConvertTo-SecureString -AsPlainText -Force
    $User = "sa-vidum"
    $cred = New-Object System.Management.Automation.PSCredential -ArgumentList $User, $pass
    
    $Data = "C:\Scripts\PowerShell\VIDUM\Data"
    $Day = (Get-Date).ToString("dd-MM-yy")
    $Collusers = (Import-csv -Path "$data\$Day-Users.csv")

    # Create the remote session to the fileserver.
    Invoke-Command -ComputerName localhost -Credential $cred -ScriptBlock {
        $CRHomedirs = $Using:CollUsers
        Write-Output "$($CRHomedirs.Count) homedirs to create!"
        
        # Loop through users
        Foreach ($User in $CRHomedirs) {
            $locDFS = "\\contoso.com\home"
            $HomeDir = $User.samAccountName

            # Check if homedir exist, if not, create.
            $Check = Test-Path -path $locDFS\$HomeDir
                    If ($Check -eq $True) { 
                        Write-Output "$Homedir already exist on: $locDFS"  
                    } Else {
                        New-Item -ItemType Directory -path "$locDFS\$Homedir"
                        Write-Output "$Homedir created on: $locDFS"
                    } # End IF/Else

        } # End Foreach

    } #End Invoke

} # End Function
New-Home
```

Not the most complicated script. But there is however a catch! ```$CRHomedirs = $Using:CollUsers``` What's with the __$Using:CollUsers__ variable?

Well, I'm executing this script in a remote session. The __$CollUsers__ variable is a only available locally. So we have to pass it to the remote session. The way to do is is by: ```$Using:```.

__$Using is only available in PowerShell V3 and higher!__

#### The Jenkins build task

1; Create a new build in Jenkins.

![Newbuild](https://codeinblue.files.wordpress.com/2017/12/1-homedirs.png)

2; Inject the password for: ```SAVIDUM```

In the PowerShell script, the password will be used as follows:

```powershell
$Pass = $env:SAVIDUM | ConvertTo-SecureString -AsPlainText -Force
```

The: ```$env:SAVIDUM``` is the Jenkins global variable.

![InjectPwd](https://codeinblue.files.wordpress.com/2017/12/2-injectpassword.png)

3; Add: ```Build Step``` and check: ```PowerShell```.

4; Copy/Paste the PowerShell script.

![PowerShellCode](https://codeinblue.files.wordpress.com/2017/12/3-buildstep.png)

5; Click the ```Build``` icon to schedule a build.

![Build](https://codeinblue.files.wordpress.com/2017/12/4-buildicon.png)

6; OK, that wasn't working as expected. The screen displays a remoting error. __Access Denied__.

What's wrong? PowerShell is secure (by default) a _non Administrator_ isn't allowed to remotely executes scripts. And since the: ```SA-VIDUM``` account is just a regular user account, the script generates an __Access Denied__.

![RemotingError](https://codeinblue.files.wordpress.com/2017/12/5-noremoting.png)

The cmdlet to see which users and/or groups are allowed to make remote connections is: ```Get-PSSessionConfiguration```

![GetRemotingConf](https://codeinblue.files.wordpress.com/2017/12/6-remoting.png)

Any user who is a member of: ```Remote Management Users``` is allowed to create remoting connections. So all there's to do is to make the ```SA-VIDUM``` account a member of this group.

7; Add the account to the remoting group.

![AddtoGroup](https://codeinblue.files.wordpress.com/2017/12/7-remotegroup.png)

8; Schedule the build again and voila!

![Sucess](https://codeinblue.files.wordpress.com/2017/12/8-success.png)

![Folders](https://codeinblue.files.wordpress.com/2017/12/9-folders.png)

### Assign access rights

For the access rights I always use the _NTFSSecurity_ module. [You can find it on the PowerShell gallery](https://www.powershellgallery.com/packages/NTFSSecurity/4.2.3)
It's a great module and makes your life a lot easier when it comes to assigning access rights to folders.

The module contains lots cmdlets. But, for this case I only need the: ```Add-NTFSAccess``` cmdlet.

The cmdlet itself is very easy. ```Add-NTFSAccess -Path "Path to folder" -Account "The user Account" -AccessRights FullControl```

I wrapped a function around it to meet my needs.

```powershell
Function Set-AccessRights {

    # Use the Jenkins global password: SAVIDUM.
    $Pass = $env:SAVIDUM | ConvertTo-SecureString -AsPlainText -Force
    $User = "sa-vidum"
    $cred = New-Object System.Management.Automation.PSCredential -ArgumentList $User, $pass
    
    $Data = "C:\Scripts\PowerShell\VIDUM\Data"
    $Day = (Get-Date).ToString("dd-MM-yy")
    $Collusers = (Import-csv -Path "$data\$Day-Users.csv")

    # Create remote session to the fileserver
    Invoke-Command -ComputerName localhost -Credential $cred -ScriptBlock {
        $CRHomedirs = $Using:CollUsers
        Write-Output "$($CRHomedirs.Count) homedirs to create!"
        
        # Loop through users
        Foreach ($User in $CRHomedirs) {
            $locDFS = "\\contoso.com\home"
            $HomeDir = $User.samAccountName

            # Assign fullcontrol rights
            Add-NTFSAccess -Path "$locDFS\$Homedir" -Account "$Homedir" -AccessRights FullControl -PassThru

        } # End Foreach

    } #End Invoke

} # End Function
Set-AccessRights
```

#### The Jenkins Task

For the Jenkins task I used the same approach as before.

Running the tasks successfully generates the following output:

![Success](https://codeinblue.files.wordpress.com/2017/12/rightssusccess.png)

### Add standard groups

Now, we also want our new users to automatically get a few memberships. In this case, I want them to be a member of the: ```All-Employees``` group. This ensures they will receive general company eMails.

```powershell
$Data = "C:\Scripts\PowerShell\VIDUM\Data"
$Day = (Get-Date).ToString("dd-MM-yy")
$UserList = "$Data\$Day-Users.csv" 
$CollUserCSV = (Import-CSV $UserList)
$Server = "contoso.com"
$GroupName = "All_Employees" 

# Loop through users
Foreach ($User in $CollUserCSV) {
        $Member = $User.samAccountName
        Add-AdGroupMember -identity "$GroupName" -Server "$Server" -Members $Member -PassTrhu
}
```

No rocketscience here! Just a simple Foreach constructor.

For the Jenkins build I also inject the password for: ```SA-VIDUM```

![InjectPwd](https://codeinblue.files.wordpress.com/2017/12/2-injectpassword.png)

The output of the build is as follows:

![Output](https://codeinblue.files.wordpress.com/2017/12/groupmembersoutput.png)

![ActiveDirGroup](https://codeinblue.files.wordpress.com/2017/12/groupmembers.png)

## Scheduling

Scheduling builds is done the cron way. Cron comes from Unix/Linux and is used to schedule shell scripts.

A cron scheduler for 08:00am is: ```0 8 * * *```

This may seem complicated. But once you've got the hang of this, it becomes second nature.

1; To add a scheduler, click on the task you want to schedule.

2; Click left on ```Configure```

![Configure](https://codeinblue.files.wordpress.com/2017/12/configure.png)

3; Scroll down to ```Build Triggers``` and set the checkbox at ```Build periodically```

![Trigger](https://codeinblue.files.wordpress.com/2017/12/trigger.png)

4; Enter a cron scheduler. For instance: ```00 08 ** 1-7``` This will schedule a daily build starting at 08:AM.

## End of part (2)

In part 3 I'll show you how to generate unique eMail addresses and how to assign an Office365 license.

[Go back](https://mufana.github.io/blog)