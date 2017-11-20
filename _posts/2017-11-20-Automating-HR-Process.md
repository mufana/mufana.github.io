---
title: "Automating an HR process (Part1)"
date: 2017-11-20
---

# Automating an HR process (Part 1)

## Background

A few months back I was asked to take part in a project to improve my company's HR process. To be more specific; the hiring process and making sure that the person getting hired, has instant access to our network and all the applications required for the function. 

A typical hiring process flows over various different departments in an organisation. _HR, finance, planning and IT_. Since I'm the IT guy, my focus is mainly on the IT part.

The IT process is as follows:

_The creation of accounts 'or identity's' is managed by an IDM process outside of our company and is beyond influance. Futhermore, it solely manages the creation of accounts and nothing more._

### Part 1 (The Manager)

* The Manager of the person getting hired makes a few phone calls to our Helpdesk and opens a request for change for a network account.

### Part 2 (The IT Admin)

* Creates the homefolder on a DFS file share *
* Assigns rights to the homefolder *
* Generates a unique eMail address *
* Adds the account to a few standard application groups *
* Assigns an Office 365 standard license to the account *
* Closes the RFC *

_(*) Carried out manually_

Normally it takes an Admin about ten minutes to 'create' the account, homefolder, eMail address and assign the standard application groups and Office 365 license. And not to forget, the Manager also invests time by opening up a RFC. So, add another 5 minutes.

That's 15 minutes for one account. My company hires a staggering 500 people each year. So this might just be a massive time and money saver.

We pretty much agreed that we needed to do something now. And, since we are an _agile_ company, doing something is exactly what we did.

### My lab evironment

How does my lab environment looks like:

* Accounts are provisioned in the local Active Directory. ```Contoso.com```.
* The homefolder for each users is located on a DFS share. ```\\contoso.com\home```.
* The group ```All_Employees```  enables access to a standard appliction set.
* The domain is called: ```Contoso.com``` so the users eMail address should be: _someone@contoso.com_.
* Since we're using Office365, I have to assign each user with a standard license.

## How to begin

Normally I always start my coding journey in the PowerShell console. It gives me room to play around a bit and found out what works best. But since this could be a big project, I decided to think it through.

### Step 1 - Break it down

The first step in the process is that; the Manager creates an RFC. This is a litte strange because the account of the person getting hired is already created in our Active Directory. There's an IDM process _managed externally_ responsible for this. So, the RFC is completely unnecessary.

And that is not very agile. So, _click, click, deleted!_.

Ok, well that's deleted. But without an RFC we (IT guys) don't know who's getting hired and which accounts need some work. So I still need to do some coding for everything that is done manually by my company's IT department. Those tasks are:

* Create the homefolder on a DFS file share
* Assign rights to the homefolder
* Generate a unique eMail address
* Add the account to a few standard application groups
* Assign an Office 365 standard license to the account

Thankfully all this can be scripted with PowerShell.

### Make it visable

I decided to make things a little more visable. (Also great for the non IT people in our project).

![Process](https://codeinblue.files.wordpress.com/2017/11/5.png)

Now I know what to code, it's time to start with a basic design. As we all know, with PowerShell there are many ways to achieve something.

#### The questions

* Where am I gonna place all server/domain specific parameters
* Write 1 huge script or split up and create a script for every task
* How to run the script(s). (Windows Taks Schedular or Jenkins)
* What about logging
* ...

#### The technical goals

* Easy to maintain
* Easy to change server/domain/eMail specific settings
* Logfile (preferably a webpage)
* Scalable (I might have to add more automated tasks in the near future)

With this in mind I came up with the following design:

#### Powershell Module (PSVIDM)

![Design](https://codeinblue.files.wordpress.com/2017/11/2017-11-20-00_20_55-jb_vidum-docx-word.png)

The module has seven different blocks.

* Get-NewADUsers.ps1
* Add-Attributes.ps1
* New-eMail.ps1
* New-HomeFolder.ps1
* Set-Permissions.ps1
* Globals.ps1

The core component of the module is the ```Get-NewADUsers.ps1``` script. This forms the input for all other scripts. 

## Time to plunge into the 21st century

Now that I have my master plan, It's time to code.

### Get-NewADUser.ps1

The first step is to gather a list of new accounts provisioned in the local Active Directory. Thankfully I know the Active Directory cmdlets quite well. For those of you who don't, there is a cmdlet called ```Get-AdUser```. So, let's play a bit and see what kind of information that brings.

```powershell
Get-ADUser -filter * -properties *
```

The above cmdlet returns lots of information about every single account in the Active Directory. Nice, but not quite what I'm after.

![get-aduser](https://codeinblue.files.wordpress.com/2017/10/2017-10-21-01_37_58-testlab1-on-interstellar-virtual-machine-connection.png)

I noticed a property called: _whencreated_. And since I only want to get the accounts created _today or yesterday_ I can create a filter based on this property.

```powershell
$WhenCreated = '-1'
$SearchDate = ((Get-Date).AddDays($WhenCreated))
Get-ADUser -filter {whencreated -eq "$WhenCreated"} -properties *
```

So, what does this do?

The first part ```$WhenCreated = '-1'``` defines a variable called: "WhenCreated" with a value of "-1".

The second part ```$SearchDate = ((Get-Date).AddDays($WhenCreated))``` defines another variable. "SearchDate".

This variable holds a cmdlet ```Get-Date```. This cmdlet gets the current date/time.

Lets say the current date is: _Friday November 17_.

![get-datestandard](https://codeinblue.files.wordpress.com/2017/11/2.png)

```.AddDays``` is a method.

PowerShell generates objects. 

Let's say a motorcycle object. Objects have properties. A motorcycle typically comes with a odometer. The figure below shows a 'motorcycle' object which has a property 'odometer' and two methods. (Start and Stop).

![object](https://codeinblue.files.wordpress.com/2017/11/2017-11-20-00_56_03-document1-recovered-word.png)

By using the method ```start``` you can change the way an object looks. If you start and ride the motorcycle, the odometer keeps track of the distance you've travelled and adds that to the current value. Riding the motorcycle changes the properties of the odometer.

Because I want to search for accounts created in the last 24 hours, I use the method: ```AddDays```. The value of this method is: "-1". Which means, subtract "-1" day from the current date.

If we subtract 1 day from _ Friday November 17_ you obviously get:

![get-dateadd](https://codeinblue.files.wordpress.com/2017/11/3.png)

The last part is to use this in the: "Get-ADUser" cmdlet. 

 ```powershell
$WhenCreated = "-4" 
$SearchDate = ((Get-Date).AddDays($WhenCreated)).Date
Get-ADUser -Filter {whenCreated -ge $SearchDate} `
           -Server "contoso.com" `
           -SearchBase "ou=users,ou=corp,dc=contoso,dc=com" `
           -Properties * `
```

__Notice that I used '-4' in this example.__

To test the code:
![OutputGetUser](https://codeinblue.files.wordpress.com/2017/11/2017-11-20-11_30_22-vidum_ad-fresh-install-running-oracle-vm-virtualbox.png)

Cool! that works. But I still haven't met my original goal to make the scripts easy to maintain. There's still server/domain specific data in the script. I don't want that.

### _Globals.ps1

In order to remove all server/domain specific data from the script(s), I create a _globals.ps1_ file. In this file I store things like:

* Domain name
* User OU in the Active Directory
* Location of homefolder on the DFS share
* Etc...

To use this file in my scripts I have to DotSource it.

#### Scoping

Each script has it's own scope. Known as the "Script Scope" All variables and functions defined in the script are only available in the script and nowhere else.

Let's pretend we have a script called: _'Script (A).ps1'_ which contains a function called: ```Start-NewFunction```.

I have another script _'Script (B).ps1'_ with the function: ```Stop-NewFunction``` that I want to use in: _Script (A).ps1'_

The easy way is to DotSource _'Script (B).ps1'_ in _'Script (A).ps1'_.

DotSourcing enables the use of variables and functions in other scripts.

```powershell
content of script (A.ps1)
.\ScriptB.ps1
function Start-NewFunction {
    Write-Host "Start-NewFunction"
}
```

My _Globals.ps1_ currently looks like this:

```powershell
$Date = (get-date).ToShortDateString().Replace("/","-")
$Domain = "Contoso.com"
$WhenCreated = "-4"
$Props = "samAccountName, sn, UserPrincipalName"
$UserOU = "ou=Users,ou=Corp,dc=contoso,dc=com"
$DFSHome = "\\contoso.com\home"
$CSVpath = "C:\PowerShell\PSVIDUM\Data"
$CSVFile = "$Date-CollADUsers.csv"
```

#### More methods

Huh! wait a second? What about this line? ```$Date = (get-date).ToShortDateString().Replace("/","-")```

Remember what I wrote about 'Methods'? ```.ToShortDateString()``` and ```.Replace()``` are methods as well.

My ```$CSVFile``` variable holds the filename for the CSV file I want to export. Part of the filename is the date the CSV file is created.

```.ToShortDateString()``` converts the date to a short string.

![short](https://codeinblue.files.wordpress.com/2017/11/2017-11-20-12_21_10-vidum_ad-fresh-install-running-oracle-vm-virtualbox.png)

As you can see the format is: _mm/dd/yyyy_. The '/' charachter is a little anoying because it's interpreted as a part of the file location. _'C:\11\20\2017\'_. So I have to make a minor tweak.

And that's where ```.Replace()``` comes to the rescue. By using this method I can replace the '/' for a '-'.

![replace](https://codeinblue.files.wordpress.com/2017/11/2017-11-20-12_21_40-vidum_ad-fresh-install-running-oracle-vm-virtualbox.png)

All there's left is to DotSource this file within the: ```Get-NewADUsers.ps1``` script.

```powershell
# DotSource the 'globals.ps1' file
.$PSScriptRoot\_Globals.ps1

$SearchDate = ((Get-Date).AddDays($WhenCreated)).Date

Get-ADUser -Filter {whenCreated -ge $SearchDate} ` #Backtick here
           -Server "$Domain" ` #Backtick here
           -SearchBase "$UserOU" ` #Backtick here
           -Properties * ` #Backtick here
| Select $props | export-csv -Path $CSVpath\$CSVfile -NoTypeInformation
```

__The variable ```$PSScriptRoot``` is a system variable. This is the current location from where the script is executed. This variable is set when the script is executed. if the script is finished, the variable will be empty.__

To export all the data to a CSV file I've added the ```Export-CSV``` cmdlet.

### New-HomeFolder.ps1

Next step is to create the users homefolder on the DFS share. Good news is that Microsoft also provided a standard cmdlet for this. ```New-Item```. This is actually a general cmdlet. Usefull in many places.

Let's play around with it.

```powershell
New-Item -ItemType Directory -Path C:\Temp\Test -WhatIf
```

The above code creates a new directory: ```C:\Temp\Test```. If one of these directory's don't exist, they will be created. 

![new](https://codeinblue.files.wordpress.com/2017/11/2017-11-20-12_06_00-vidum_ad-fresh-install-running-oracle-vm-virtualbox.png)

So, that works. But of course there's a little more to it then this.

First I have to import the CSV file I created with my _Get-NewADUsers.ps1_ script.
```$CollUsers = Import-Csv $CSVpath\$CSVFile```

The next step if to create a foreach statement.
```Foreach ($User in $CollUser)```

Now, the foldername of the HomeFolder I want to create is the samAccountName of the user.
```$FolderName = $User.samAccountName```

I also want to make sure that I only create folders that don't exist. To do that I create a simple test. ```$Test = (test-path -path $Location\$FolderName)```.

Last I validate the test within an 'if/else' statement. 

The script:

```powershell
.$PSScriptRoot\_Globals.ps1
$CollUsers = Import-Csv $CSVpath\$CSVFile

Foreach ($User in $CollUsers) {
    $FolderName = $User.samAccountName
    $Test = (test-path -path $DFSHome\$FolderName)

    If ($Test -eq $True) {
        write-error "<New-HomeFolder>: The folder: <$FolderName> already exist on: <$DFSHome>"
          
    } Else {
        New-Item -ItemType Directory -Path $DFSHome\$FolderName -Verbose -WhatIf
        }           
}
```

And a little test to see if it works...

![folders](https://codeinblue.files.wordpress.com/2017/11/2017-11-20-12_53_11-vidum_ad-fresh-install-running-oracle-vm-virtualbox.png)

### End of part 1

Since this is an exeptional long post, I decided to split it up. So make sure to come back for part 2.

[Go back](https://mufana.github.io/blog)