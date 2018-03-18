---
title: "The stale account"
date: 2018-03-18
---

![Car](https://images.pexels.com/photos/532001/pexels-photo-532001.jpeg)

# Case File NO "12043412" (The 'stale' account)

__The PowerShell code in this blog has been tested with PowerShell 5.1__

Remember those days when a person leaves your company, someone from the HR department makes a few phonecalls to the helpdesk and asks them to delete the account and user data. Few days later, someone from the IT department manually deletes the Active Directory account and archives the homefolder. 

The sheer nostalgia of the 90s! 

Unfortunatley there are lots of company's out there, keeping the 90's very much alive and kicking. 

## FFWD to 2018

Nowadays, there are far better ways to clean-up stale accounts. It's either a part of some Identity Management solution or automated with PowerShell. 

## Stand back kids! Daddy's gonna show you how to automate!

What I want to achieve with PowerShell:

* Gather a list of stale accounts from Active Directory.
* Either delete or disable the account.
    * Delete the Active Directory account.
    * Disable it and move it to a different Organizational Unit.
* Delete (or archive) all relevent userdata.

The first step is to define a 'stale' account and what to do with them.

To be honest; that is not so easy. Simply because 'stale' could mean anything. If someone were to ask me to define a 'stale' account, I'd refer them to the HR or Security department.

_Example_

What if 'stale' means that the account has not been used for 3 (or more) months. And that it should be disabled, moved to a different AD Organizational Unit and that all userdata should be removed. 

## Step 1: How do I look

Let's see what a 'stale' or expired account looks like in Active Directory. 

![Good](https://codeinblue.files.wordpress.com/2018/03/11.png)

It seems that the _Account Expiration_ attribute is set. In this case it's set to: _Thursday, November 30 2017_. 

Checking the same attribute with PowerShell:

```powershell
get-aduser -identity 103 -properties * | select accountexpires
```

![What?](https://codeinblue.files.wordpress.com/2018/03/21.png)

Unfortunate, the date is shown in the LDAP\FileTime format. 

_The timestamp is the number of nanoseconds since Jan 1, 1601._ 

To convert it a 'normal' format:

```powershell
Get-Date 131565564000000000 -uFormat %x
```

![NormalDate](https://codeinblue.files.wordpress.com/2018/03/31.png)

And vóila! the date is shown as: ```11/30/2017```. 

So, what do I need to code?

* Gather expired accounts (3 or more months) from Active Directory.

* Check if there's is a homefolder for the Active Directory account.

* Disable the AD account.

* Move it to a different OU.

* Delete the homefolder.

## Coding time

Now that I have a basic idea what code, it's time to move over to the ISE. _(For large projects, I highly recommend Visual Studio Code. https://code.visualstudio.com/)_ 

## Step 2: The first code block

First step is to find all accounts that are expired for 3 (or more) months. So, the first line of code is: ```(Get-Days).addDays("-90)```. Which means; _Subtract 3 months from the current date._
We store that in a variable called; ```$Date```.

```powershell
# Set the search date of the: collExpiredEmployees
$Date = (Get-Date).AddDays("-90")
```

Next is to define the attributes I want to export with the _Get-ADuser_ cmdlet in Active Directory and set the location of the Homefolder. Could be either a DFS or local share.

```powershell
# Home/Profile location of the DFS share
$HomeDir    = "C:\DFSRoots\Home"

# Splat params for Get-ADUser
$Params = @{
    LDAPFilter = "(&(objectCategory=person)(objectClass=user))"
    SearchBase = "ou=users,ou=corp,dc=contoso,dc=com"
    Server     =  "contoso.com"
    Properties = @("SamAccountName", "GivenName", "Surname", "userPrincipalName", "AccountExpirationDate", "AccountExpires", "lastLogon", "PasswordExpired", "Enabled", "CanonicalName") 
}

# Define empty results array
$Results = @()

# Run the query on Active Directory and search for users whose accounts has expired for at least 3 months
$collExpiredEmployees = (Get-AdUser @Params) | Where-Object {$_.AccountExpires -le "$($Date.ToFileTimeUtc())"} 
```

### Break it down

So, what happens in the code block above?

#### $Params block

First, the ```$Params``` block. You might notice that this block contains all the parameters for the; ```get-aduser``` cmdlet. Although they are stored in a hashtable!?

#### Get-ADUser

Second there's this: ```(Get-AdUser @Params)```

This method is called _splatting_. A different way to parse parameters to a cmdlet. But how does it work?

To explain that, the first thing we have to remember is that a variable is always defined by the: __$__ sign. As is the case with the: ```$Params``` block. 

The __@__ sign is a little weird because it's used in multiple ways. 

1. To define a new hashtable. ```$Params = @{}```
2. As an operator. ```@Params```

The operator __@__ sign behaves _almost_ the same as the __$__ sign. However, there are four actions taking place beneath the hood. 

1. Treat everything behind the __@__ sign as a variable.
2. The content of the variable as a hashtable.
3. The keys as parameternames.
4. The values as the value of the parameters.

In the end, ```Get-AdUser @Params``` is actually the same as: ```Get-ADUser -LDAPfilter "Value" -Server "Value"```

#### Filter

Third, there's a filter. ```{$_.AccountExpires -le "$($Date.ToFileTimeUtc())"}``` Remember my ```$Date``` variable? This filters out only accounts accounts expired for at least 3 months.  

#### The empty array

Last, there's an empty array. ```$Results = @()```

## Step 3: The second code block

In the next block of code, we loop through all users within the ```$collExpiredEmployees = (Get-AdUser @Params)``` variable. 

```powershell
# Loop through users
Foreach ($Employee in $collExpiredEmployees)
{
```

## Step 4: Check home/profile directory

The next step is to check for home and 
profiledirectories.

The most obvious way to test for existence of a folder is by using the ```Test-Path``` cmdlet. This returns a value of __$True__ or __$False__. 

![True](https://codeinblue.files.wordpress.com/2018/02/d8.png)

Which is nice, but I also want the fullname of the folder. 

In this case I'll go with ```Get-Item```. This is actually a general cmdlet. Useful in many places. 

![Name](https://codeinblue.files.wordpress.com/2018/02/d9.png)

```Get-Item``` throws an error if the folder does not exists. 

![Name](https://codeinblue.files.wordpress.com/2018/02/d10.png)

Using the ```fullname``` method, we get the fullname / path of the directory.

![Name](https://codeinblue.files.wordpress.com/2018/02/d11.png)

Step 5: Convert LDAP time to UTC

We need to convert the AccountExpirationDate to a readable format. As a bonus, we also convert the lastLogon time. 

```powershell
    # Convert the FileTime to UTC Time
    $Expires = ($Employee.AccountExpires)
    Foreach ($ExpirationDate in $Expires) {$ExpiredDate = (Get-Date $Expires -UFormat %x)}

    # Convert the FileTime to UTC Time
    $LastLogon = ($Employee.lastLogon)
    Foreach ($Logon in $LastLogon) {$LastDate = (Get-Date $Logon -UFormat %x)}
```

## Step 6: Create new objects

_Always try to think ahead when you are creating a script. In this case, you might want to export all the information to an excel file in case the security or HR department asks for it._

Remember my empty array!? 

Because I want to export all the information to an Excel file or website, I have to create some sort of table. And of course, we can pipe ```Get-AdUser``` to ```Export-Csv```. But, if you have to send out this information to non-technical people in your organisation, you might want to amke the output more _userfriendly_. 

Therefore I create a new hashtable with _userfriendly_ names.

```powershell
# Create the property block
$Props = @{
    EmployeeID                            = $Employee.samAccountName
    FirstName                             = $Employee.GivenName
    LastName                              = $Employee.Surname
    Active                                = $Employee.Enabled
    "Account active until: (mm-dd-yyyy)"  = $ExpiredDate
    Homedirectory                         = $CheckHomeDir
}
```

And create a new object for each user with an expired account.

```powershell
# Create new psobject for each user
$Results += New-Object psobject -Property $Props
 ``` 

## Step 7: Create a function

Ok, up to this point we have a script that gathers information from Active Directory. But I also want to cleanup accounts and folders with the same tool. 

Therefore, I wrapped my code in a function and added __[Switch]Parameters__. 

```powershell
Function Get-ExpiredAccount {
 
    [cmdletbinding()]
 
    Param (
        [Switch]$Contoso,
    )

    # Set the search date of the: collExpiredEmployees
    $Date = (Get-Date).AddDays("-90")

# Switch organisation contoso
    If ($Contoso)
    {
        # Home/Profile location of the DFS share
        $HomeDir    = "C:\DFSRoots\Home"
 
        # Splat params for Get-ADUser
        $Params = @{
            LDAPFilter = "(&(objectCategory=person)(objectClass=user))"
            SearchBase = "ou=users,ou=corp,dc=contoso,dc=com"
            Server     =  "contoso.com"
            Properties = @("SamAccountName", "GivenName", "Surname", "userPrincipalName", "AccountExpirationDate", "AccountExpires", "lastLogon", "PasswordExpired", "Enabled", "CanonicalName") 
        }
    }
```

Why the __Contoso__ switch? well, there are organisations with an Active Directory forrest that has multiple child domains. You can add multiple switches corresponding with other domains. 

## Step 8: Add another parameter

```[Switch]$CollectUsers```

I've added another parameter called: __$CollectUsers__. 

This block gathers the information from Active Directory and creates a nice hashtable.

```powershell
# Collect expired users
If ($CollectUsers)
{
    # Select the results (parameters) to export 
    $Select = @("EmployeeID", "FirstName", "LastName", "Active", "Account active until: (mm-dd-yyyy)", "Homedirectory")

    # Loop through users
    Foreach ($Employee in $collExpiredEmployees)
    {
        $UserID = $Employee.SamAccountName

        # Check if the employee has a homedirectory
        $CheckHomeDir = (Get-Item "$HomeDir\$UserID*")
            
        # Convert the UnixDate to UTC Time
        $Expires = ($Employee.AccountExpires)
        Foreach ($ExpirationDate in $Expires) {$ExpiredDate = (Get-Date $Expires -UFormat %x)}

        # Convert the UnixDate to UTC Time
        $LastLogon = ($Employee.lastLogon)
        Foreach ($Logon in $LastLogon) {$LastDate = (Get-Date $Logon -UFormat %x)}
    
        # Create the property block
        $Props = @{
            EmployeeID                            = $Employee.samAccountName
            FirstName                             = $Employee.GivenName
            LastName                              = $Employee.Surname
            Active                                = $Employee.Enabled
            "Account active until: (mm-dd-yyyy)"  = $ExpiredDate
            Homedirectory                         = $CheckHomeDir
        }

        # Create new psobject for each user
        $Results += New-Object psobject -Property $Props
    }  
}

# Display results on screen
$Results | Select $Select
```

## Step 9: Time to move

For a large part, the PowerShell code to actually move user accounts and delete homefolders is the same. 
I did however, added a few extra lines.

### Check homedirectory

To remove homedirectories I first have to make sure there is one. So, that's the first line of code. ```If ($CheckHomeDir.Exists -eq $True)``` Do something.

And then, you'll see something weird. __Robocopy__ _"I thought you wanted to delete folders?"_ And I do. But, the problem here is the 256 characterlimit of Windows. And that's something even PowerShell can't work around. 

The trick is to use Robocopy. 

1. Create an empty directory. In this example I've used: ```C:\Temp\Empty```
2. Exectute: ```robocopy C:\temp\Empty PathToTargetDir /MIR```

This mirors the empty directory with the targetdirectory. And it will automatically purge all files without any problems. Even the files that exceed 256 characters. 

```powershell
# Delete the homedirectory
If ($CheckHomeDir.Exists -eq $True)
{
    # Remove directory structure
    robocopy C:\temp\Empty $CheckHomeDir /MIR

    # Remove directory
    Remove-Item $CheckHomeDir
    $Checkhomedir = "Deleted"
}
```

### Disable and move the AD account

Next is to disable the useraccount and move it to another OU.

```powershell
# Disable the useraccount
Set-Aduser $UserID -Enabled $False -WhatIf

# Move the Account to different OU
Move-ADObject -Identity "CN=$UserID,ou=Users,ou=corp,dc=contoso,dc=com" -TargetPath $Target
```

### Switch it

Last, I created another parameter called: __CleanUp__ and I've added another custom array to get a report. 

```powershell
# CleanUp the home/profile directories and disable expired accounts
If ($Cleanup)
{
    # Select the results (parameters) to export 
    $Select = @("EmployeeID", "FirstName", "LastName", "Active", "Account active until: (mm-dd-yyyy)", "Homedirectory","Location")

    # Set the tartget path for the move
    $Target = "ou=InActive,ou=corp,dc=contoso,dc=com"

    # Loop through users
    Foreach ($Employee in $collExpiredEmployees)
    {
        $UserID = $Employee.SamAccountName

        # Check if the employee has a homedirectory / profiledirectory
        $CheckHomeDir = (Get-Item "$HomeDir\$UserID*")

        # Delete the homedirectory
        If ($CheckHomeDir.Exists -eq $True)
        {

            # Remove directory structure
            robocopy C:\temp\Empty $CheckHomeDir /MIR

            # Remove directory
            Remove-Item $CheckHomeDir -WhatIf
            $Checkhomedir = "Deleted"
        }

        # Disable the useraccount
        Set-Aduser $UserID -Enabled $False -WhatIf

        # Move the Account to different OU
        Move-ADObject -Identity "CN=$UserID,ou=Users,ou=corp,dc=contoso,dc=com" -TargetPath $Target

        # Get-ADUser
        $Collect = Get-ADUser -Identity $UserID -server "contoso.com" -Properties *

        Foreach ($User in $Collect)
        {

            # Convert the UnixDate to UTC Time
            $Expires = ($User.AccountExpires)
            Foreach ($ExpirationDate in $Expires) {$ExpiredDate = (Get-Date $Expires -UFormat %x)}

            # Create a new property block
            $Props = @{
                EmployeeID                           = $User.samAccountName
                FirstName                            = $User.GivenName
                LastName                             = $User.Surname
                Active                               = $User.Enabled
                "Account active until: (mm-dd-yyyy)" = $ExpiredDate
                Homedirectory                        = $CheckHomeDir
                Location                             = $User.DistinguishedName
            }

            $Results += New-Object psobject -Property $Props
        }
    }

        # Display results on screen
        $Results | Select $Select
}
```

## Finalize

I saved my script as: ```Get-ExpiredAccounts```

When I run it and type: ```Get-ExpiredAccount -Contoso -CollectUsers```
It will generate an nice report with all the information I need.

![Collect](https://codeinblue.files.wordpress.com/2018/03/41.png)

And using the: ```Get-ExpiredAccount -Contoso -CleanUp```
Cleans up the Active Directory and homefolders. And, as a bonus, also creates a nice report.

![Clean](https://codeinblue.files.wordpress.com/2018/03/51.png)

# Wrapping up
This is just one way to clean-up your Active Directory and DFS shares. And it's just a basic idea. But it gives an idea if you ever need to implement something similair. 

Get the full script [here](https://raw.githubusercontent.com/mufana/PowerShell/master/Get-ExpiredAccount.ps1)

![del](https://media.giphy.com/media/vohOR29F78sGk/giphy.gif)

## # Case File NO "12043412" (The 'stale' account) - SOLVED

[Go back](https://mufana.github.io/blog)