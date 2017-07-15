---
title: "How to PowerShell (Part 1)"
date: 2017-06-14
---

# How to PowerShell (Part1)

My first steps into the world of PowerShell we’re a bit awkward. I fired up the blue console and literally stared at it. I had no idea what to do. Most of the times I closed it and quickly made my way to the relative safety of the black Command Prompt. Until exactly one year ago. I decided to stick with the blue and give it a try. The reason; PowerShell is here and it’s here to stay. Nowadays PowerShell is an important (if not the most important) way to get information / automate and get things done. Whether that be; Exchange, SharePoint, Azure, your own Windows box, custom script/tools to automate certain tasks or PowerShell Desired State Configuration.

## What is PowerShell

Well, PowerShell is exactly that. A shell. Just like the Unix Bash or Windows Command Prompt. There is however one big difference compared to Bash and Prompt. Where Bash and Prompt can only generate text output, PowerShell generates objects. And that is important. Why? Because we can do things with objects. PowerShell is like playing with lego. You can build really cool things with objects. I challenge you to try that with text.

Another great thing about PowerShell is that we have access to the complete .NET framework. Unlike the old VBScript.

## Open the Console

The first time you open up the blue console, you will see something similar to this:

![Console](https://codeinblue.files.wordpress.com/2016/03/1.png)

At first this blue thing can be a bit daunting. But! Don’t be scared of it. Because I can assure you, even with no knowledge of PowerShell you’ll still be able to use it. And you may already use PowerShell without even knowing!


### Say what

You can use native ‘Command Promp’ commands. So: Dir, CD or CLS will work. Even if you come from Linux. Because, Linux commands will work. (ls, Clear).

![Alias](https://codeinblue.files.wordpress.com/2016/03/2.png)

Notice that ‘dir’ and ‘ls’ generate the same output. That’s because ‘Dir” and 'ls’ are aliases for a PowerShell cmdlet called: ‘Get-ChildItem’. So if we type ‘Get-Childitem’ we get exactly the same as with ‘dir’ and ‘ls’.


To see which aliases are defined for a particular cmdlet:

![alias2](https://codeinblue.files.wordpress.com/2016/03/4.png?)

### Why an alias

Well, first; Microsoft is a nice company. They wanted to make it easier for you and me to learn PowerShell. No matter if you come from Windows or Linux. And second; aliases require less typing. Which is always a good thing.

However; there’s always a catch. I strongly recommend that you use full cmdlet names instead of aliases. The use of an alias could make your script harder to read for someone with little or no knowledge of PowerShell. So only use the alias in the shell.

#### How to find all aliases

That easy. Simply type:

```PowerShell
Get-Alias
```

![get-alias](https://codeinblue.files.wordpress.com/2016/03/8.png?)

While this is the default list of aliases that comes with PowerShell, you can also add your own aliases. For instance; I’m a Linux guy. And used working with <cat> and <grep>.  Where <cat> is an actual default alias for <Get-Content>, there is no alias like <grep> for <Select-String>.

But that doesn't stop me from creating my own.

![create-alias](https://codeinblue.files.wordpress.com/2016/03/91.png?)

The proof that my newly created alias is actually there:

![Aliasisthere](https://codeinblue.files.wordpress.com/2016/03/101.png)

And that my alias works:

![aliasworks](https://codeinblue.files.wordpress.com/2016/03/11.png)

#### Gone with the wind

Now, there is one thing to remember. Aliases created like I did in the examples above won’t last forever. As soon as you close your PowerShell console, your aliases will be gone. Why and how to resolve this is something we will discuss in part 2 of this series.

#### And why cmdlets

Why cmdlets and not commands? PowerShell has lots of different cmdlets. Over 1300 in Windows 10. The reason they are called cmdlets is for search engines. If you search for ‘command’ on Google, you get millions of hits. Type; ‘cmdlet’ and you only find PowerShell related hits.

#### Find all cmdlets on your Windows machine

Simply type:

```PowerShell
Get-Command
````

Ok, I know! this one is called 'command' and not cmdlet. 

In the next blog we’ll dig a little deeper and start exploring PowerShell.

[Go back](https://mufana.github.io/blog)
