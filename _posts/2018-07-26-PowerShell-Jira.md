---
title: "PowerShell and Jira"
date: 2018-07-26
---

# PowerShell and the Jira API (Part 1)

## Background

I'm currently working on a PowerShell project to create users in Atlassian Jira. For those who don't know Jira, Jira is a software suite that contains an IT ServiceDesk Tool, Project and Issue tracking. Either in the cloud or on-premise. You can sign up for a free trial and check it out. https://www.atlassian.com/
  
For my project, I will be using the Jira cloud products. And that means, leveraging the API. 

Scripting against an API sometimes really is a matter of perseverance. Of course, extracting data from an API is easy. Especially when the API is well documented. As is the case with the Jira API.
However, (there always a big if...) you do need some experience with software development, or at least have a basic idea of how things work.  

This time, I'm intended to write a PowerShell tool to create users in Jira. Now, for most IT admins, a relatively simple script will do the job. And in most cases, I don't worry too much about error handling. In this case, my tool will be used by end users. So I need hardcore error handling. 

Error handling adds another layer of complexity to your scripts. With software development, about 90% of the code is error handling. 

In this blog series, I'll take you on a journey to develop a well-written piece of software. This is the very first part of this series.

### What do we want?
Our Jira environment is cloud-based. There's no connection with AzureAD or Active Directory. The goals:

* Create users in Jira with PowerShell,
* Make sure the tool is 'dummy' proof. Meaning, we will handle all errors.
* Create a parameterized build with the help of Jenkins.

#### Approach

When it comes to developing software, It's smart to think things through. Developing software is almost like creating a painting. Van Gogh probably didn't start painting the sunflowers stamen, stigma and whatever parts a sunflower contains. He probably started by drawing a frame and a rough outline of the sunflowers before focussing on the details. This approach leaves room for discussion and adjustments as you go.

### Best practices

Furthermore, I use PowerShell best practices. Meaning, I will follow the natural PowerShell flow as much as possible. I use standard verb-nouns and parameter names. If I have to re-use the code in my tool, I will write a function. 

## Preparation

Every journey begins with reading. Getting a sense of the places to visit, what to do (and what not). What kind of language is spoken, how to get in. Do I need apply for a VISA or will my passport suffice?  

### Step 1 - The API documentation

The API documentation is kinda like the Lonely Planet. Here I can find out how the API works, how to authenticate and how I can make different kinds of requests to it.  

You can find it here: https://developer.atlassian.com/cloud/jira/platform/rest/

#### How to get data from the API

First, I will focus on the user section. And, to make my life a little bit easier, I'll start with; [Get user](https://developer.atlassian.com/cloud/jira/platform/rest/#api-api-2-user-get)

In the documentation I notice a there are a few parameters I can use to query a particular user. In this case; ```Key```, ```Accountid```, ```expand```, ```Username```. The response I should get back is in the form of JSON. Good thing to remember.

To help me a little bit, Atlassian provided a curl example. 

But 'curl' isn't PowerShell! Right!? 

Well almost! 

![curl](https://codeinblue.files.wordpress.com/2018/07/1.png)

You'll see that 'curl' translates to; ```Invoke-WebRequest```. But, there's another cmdlet called: ```Invoke-RestMethod```. 

Now both of these cmdlets are wrapped around the same kind of functionality. The difference is that they both output differently. ```Invoke-WebRequest``` is better dealing with HTML. And; ```Invoke-RestMethod``` does a better job with XML or JSON. You have to experiment a little bit to find out which works best. 

Since I'm expecting a JSON response, I'll go with; ```Invoke-RestMethod```. 

#### How to authenticate

I need to know how to authenticate to the API. According to the documentation, there are currently three methods of authentication.

* oAuth
* Cookie based
* Basic (Username/Password)

For the examples in this blog series, I use the basic authentication method.

### Step 2 - Play around 

Now I know what to expect, it's time to play around. Normally I'll start my coding journey in the PowerShell Console. But, for this project, I will make an exception and go directly to the ISE.

#### Authentication and getting the first result

I already created the user: _John Doe_ in Jira. And, I'm curious to see how dear old John looks in PowerShell.

![JiraResult](https://codeinblue.files.wordpress.com/2018/07/2018-07-26-19_44_31-microsoft-edge.png)

First, I need to create an authentication code block.

```powershell
$Username = "" 
$Password = "" 

$Token = "Basic " + [System.Convert]::ToBase64String([System.Text.Encoding]::ASCII.GetBytes("$($Username):$Password"))
```

The above block gets the Username and Password for my Jira site, converts them to a Base64String and stores them in a variable called; ```$Token```. 

The string; ```$Token + Basic +``` is exactly what Jira expects. See the [basic authentication documentation.](https://developer.atlassian.com/cloud/jira/platform/jira-rest-api-basic-authentication/)

Next, I'll define a variable called; ```$Header```. This includes the token. 

```powershell
$Header = @{Authorization = $Token}
```

Next, I'll define the URL of my Jira site. 

```powershell
$jiraUri = "https://yourjirasiteURL/rest/api/2/user/search?username=JohnDoe"
```

And last; ```Invoke-RestMethod```.

```powershell
Invoke-RestMethod -Uri $jiraUri -Method Get -Headers $Header
```

And voilá there's my JSON result. 

![Result](https://codeinblue.files.wordpress.com/2018/07/2018-07-26-19_42_44-powershell-integrated-scripting-environment-5-1-17134-165-__-in-code-we-trust-__.png)

Cool! But, John seems a bit lonely. So, let's get him settled with Jane. 

To create Jane, I can use pretty much the same code as above. Except, I need a body with the user parameters. This body has to be in JSON format.

And, instead of a _get_ method, I'll use _post_.

```powershell
$Username = "" 
$Password = "" 

$Token = "Basic " + [System.Convert]::ToBase64String([System.Text.Encoding]::ASCII.GetBytes("$($Username):$Password"))
$Header = @{Authorization = $Token}

$userCreationBody = @{
    name = "Jane Doe"
    password = "123131"
    emailAddress = "JaneDoe@somedomain.com"
    displayName = "Jane Doe"
} | ConvertTo-Json

$jiraUri = "YourJiraURL"

Invoke-RestMethod -Uri $jiraUri -Method post -Headers $header -Body $UserCreationBody -ContentType application/json
```

And if Jane is alive, I get back a JSON response.

![response](https://codeinblue.files.wordpress.com/2018/07/2018-07-26-19_56_39-powershell-integrated-scripting-environment-5-1-17134-165-__-in-code-we-trust-__.png)

And the result in Jira.

![Jira1](https://codeinblue.files.wordpress.com/2018/07/2018-07-26-19_57_21-microsoft-edge.png)

Nice! That wasn't to hard!

### Step 3 - The master plan

The next step is to come up with a plan. There are lots of different ways to create a plan. And neither is good or bad. It's more personal preference. I always use some sort of mind mapping tool to make things a little more visual. 

The tool will be used to create users in Jira and nothing more. To create users in Jira, I also want to make sure that the tool only creates users that don't exist already. _I don't want to create another Jane Doe, since this might get awkward for John!_.  

Another important thing to consider is; when a user is created in Jira, an invitation email will be sent to the email address belonging to that user. So, the email address needs to unique.

#### Gather all users from Jira

At some point, I want to gather all existing users from Jira and store them; either in a variable or a CSV (or whatever file). So that I can make sure I only create users who are unique.

#### Limit the use of Invoke-RestMethod

Each time I use an Invoke, I will make a call to a remote system. Each time I going to make a call, things can go wrong. I want to handle as much as possible on a local machine. 

#### Each call must be wrapped inside a Try/Catch block. 

I need hardcore error handling. Also, I need to figure out all errors I might get back from Jira. And, when they are completely understandable for end users, I want to throw my own custom errors. I might have to write a custom error handling function.

My script needs to 'flow' and never jump out of it's routine. This tool might be used to create 100 users in Jira at the same time. We don't want the script to break if there's an error. 

#### Logging

Error handling also means, logging. What happens from a -> z. If an error occurred, where did the error occur and most importantly; Why? Logging actually is a form of storytelling. Guiding the end user through the process and make sure they can figure out where things went wrong and why.

## Wrap-up

Now, this was an exceptional long introduction. I decided to split-up this blog series because I don't want to overload you with information.
Be aware of part 2. 