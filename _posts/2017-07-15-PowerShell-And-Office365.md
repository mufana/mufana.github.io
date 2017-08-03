---
title: "PowerShell and Office365"
date: 2017-08-03
---

# PowerShell and Office365

## Background

A few days ago I was aksed to get a list of mailboxstatistics from a few or our users. Or to be more precise; a list of users with their LogonName,firstName,lastName,department,Title and the size of the mailbox. 

Now, I don't have lots of Office365 knowledge. However I'm equipped with the magic of PowerShell. So this had to an easy task. And if not; there's always 'Get-Help'.

## What to script

Before I could begin scripting I had to know what to look for in the Office365 enviroment. So the first step was to connect to the Office365 enviroment and play around with the available cmdlets.

Therefore I used a script that I had used before.

```PowerShell
$pass = ConvertTo-SecureString "Password" -AsPlainText -Force
$UserCredential = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList "UserID@onmicrosoft.com", $pass
$Session = New-PSSession `
            -ConfigurationName Microsoft.Exchange `
            -ConnectionUri https://outlook.office365.com/powershell `
            -Credential $UserCredential `
            -Authentication Basic `
            -AllowRedirection 
Import-PSSession $Session
```

This script connects to the Office365 environment and will import all necessary cmdlets for you in a #tmp container.

![image of temp](https://i0.wp.com/codeinblue.files.wordpress.com/2017/08/2.png)

Want to know which cmdlets are available in the module?

```PowerShell
Get-Command -Module tmp*
```

![Image of Module](https://i1.wp.com/codeinblue.files.wordpress.com/2017/08/3.png)

## Time to play

I noticed a cmdlet called 'Get-Mailbox'. That could be the one i'd need. 

```PowerShell
Get-Mailbox 'Mufana' | Select *
```

![Image of GetMailbox](https://i0.wp.com/codeinblue.files.wordpress.com/2017/08/4.png)

That gives me lots of information. But not quite what I'm looking for. 

Lets search for all cmdlets that start with the verb 'get' and have a noun with word 'mailbox' in it.

```PowerShell
Get-Command -verb get -noun mailbox*
```

![image get-command](https://i1.wp.com/codeinblue.files.wordpress.com/2017/08/6.png)

Nice! A potenial one. 'Get-MailboxStatistics'.

Lets give that puppy a try!

```PowerShell
Get-MailboxStatistics Mufana | select *
```

And yes! there it was! 'TotalItemSize'. Exactly what I'm after!

![image of size](https://i1.wp.com/codeinblue.files.wordpress.com/2017/08/5.png)

## Time to get to work

### Step 1

Create a list of all users I need to query. Since we're using Office365, I decided to get the userlist also from Office365 instead of our local Active Directory. Therefore I didn't had to query a user who existed in our local Active Directory but not in Office365. Yes, it happens. Hate it.

```PowerShell
(get-user -ResultSize $Resultsize |? {$_.recipienttype -eq "userMailBox"}).displayname | out-file C:\Temp\Userlist.txt"
```

### Step 2 - Importing the userlist to use in the script

```PowerShell
$Users = (Get-Content "C:\Temp\Userslist.txt")
```

 Create an empty array to hold the results. I almost always do this in every script I write. I'm a sucker for empty array's.

```PowerShell
$Results = @()
```

Now, I have to query two different objects within Office365.

* The User object. (I need the: firstName, lastName, title, Department and eMail address).

* The MailboxStatistics for the totalitemSize.

### Step 4 - Query time

#### First query

```PowerShell
Foreach ($usr in $Users) {$MailboxUsers = Get-User $usr | Where-Object {$_.recipienttype -eq "UserMailbox"}
```

What?
Well, for all the users in the userlist.txt, I create a new variable '$MailboxUsers' and I use that to store the output of the 'get-user' query. And, I only want users with a UserMailbox. We do have uers with no mailbox or shared mailboxes.

#### Second query

```PowerShell
Foreach ($user in $mailboxUsers) {
    $UserPrin = $user.userprincipalname
    $Stats = Get-MailboxStatistics $UserPrin
}
```

For all the users with a mailbox, I set the variable $UserPrin to the userPrincipalName and, I create a new variable called 'Stats' and use that to hold all the MailboxStatistiscs.

### Step 3 - Create a property table.

Create a table that holds all the relevant data.

```PowerShell
$Properties = @{
    Logon = $user.name
    Name = $user.displayname
    Department = $user.department
    eMail = $user.windowsemailaddress
    LoggedIn = $MbxStats.LastLogonTime
    MailboxSize = $MbxStats.totalitemsize
} # End Property Block
```

Next we create a new psobject with the property block.

```PowerShell
$Results += New-Object psobject -Property $properties
```

### Step 4 - Putting it all together

The last step is to put all things together and build the output. As a bonus we convert the mailboxsize to a more 'friendly' output in GB. 

```PowerShell
$Results | Select-Object Logon,Name,Department,eMail,LoggedIn,@{name=”MailBoxSize (GB)”; expression={[math]::Round( ` ($_.mailboxsize.ToString().Split(“(“)[1].Split(” “)[0].Replace(“,”,””)/1GB),2)}}
```

## Wrapping things up

Export these results to a csv and the result is almost magical!

![Image of result](https://i2.wp.com/codeinblue.files.wordpress.com/2017/08/7.png)

[Go back](https://mufana.github.io/blog)
