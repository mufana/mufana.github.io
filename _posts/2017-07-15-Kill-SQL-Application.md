---
title: "PowerShell and SQL (Headshots)"
date: 2017-07-15
---

# PowerShell and SQL (headshots)

## Background

Recently I ran into a challenge where I had to update a SQL application. This particlair application consisted of a client part. (running on VMware View clients) and a server part. (running on Windows 2012 R2 / SQL 2014 SP2)

Before I could begin with the update itself, I had to phycially close every instance of the application on the VMware View clients. Therefore, I had to login to the SQL Server, open up the SQL Management Studio, go to the processes and check which VMware View client is connected. (Then, login to VMware View Administrator, find the clients and check who's logged in to that client. Last, find the phonenumber of that person and make a phonecall.

Now, I like my work. I really do. But what I don't like is make phonecalls to 100+ individual users and ask them to please stop working in the application.

## The environment

The application ran on a DTAP environment. Each enviroment had to be updated on different times and; on each of them, people where doing there work. And of course, each enviroment was placed on a different SQL server with different database names and SQL instances.

## Recreating

* The SQL database running on server: Lab1

* The client computer also running on: Lab1

Now, the challenge is to stop the application on Lab1.

## The SQL script

As you can see in the image below, I have two databases. I want to find out who is connected to the production database. 

![Image of SQL](https://codeinblue.files.wordpress.com/2017/02/1.png)

To find this out I ran 'exec sp who' against the production DB.

```SQL
exec sp who
```

This resulted in the information displayed below:

![Image of SQL](https://codeinblue.files.wordpress.com/2017/02/3.png)

Nice...
As you can see, the output displays the names of the clients connected to the database. That makes it easy to simply create a PowerShell script that executes something like:

```batch
taskkill /s HOSTNAME /im APPLICATION.EXE
```

## The PowerShell magic

The SQL server had more then 100 connections to the database. So, no way I was gonna run the taskkill command to each of them.

I decided to use that time to write a script that does the job for me.

### A few things to take into consideration

* Make this script a simple tool to use for someone who has no knowledge of PowerShell.

* Make sure that this script can utilize all SQL servers, databases and instances and 'most importantly' stop the application that belongs to a specific enviroment.

* Prevent an administrator to stop an application by accident.

### Step 1

Create vars to hold the path to the different applications

```PowerShell
$TestAppExe = "Test\Notepad.exe"
$ProdAppExe = "Prod\Notepad.exe"
```

### Step 2

Create a datatable to hold enviroment name, server, instance, database and executable names.


```PowerShell
$App = New-Object System.Data.DataTable
$App.Columns.Add((New-Object System.Data.DataColumn ‘Name’, ([string])))
$App.Columns.Add((New-Object System.Data.DataColumn ‘SQLServer’, ([string])))
$App.Columns.Add((New-Object System.Data.DataColumn ‘SQLInstance’, ([string])))
$App.Columns.Add((New-Object System.Data.DataColumn ‘Database’, ([string])))
$App.Columns.Add((New-Object System.Data.DataColumn ‘Executable’, ([string])))
```

### Step 3

Add a row for Test and Production with the correct server/databases/instance names. The columns correspond with the defined columns in step 2.

```PowerShell
$App.Rows.Add("Test","Lab1","Lab1","TestDB","$TestAppExe")
$App.Rows.Add("Production","Lab1","Lab1","ProductionDB","$ProdAppExe")
```

### Step 4

Create a function to 'select' the enviroment specified in the rows from step 3.
This is displayed to the admin through a simple selection box.

The GUI:

![Image of select](https://codeinblue.files.wordpress.com/2017/02/21.png)

The script:

```PowerShell
Function Get-Environment
{
$objForm = New-Object System.Windows.Forms.Form 
$objForm.Text = "Select environment"
$objForm.Size = New-Object System.Drawing.Size(300,250) 
$objForm.StartPosition = "CenterScreen"

$objForm.KeyPreview = $True
$objForm.Add_KeyDown({if ($_.KeyCode -eq "Enter") 
    {$global:objChoice=$objListBox.SelectedItem;$objForm.Close()}})
$objForm.Add_KeyDown({if ($_.KeyCode -eq "Escape") 
    {$objForm.Close()}})

$OKButton = New-Object System.Windows.Forms.Button
$OKButton.Location = New-Object System.Drawing.Size(75,130)
$OKButton.Size = New-Object System.Drawing.Size(75,23)
$OKButton.Text = "OK"
$OKButton.Add_Click({$global:objChoice=$objListBox.SelectedItem;$objForm.Close()})
$objForm.Controls.Add($OKButton)

$CancelButton = New-Object System.Windows.Forms.Button
$CancelButton.Location = New-Object System.Drawing.Size(150,130)
$CancelButton.Size = New-Object System.Drawing.Size(75,23)
$CancelButton.Text = "Cancel"
$CancelButton.Add_Click({$objForm.Close()})
$objForm.Controls.Add($CancelButton)

$objLabel = New-Object System.Windows.Forms.Label
$objLabel.Location = New-Object System.Drawing.Size(10,20) 
$objLabel.Size = New-Object System.Drawing.Size(280,20) 
$objLabel.Text = "Select environment: "
$objForm.Controls.Add($objLabel) 

$objListBox = New-Object System.Windows.Forms.ListBox 
$objListBox.Location = New-Object System.Drawing.Size(10,40) 
$objListBox.Size = New-Object System.Drawing.Size(260,20) 
$objListBox.Height = 90

$App | foreach-object{
[void] $objListBox.Items.Add($_.Name)
}

$objForm.Controls.Add($objListBox) 

$objForm.Topmost = $True

$objForm.Add_Shown({$objForm.Activate()})
[void] $objForm.ShowDialog()
Return $objChoice
}
```

### Step 5

To prevent an administrator to stop an application by accident, I decided to throw in two extra functions.

* One to display a messagebox with some 'do you want to continue' questions.

```PowerShell
Function Show-MsgBox
 
($Text,$Title="",[Windows.Forms.MessageBoxButtons]$Button = "OK",           [Windows.Forms.MessageBoxIcon]$Icon="Information"){
[Windows.Forms.MessageBox]::Show("$Text", "$Title", [Windows.Forms.MessageBoxButtons]::$Button, $Icon) | ?{(!($_ -eq "OK"))}
}
```

* And another function to write specific information to the console.

```PowerShell
Function Write-Console {

    Param (
        [String]$String
        )

        $Time = Get-Date -Format T
        $Write = $time + " " + $String
        Write-Host -fore green $Write
    }
```

### Step 6

## Time for the real script

### First step is to get the selection the admin made

If the admin clicks on 'Production', we need to gather all the information (server, database, instance, application) belonging to that selection. (Remember the datatable from Step 2 and 3.)

![Image of select](https://codeinblue.files.wordpress.com/2017/02/21.png)

```PowerShell
$Environment = Get-Environment
Write-Console "Chosen environment: $Environment"
If((Show-MsgBox -Title "Chosen environmentis: $Environment" -Text "The Chosen environment is: <$Environment> Continue!?" -Button YesNo -Icon information) -eq 'No'){Exit}

else{
    $chosen = $App |select-object * | Where-Object{$_.Name -eq $Environment}
   }
```

### Second step is to build new vars

The 'Get-Environment' function gets the selected environment. (Rememeber the rows from step 3?)

```PowerShell
$App.Rows.Add("Test","Lab1","Lab1","TestDB","$TestAppExe")
$App.Rows.Add("Production","Lab1","Lab1","ProductionDB","$ProdAppExe")
```

Based on the selection I build new vars to contain the server,database,instance belonging to the selection. The selection is stored in a var called '$Chosen'.

```PowerShell
# Set the local vars
$SQLServer = $chosen.SQLServer
$SQLInstance = $chosen.SQLInstance
$SQLDatabase = $chosen.Database
$ApplicionExe = $chosen.Executable
$Outfile = "C:\Scripts\ActiveComputers.txt"
```

### Create the remote session to the SQL server

Next is to setup the remote session to the SQL server and actually run the 'sp who' query against the database.

```PowerShell
# Create the new PowerShell session and pass the local vars to the remote sessions with $Using    
$Session = New-PSSession -ComputerName $SQLServer
$Connections = Invoke-Command -Session $Session -ScriptBlock { Invoke-SQLcmd -query "exec sp_who" -Serverinstance `
 $Using:SQLInstance -Database $Using:SQLDatabase }
Remove-PSSession $Session
# Sort active connections based on machinename
$Temp = $Connections |Where-object {$_.hostname -like "*Lab*"}
$ActiveHosts = $temp.hostname | sort-object -Unique
```

#### Stranger things

Notice the $Using variable?

```PowerShell
Invoke-Command -Session $Session -ScriptBlock { Invoke-SQLcmd -query "exec sp_who" -Serverinstance `
 $Using:SQLInstance -Database $Using:SQLDatabase }
 ```

 Now, I'm making a remote session. In this remote session; my variable '$SQLDatabase' doens't exists. It is only available localy. But I still want to use it. To do this: I specify '$Using'. This indicates that the variable can be used in a remote command. The $Using scope identifies the $SQLDatabase variable as a local one.

 Now, the SQL query 'sp who' stores the output in variable called '$temp'.
 Since the application sometimes creates four or more connections Ffrom the same client) to the database, I have to filter on Unique hostnames.

 ```PowerShell
 # Sort active connections based on machinename
$Temp = $Connections |Where-object {$_.hostname -like "*Lab*"}
$ActiveHosts = $temp.hostname | sort-object -Unique
```

### Simply 'kill' the bastard

The last step is to stop the application on the host where it's running.

To do this we simply add a 'foreach' constructor and call taskkill.
We write every kill to the console.

```PowerShell
Foreach ($Comp in $ActiveHosts) {
    taskkill /s $Comp /IM "Notepad.exe" 
    Write-Console "Application.exe closed op: $Comp"
    }
```

It's just like playing Quake! Minus the headshots!

## Wrapping things up

I wrote this script in less then an hour. And I'm using it every time I have to do updates for the application I wrote it for. Sometimes this happens every week. Saves me lots of phonecalls and time.

## The complete script

```PowerShell
#Requires -Version 3.0

<#
=================================================================
Filename      :  Kill-Application.ps1
Version       :  0.2
Created by    :  Joe Blaaw
Created on    :  16-01-2017

Last Modified : 

OS            :  Microsoft Windows Server 2012 R2
PSversion     :  3.0
================================================================
#>

<#
===============================TODO=============================


===============================TODO=============================
#>

<#
============================ChangeLog===========================
Author     :: Mufana
#_______________________________________________________________
Version    :: 0.1 - Initial release
#_______________________________________________________________
Version    :: 0.2 - Update
ChangedBy  :: Mufana
ChangeLog  :: Added multiple GUI / Verification windows + Added logging.
              Solved bug that script didn't exit if no environment was selected. 
              Added function <Show-msgbox> to display information to the user.
              Added function <Write-Console> to write specific info to the console.   
ChangeDate :: 17-01-2017
#_______________________________________________________________
Version    ::
ChangedBy  :: 
ChangeLog  :: 
ChangeDate ::
#_______________________________________________________________
Version    ::
ChangedBy  :: 
ChangeLog  :: 
ChangeDate ::
#_______________________________________________________________
Version    ::
ChangedBy  :: 
ChangeLog  :: 
ChangeDate ::
============================ChangeLog===========================
#>


# Sideload additional modules to generate the GUI.
# _______________________________________________________________________________ 
#[void] [System.Reflection.Assembly]::LoadWithPartialName("System.Windows.Forms")
#[void] [System.Reflection.Assembly]::LoadWithPartialName("System.Drawing") 

# Set Vars
# _______________________________________________________________________________ 
$TestAppExe = "Notepad.exe"
$ProdAppExe = "Notepad.exe"

# Create the table that contains all the data
# _______________________________________________________________________________ 
$App = New-Object System.Data.DataTable
$App.Columns.Add((New-Object System.Data.DataColumn ‘Name’, ([string])))
$App.Columns.Add((New-Object System.Data.DataColumn ‘SQLServer’, ([string])))
$App.Columns.Add((New-Object System.Data.DataColumn ‘SQLInstance’, ([string])))
$App.Columns.Add((New-Object System.Data.DataColumn ‘Database’, ([string])))
$App.Columns.Add((New-Object System.Data.DataColumn ‘Executable’, ([string])))
$App.Rows.Add("Test","Lab1","Lab1","TestDB","$TestAppExe")
$App.Rows.Add("Production","Lab1","Lab1","ProductionDB","$ProdAppExe")

# Function Get-Environment
# _______________________________________________________________________________ 
Function Get-Environment
{
$objForm = New-Object System.Windows.Forms.Form 
$objForm.Text = "Select environment"
$objForm.Size = New-Object System.Drawing.Size(300,250) 
$objForm.StartPosition = "CenterScreen"

$objForm.KeyPreview = $True
$objForm.Add_KeyDown({if ($_.KeyCode -eq "Enter") 
    {$global:objChoice=$objListBox.SelectedItem;$objForm.Close()}})
$objForm.Add_KeyDown({if ($_.KeyCode -eq "Escape") 
    {$objForm.Close()}})

$OKButton = New-Object System.Windows.Forms.Button
$OKButton.Location = New-Object System.Drawing.Size(75,130)
$OKButton.Size = New-Object System.Drawing.Size(75,23)
$OKButton.Text = "OK"
$OKButton.Add_Click({$global:objChoice=$objListBox.SelectedItem;$objForm.Close()})
$objForm.Controls.Add($OKButton)

$CancelButton = New-Object System.Windows.Forms.Button
$CancelButton.Location = New-Object System.Drawing.Size(150,130)
$CancelButton.Size = New-Object System.Drawing.Size(75,23)
$CancelButton.Text = "Cancel"
$CancelButton.Add_Click({$objForm.Close()})
$objForm.Controls.Add($CancelButton)

$objLabel = New-Object System.Windows.Forms.Label
$objLabel.Location = New-Object System.Drawing.Size(10,20) 
$objLabel.Size = New-Object System.Drawing.Size(280,20) 
$objLabel.Text = "Select environment: "
$objForm.Controls.Add($objLabel) 

$objListBox = New-Object System.Windows.Forms.ListBox 
$objListBox.Location = New-Object System.Drawing.Size(10,40) 
$objListBox.Size = New-Object System.Drawing.Size(260,20) 
$objListBox.Height = 90

$App | foreach-object{
[void] $objListBox.Items.Add($_.Name)
}

$objForm.Controls.Add($objListBox) 

$objForm.Topmost = $True

$objForm.Add_Shown({$objForm.Activate()})
[void] $objForm.ShowDialog()
Return $objChoice
}
# _______________________________________________________________________________ 


# Function Messagebox
# _______________________________________________________________________________ 
Function Show-MsgBox
 
    ($Text,$Title="",[Windows.Forms.MessageBoxButtons]$Button = "OK",[Windows.Forms.MessageBoxIcon]$Icon="Information"){
    [Windows.Forms.MessageBox]::Show("$Text", "$Title", [Windows.Forms.MessageBoxButtons]::$Button, $Icon) | ?{(!($_ -eq "OK"))}
}
# _______________________________________________________________________________ 



# Function Write-Console
# _______________________________________________________________________________ 
Function Write-Console {

    Param (
        [String]$String
        )

        $Time = Get-Date -Format T
        $Write = $time + " " + $String
        Write-Host -fore green $Write

    }
# _______________________________________________________________________________ 



# General Script Body
# _______________________________________________________________________________
Write-Console "Kill Appliction Executable"
Write-Console "Started on: $env:COMPUTERNAME"
Write-Console "By: $env:USERNAME"
 
$Environment = Get-Environment
Write-Console "Chosen environment: $Environment"
If((Show-MsgBox -Title "Chosen environmentis: $Environment" -Text "The Chosen environment is: <$Environment> Continue!?" -Button YesNo -Icon information) -eq 'No'){Exit}

else{
    $chosen = $App |select-object * | Where-Object{$_.Name -eq $Environment}
   }

# Set the local vars
$SQLServer = $chosen.SQLServer
$SQLInstance = $chosen.SQLInstance
$SQLDatabase = $chosen.Database
$ApplicionExe = $chosen.Executable
$Outfile = "C:\Scripts\ActiveComputers.txt"

# Create the new PowerShell session and pass the local vars to the remote sessions with $Using    
$Session = New-PSSession -ComputerName $SQLServer
$Connections = Invoke-Command -Session $Session -ScriptBlock { Invoke-SQLcmd -query "exec sp_who" -Serverinstance `
 $Using:SQLInstance -Database $Using:SQLDatabase }
Remove-PSSession $Session
# Sort active connections based on machinename
$Temp = $Connections |Where-object {$_.hostname -like "*Lab*"}
$ActiveHosts = $temp.hostname | sort-object -Unique

# Kill the Application process on each host
$Activehosts | Out-File $Outfile
Write-Console "To see on which hosts the application is will be closed, check: $Outfile"
If((Show-MsgBox -Title "Application will be closed" -Text "On $($ActiveHosts.count) desktop(s) The application will be closed. Continue!?" -Button YesNo -Icon Warning) -eq 'No'){Exit}
Foreach ($Comp in $ActiveHosts) {
    taskkill /s $Comp /IM "Notepad.exe" 
    Write-Console "Application.exe closed op: $Comp"
    }
If((Show-MsgBox -Title "Application closed" -Text "Application is closed on: $($ActiveHosts.count) desktop(s)." -Icon Information)){Exit}
```

[Go back](https://mufana.github.io/blog)
