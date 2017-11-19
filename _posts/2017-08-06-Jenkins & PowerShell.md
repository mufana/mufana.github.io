---
title: "Jenkins & PowerShell"
date: 2017-08-06
---

# Jenkins & PowerShell

## Background

Recently I found the [blog](https://hodgkins.io/automating-with-jenkins-and-powershell-on-windows-part-1) from 'Matthew Hodgkins' about setting up a Jenkins server and do stuff with PowerShell. Cool stuff! so definitely read that post!

### Use cases

Currently I'm working on a few projects that involve loads of PowerShell scripting. One of my tasks is to automate the creation of a users homefolder on our DFS shares. This task is 'at this moment' something that has to be carried out manually. Or; One has to start a VBscript tool that does the job. And even though it works fine, it's not what we want. My company has about 5000 employees. At times, more then a 100 people are hired at the same time. Now, I don't want to start a tool 100 times.

### Here I come to save the day!

And that's where PowerShell comes to the rescue once again.
Imagine a script (currently writing a blog about it) that creates homefolders and sets the owner rights!? Imagine! wouldn't that be a sight! Truth be told. No it wouldn't. Since this is something everybody (with a little common sense) would do. Unless you love to do the same trick over and over.

### Why not use Windows Taks Scheduler

So, we have the script that does the job for us. Next it would be easy to schedule with the Task Scheduler on a server. 

But... where is the fun in that? 

Wouldn't it be way cooler to schedule such a script on a automation platform that is actually build for this? 

Of course! and there are more arguments to do just that.

* Scripts need to be maintained. Jenkins has Git support. 
* Pester tests can also be stored and executed from Jenkins.
* Anybody (with access rights to Jenkins) can excecute the script.

Last, my experience with scripts is that they literally are all over the place. On every server in your infrastructure, scripts are stored. At some point nobody knows what they do, if they still work, who wrote them. And; what if something in your infrastructure breaks and you know it's caused by a script!? 

Now is definitely the time to work professionaly and use some form of Source Control or at least, store all your scripts in one place and describe what they do. (or break).

### Or, use Git and integrate Git with an Automation Server

Since I have to automate a small portion of my company's 'onboard and exit' process, there are lots of scripts to be build. (I know, Identity Management and RBAC? Yes! we have that. Limited.) And, I do want to do this the right way. Meaning:

* Write pester tests for all my scripts
* Write the Help documentation
* Use Source Control
* Store and execute all scripts from one server

## So, Jenkins it is!

First I read Matthew's blog about Jenkins and PowerShell to get me started. You're up & running in under 5 minutes. Okay, takes a little longer if you want to do this in Docker.

When you login to Jenkins you will see something similair to this:

![Image of Jenkins](https://codeinblue.files.wordpress.com/2017/08/j1.png)

Now, I created the folowing items:

* Create-File
* HomeFolders

Create-File is the task from Matthew's example!

### Setup a schedule

For my 'HomeFolder' task I created a trigger. We don't want to manually run this task every week. Now; Jenkins triggers are a little 'tricky'. It begins with the minutes (42), then the hour (20) and last the days of the week. (1-7 equals every day of the week).

![Image of trigger](https://codeinblue.files.wordpress.com/2017/08/j2.png)

### The PowerShell code

Next, I entered my PowerShell code. 

![Image of code](https://codeinblue.files.wordpress.com/2017/08/j3.png)

And we're done. All we have to to is wait for the scheduler to execute. Should be start executing in a matter of seconds.

### 7200 seconds later!

Ah ok... I set the scheduler a little to tight!
But, it executed. 

![Image of execute](https://codeinblue.files.wordpress.com/2017/08/j4.png)

## Error handling

It is wise to do some sort of error handling in your script. Otherwise, Jenkins will show every run as a 'success'. Even if the script generated an error. 

## Wrapping things up

So, I think this might be a nice solution for my and my collegue's and an important step forward in order to maintain PowerShell scripts.

But, I'm open for suggestions! 

[Go back](https://mufana.github.io/blog)
