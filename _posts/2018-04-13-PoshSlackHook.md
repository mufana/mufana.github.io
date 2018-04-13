---
title: "PowerShell and a Slack API webhook"
date: 2018-04-13
---

![Hook](https://images.pexels.com/photos/358454/pexels-photo-358454.jpeg?cs=srgb&dl=bay-beach-boat-358454.jpg&fm=jpg)

# PowerShell and a Slack API webhook

I'm currently working on a few PowerShell projects that require me to send a message to slack. Previously I had done this with the help of the PowerShell [PSSlack](http://ramblingcookiemonster.github.io/PSSlack/) module written by: __@RamblingCookieMonster__. But, since that required me to setup a few things and install modules on servers, I was looking for a much simpler way to send a slack message.

Welcome to the world of the webhook!

_Scripting against an API might seem a little daunting. But, with so many API's out there, it's almost impossible to avoid them. And it's really not that hard._

## WebHooks and API's

What is a _webhook_ ? well, it's an API. Nothing more. However, there's a difference. When using an API, a request is send to the API, the request is then handled by the API and a response will be send. A webhook just sends something to a webhook service. Webhooks require you to register a URL providing the service.

## My first API

As I mentioned earlier, there are lots of API's out there. For instance the: [People in Space API](http://open-notify.org/Open-Notify-API/People-In-Space/). Using this API we can find out how many people are in space and who they are. 

The URL for this API is: ```http://api.open-notify.org/astros.json```

1. Open the PowerShell ISE.

2. Enter: ```Invoke-RestMethod -Method```
Now, intelisense shows you a few methods, the: ```Get``` is important, since we want to extract information from the API.
![Slack3](https://codeinblue.files.wordpress.com/2018/04/slack-3.png)

3. Enter the URL of the API. ```-uri```.

```powershell
Invoke-RestMethod -Method get -uri http://api.open-notify.org/astros.json)
```

4. Run the code and be proud! you've just extracted information from an API.

![Slack4](https://codeinblue.files.wordpress.com/2018/04/slack-4.png)

This is a basic example how to get information from an API with PowerShell. Not that hard was it!?

### Back to Slack...

## Setup the WebHook integration for Slack

1. The first step is to create a webhook integration for [your channel](https://my.slack.com/services/new/incoming-webhook/)

3. Select a channel, (In this example I've used my _#test_ channel) and click on: ```Add Incoming Webhooks Integration```.

![slack1](https://codeinblue.files.wordpress.com/2018/04/slack-1.png)

4. Copy the URL displayed in the ```Webhook URL``` section. It's starts with: _https://hooks.slack.com_.

![Slack2](https://codeinblue.files.wordpress.com/2018/04/slack-2.png)

## Read the manual 

Now that we have our webhook integration, what's next? 
Well, it all comes down to reading the documentation. We have to figure out how the webhook works before we can use it.

You will find it [here.](https://api.slack.com/incoming-webhooks#sending_messages)

At first glance you might think: _these are all _curl_ examples, there's no PowerShell, I have no idea what to do, what am I getting into! Where's Mum?_

Things to remember: 
1. Obviously we're not going to use ```curl```. Instead, we'll be using ```Invoke-RestMethod```.

2. ```POST``` is another method. Like: ```GET```.

3. We have to send a JSON payload that includes our message.

### Step 1 - Define the JSON payload

Open your favorite IDE (In this example I'm using the ISE. For large projects I recommend VScode.)

1. Create a new variable with a here-string

```powershell
$Payload = @"
"@
```

2. Add the JSON payload as described in the documentation.

```powershell
$Payload = @"
    {
        "text": "This is a line of text.\nAnd this is another one."
    }
"@
```

### Step 2 - Send the payload

1. To send it to the webhook, again we use ```Invoke-RestMethod```. This time however, we use the method ```post```.

__Don't forget to modify the URL to the URL of your own Slack webhook.__

```powershell
Invoke-RestMethod -Uri "https://hooks.slack.com/services/_YOUR-URL-HERE_" -Method Post -Body $Payload
```

A few seconds later, a message pops-up with our message.

![Slack-5](https://codeinblue.files.wordpress.com/2018/04/slack-5.png)

## Wrap-Up

I've created a very simple function to send messages to Slack just to give an idea of the possibilities when using the webhook. For instance, checking if an important service on your servers is running. And, if not, send a message to slack.

```powershell
Send-SlackNotification -uri "https://" -Message "The Windows Service 'Bits' is stopped on: Server001" -Status failure
```

![slack-6](https://codeinblue.files.wordpress.com/2018/04/slack-6.png)

```powershell
Function Send-SlackNotification
{
    [cmdletbinding()]

    Param
    (
        $Uri = "",
        [String]$Message,
        [String]$Status # good, failure
    )

    If ($status -eq "good")    {$Status = "good";$Message = $Message;$Priority = "Low"}
    If ($status -eq "failure") {$Status = "danger";$Message = $Message;$Priority = "High"}

$Payload = @"
    payload={
    "attachments": [
        {
            "color": "$status",
            "text": "$Message",
            "fields": [
                {
                    "title": "Priority",
                    "value": "$Priority",
                }
            ]
        }
    ]
}
"@

    Invoke-RestMethod -Uri $uri -Method Post -Body $Payload
}
```

[Go back](https://mufana.github.io/blog)
