---
title: "SQL job reports with PowerShell"
date: 2017-08-10
excerpt: "Getting SQL Job reports with PowerShell"
tags:
- PowerShell
- SQL
categories: powershell
comments: true
author_profile: true
---

# SQL job reports with PowerShell

## Background

One of the primary tasks in my work as System Engineer is to administer a SQL Server environment. And this also involves keeping track of all SQL Server jobs running on the SQL agent. Those jobs include:

* Backup of databases
* Delete purge history
* Optimize index
* Application specific jobs

Each of these jobs is scheduled to run at a specific time. Either: Daily or Weekly.

Now it is absolutely vital to know the outcome of job. If something goes wrong with 'e.g' the database backup, I want to know so that I can take appropriate action. They way to do this is by email.

I want this email to be sent to my colleagues and functional analyst. If certain application specific jobs did not run, they are the ones who would like to be informed.

### How to mail

Well, there are multiple ways. For instance: Configure _SQL Server Agent database mail_. That gives me pretty much what I want. 

![NoFormat](https://codeinblue.files.wordpress.com/2017/08/sql1.png) 

Except, it doesn't look very nice. As I mentioned before, I want to send this to a few people in my organisation. So, it has to look professionally.

### PowerShell

And that's where PowerShell comes to the rescue. 

### Play time

I like to begin coding on the PowerShell Console. That gives me the opportunity to play around a bit and find what works best. In this case I start my journey on a SQL server.

First: I have to load the correct assembly. Now, this method is deprecated. However, it still works. I may change it in the future though. 

1. Import the SQL Server assembly:

```powershell
    [void][System.Reflection.Assembly]::LoadWithPartialName("Microsoft.SqlServer.SMO")
```

2. Create a new variable ```$SQLServername``` and set it to: ```Localhost```.

```powershell
$SQLServername = 'Localhost'
```

3. Create a new object: ```Microsoft.sqlserver.Management.smo.server``` and point it to the: ```$SQLservername``` variable.

```powershell
$SQLServer = new-object Microsoft.sqlserver.management.smo.server($SQLServername)'
```

4. Get the jobs status.

```powershell
$SQLServer.JobServer.jobs
```

This will generate the following output.

![Output](https://codeinblue.files.wordpress.com/2017/08/sql4.png)

It will display properties like: _LastRunOutcome_,_NextRunDate_. Nice! that's exactly what I'm after. 

##### You can't see those properties since I had to cut some information from the image.

### Getting serious

So, the above code works! But I wan't to be able to run this from a management server. Thus; executing this code remotely. How to do this?

1. Create a new ```PSSession```.

```powershell
$JobSession = New-PSSession -ComputerName $Server
```

I have my job session. But I want to execute code within that session. Now, the cmdlet ```Invoke-Command``` comes with a parameter to accomplish that. ```-ScriptBlock```. So, I simply add the code I already used to the scriptblock! 

3. 

```powershell
$JobSession = New-PSSession -ComputerName $Server
    Invoke-Command -Session $JobSession -ScriptBlock {

        [void][System.Reflection.Assembly]::LoadWithPartialName("Microsoft.SqlServer.SMO")
        $sqlServerName='localhost'

        $sqlServer = New-Object Microsoft.SqlServer.Management.Smo.Server($Server)
        $sqlServer.JobServer.Jobs
}
```

But! remember, running this code gave me a complete overview of all the properties. I just want to see the interesting ones. To do this I add the variable: ```$job``` with a value of: ```$sqlServer.JobServer.Jobs```. The only thing remaining is to pipe that to: ```Select``` and select the properties I need.

```PowerShell
$job = $sqlServer.JobServer.Jobs
$job | select Name, CurrentRunStatus, IsEnabled, LastRunDate,LastRunOutcome,NextRunDate
```
This gives me the following output:

![Output](https://codeinblue.files.wordpress.com/2017/08/sql5.png)

### Create the function

What's left is to turn this code into a function. 

```powershell
Function Get-SQLJobs {
    
    [CmdletBinding()]

    Param[String]$Server

    $JobSession = New-PSSession -ComputerName $Server
        Invoke-Command -Session $JobSession -ScriptBlock {

        [void][System.Reflection.Assembly]::LoadWithPartialName("Microsoft.SqlServer.SMO")
        $sqlServerName='localhost'

        $sqlServer = New-Object Microsoft.SqlServer.Management.Smo.Server($Server)

        $job = $sqlServer.JobServer.Jobs
        $job | select Name, CurrentRunStatus, IsEnabled, LastRunDate,LastRunOutcome,NextRunDate
    }
}
```

But we're not done yet. Remember! I want to email the output of the: ```Get_SQLJobs``` function. And preferably in a nice HTML format.

## That's got spam in it

To accomplish that, the first thing I have to do is create a CSS stylesheet. For those who are not familiar with ```CSS```, A CSS stylesheet contains a written presentation of a HTML document. Like:

* Fonts
* Colors
* Layout
* Border size
* ...

With a stylesheet you define how a HTML page looks.

### The stylesheet

The code for the stylesheet is pretty straight forward. There are many examples to find on the Internet so you don't have to invent your own.

```css
<style>
@charset "UTF-8"; 

table
{
font-family:"Trebuchet MS", sans-serif;
border-collapse:collapse;
}
td 
{
font-size:1em;
border:1px solid #778899;
padding:5px 5px 5px 5px;
}
th 
{
font-size:1.1em;
text-align:center;
padding-top:5px;
padding-bottom:5px;
padding-right:7px;
padding-left:7px;
background-color:#778899;
color:#ffffff;
}
name tr
{
color:#F00000;
background-color:#778899;
}
</style>
```

### The PowerShell code

Now that I have the stylesheet, it's time to create a new script.

1. Create a new variable: ```$Header``` that will hold the stylesheet.

2. The value of: ```$Header``` is going to be: ```@"``` to hold a string of data. 

```powershell
$header=@"
<style>
@charset "UTF-8"; 

table
{
font-family:"Trebuchet MS", sans-serif;
border-collapse:collapse;
}
td 
{
font-size:1em;
border:1px solid #778899;
padding:5px 5px 5px 5px;
}
th 
{
font-size:1.1em;
text-align:center;
padding-top:5px;
padding-bottom:5px;
padding-right:7px;
padding-left:7px;
background-color:#778899;
color:#ffffff;
}
name tr
{
color:#F00000;
background-color:#778899;
}
</style>
"@
```

3. I want to give my email a date. To be precise; the current date. I want to use that in both the: ```Body``` and: ```subject``` of the email.

```powershell
$Date = Get-date -Format F
```

![Date](https://codeinblue.files.wordpress.com/2017/08/sql6.png)

4. Next step is create a new variable called: ```$Data```. The value of this variable is the acutal function: ```Get-SQLJobs```

```powershell
$data = Get-SQLJobs
```

5. Create another variable called: ```$HTML```. The value of this variable is the: ```$data``` variable piped to: ```select```. 

Then, pipe ```select``` to: ```ConvertTo-HTML```. 

* The parameter: ```-head``` points to the stylesheet variable I've created earlier on.

* The parameter ```-PreContent``` adds text before an opening table.

Finally, pipe ```ConvertTo-HTML``` to: ```Out-String```.

```powershell
$HTML = $data | select Name, LastRundate,NextRunDate, LastRunoutcome, IsEnabled | ConvertTo-Html -head $header -PreContent "<h2>Daily SQL Job Report generated on $date</h2>" | Out-String
```

6. The only thing left is to use the: ```Send-MailMessage``` cmdlet.

```powershell
Send-MailMessage -SmtpServer smtpserver `
-to someone@somedomain.com `
-subject "Daily SQL Job Report generated on $date" `
-from someone@somedomain.com
```

Schedule this script on a server and you're good to go!
The output looks similair to this:

![finally](https://codeinblue.files.wordpress.com/2017/08/sql7.png)

The complete script:

```powershell
$header=@"
<style>
@charset "UTF-8"; 

table
{
font-family:"Trebuchet MS", sans-serif;
border-collapse:collapse;
}
td 
{
font-size:1em;
border:1px solid #778899;
padding:5px 5px 5px 5px;
}
th 
{
font-size:1.1em;
text-align:center;
padding-top:5px;
padding-bottom:5px;
padding-right:7px;
padding-left:7px;
background-color:#778899;
color:#ffffff;
}
name tr
{
color:#F00000;
background-color:#778899;
}
</style>
"@

$Date = Get-date -Format F

$data = Get-SQLJobs

$HTML = $data | select Name, LastRundate,NextRunDate, LastRunoutcome, IsEnabled | ConvertTo-Html -head $header -PreContent "<h2>Daily SQL Job Report generated on $date</h2>" | Out-String

Send-MailMessage -SmtpServer smtpserver `
-to someone@somedomain.com `
-subject "Daily SQL Job Report generated on $date" `
-from someone@somedomain.com
```




