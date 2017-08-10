---
title: "Unblock files with PowerShell"
date: 2017-08-10excerpt: "Unblock files with a PowerShell oneliner!"
tags:
- PowerShell
categories:
- powershell
comments: true
author_profile: true
---

# Unblock files with PowerShell

A simple oneliner to unblock files with PowerShell.

```powershell
Get-ChildItem -recurse "PathToFilesOrFolder" | Unblock-File -verbose
````
