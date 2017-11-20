---
title: "PowerShell and Office365"
date: 2017-08-03
---

# PowerShell and Office365

## Background

A few days ago I was aksed to get a list of mailboxstatistics from a few or our users. Or to be more precise; a list of users with their LogonName,firstName,lastName,department,Title and the size of the mailbox.

## What to script

Before I could begin scripting I had to know what to look for in the Office365 enviroment. So the first step was to connect to the Office365 enviroment and play around with the available cmdlets.

Therefore I used a script that I had used before.

```powershell
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

This script connects to the Office365 environment and will import all necessary cmdlets for you in a tempory module.

![image of temp](https://i0.wp.com/codeinblue.files.wordpress.com/2017/08/2.png)

Want to know which cmdlets are available in the module?

```powershell
Get-Command -Module tmp*
```

![Image of Module](https://i1.wp.com/codeinblue.files.wordpress.com/2017/08/3.png)

## Time to play

I noticed a cmdlet called 'Get-Mailbox'. That could be the one i'd need.

```powershell
Get-Mailbox 'Mufana' | Select *
```

![Image of GetMailbox](https://i0.wp.com/codeinblue.files.wordpress.com/2017/08/4.png)

That gives me lots of information. But not quite what I'm looking for.

Lets search for all cmdlets that start with the verb 'get' and have a noun with word 'mailbox' in it.

```powershell
Get-Command -verb get -noun mailbox*
```

![image get-command](https://i1.wp.com/codeinblue.files.wordpress.com/2017/08/6.png)

Nice! A potenial one. 'Get-MailboxStatistics'.

Lets give that puppy a try!

```powershell
Get-MailboxStatistics "Jeroen" | select *
```

And yes! there it was! 'TotalItemSize'. Exactly what I'm after!

![image of size](https://i1.wp.com/codeinblue.files.wordpress.com/2017/08/5.png)

## Time to get to work

### Step 1

Create a list of all users I need to query. Since we're using Office365, I decided to get the userlist also from Office365 instead of our local Active Directory. Therefore I didn't had to query a user who existed in our local Active Directory but not in Office365.

```powershell
(get-user -ResultSize $Resultsize |? {$_.recipienttype -eq "userMailBox"}).displayname | out-file C:\Temp\Userlist.txt"
```

The above cmdlet exports the users displaynames to a txt file.


### Step 2 - Importing the userlist to use in the script

```powershell
$Users = (Get-Content "C:\Temp\Userslist.txt")
```

 Create an empty array to hold the results. I almost always do this in every script I write. I'm a sucker for empty array's.

```powershell
$Results = @()
```

Now, I have to query two different objects within Office365.

* The User object. (I need the: firstName, lastName, title, Department and eMail address).

* The MailboxStatistics. (TotalItemSize)

### Step 4 - Query time

#### First query

For all the users in the userlist.txt, I create a new variable '$MailboxUsers' and I use that to store the output of the 'get-user' query.

```Foreach ($usr in $Users) {$MailboxUsers = Get-User $usr```

Since Office365 also has ShardMailBoxes etc... I want to make sure to only export the UserMailBoxes. ```| Where-Object {$_.recipienttype -eq "UserMailbox"}```

And, I only want users with a UserMailbox. We do have users with no mailbox or shared mailboxes. And, 'Get-User' has all the data (department, title etc...) that I need.

```powershell
Foreach ($usr in $Users) {$MailboxUsers = Get-User $usr | Where-Object {$_.recipienttype -eq "UserMailbox"}
```

#### Second query

For all the users with a mailbox, I set the variable $UserPrin to the userPrincipalName. Next I create a new variable called 'Stats' and use that to hold the data from the 'Get-MailboxStatistics' cmdlet.

```powershell
Foreach ($user in $mailboxUsers) {
    $UserPrin = $user.userprincipalname
    $Stats = Get-MailboxStatistics $UserPrin
}
```

### Step 3 - Create a property table.

Create a table that holds all the relevant data. 

```powershell
$Properties = @{
    Logon = $user.name
    Name = $user.displayname
    Department = $user.department
    eMail = $user.windowsemailaddress
    LoggedIn = $MbxStats.LastLogonTime
    MailboxSize = $MbxStats.totalitemsize
} # End Property Block
```

Now that I have all my properties sorted out, I have to create a new PSObject for each user in the userlist.
(I can finally get to use that empty array!)

```powershell
$Results += New-Object psobject -Property $properties
```

### Step 4 - Putting it all together

The last step is to put all things together and build the output. As a bonus we convert the mailboxsize to a more 'friendly' output in GB. 

```powershell
$Results | Select-Object Logon,Name,Department,eMail,LoggedIn,@{name=”MailBoxSize (GB)”; expression={[math]::Round( ` ($_.mailboxsize.ToString().Split(“(“)[1].Split(” “)[0].Replace(“,”,””)/1GB),2)}}
```

## Wrapping things up

Export these results to a csv/excel document, send it to my manager and I'm good!

![Image of result](https://i2.wp.com/codeinblue.files.wordpress.com/2017/08/7.png)

I'm currently writting an Office365 module designed for multiple tasks. So make sure to come back!

[Go back](https://mufana.github.io/blog)
