---
layout: post
title: "James Rankin's first autologon automated"
description: "James Rankin's first autologon automated"
excerpt: "James Rankin's first autologon automated"
date: 2020-12-02
author:  " Steven"
image: "/img/title_powershell.jpg"
thumbnail: "/img/title_powershell.jpg"
published: true 
tags: Windows PowerShell Autologon Automated
permalink: "james-rankins-first-autologon-automated/"
categories: Windows PowerShell    
---

Last month, well-known Citrix Expert James Rankin released the [part 3 of his new series about Windows logon time optimizations](https://james-rankin.com/articles/the-ultimate-guide-to-windows-logon-optimizations-part-3/). We are all aware of logon duration and how much it can be a pain to deal with in our environments, especially since Microsoft added more and more features to its operating system.

New versions continue to increase user logon duration and can raise up to one minute before letting users do their work. One minute does not sound like a big deal. But depending on the context, it can be a hell of a difference (think about hospitals). Thus, any improvement we can make on this part is helpful and must be implemented.

Even if our current logon times are acceptable, it is our duty, as experts, to deliver the best experiences and performances we can offer. Companies don't pay us for clicking "next, next, next" on an installer. They pay us for all the knowledge we have and can leverage on.

So, when James Rankin shared about how to automatize the first logon after pointing out later ones are always faster in his article, I wondered if I can go a bit further in the automation of this. He does it through a couple of GPO, I personally am a "put-everything-you-can-in-your-Golden-Image" guy. I don't have any argument here to justify one approach over the other. I was trained this way, to keep the GPO as minimal as possible. But share your thoughts on this one if you feel your way is better.

In the upcoming months, I am projecting to automatize the creation of my Golden Image on a regular basis, thanks to Packer and Ansible, and this first automated logon should be part of it.

And there is always something new to learn.

Take the first part of James Rankin's implementation : a shutdown script.

How would you proceed without GPO ? First, I tried with a scheduled task. There is no trigger for the shutdown of a computer. Then, I thought about capturing the event 6006. It did not work. I ended up on an article about implementing the equivalent of local gpedit.msc shutdown script through the registry. It is not the cleanest way to do so, in my opinion, but it does the job. If you have any idea how to improve this part, please share it, I'll be glad to add your contribution.

The remaining parts were straight forward actually: set an autologon script for the autologon user to autologoff (high-five if you understood this at first read ✋).

Of course, I added some verification, like checking [Sysinternal tool Autologon](https://docs.microsoft.com/en-us/sysinternals/downloads/autologon) is available on the system.

You will probably wonder by seeing the script why I don't Convert-ToSecureString the password parameter or require to have it secured prior to the script's execution. Well, I am not entirely sure how it would handle this part with Packer or Ansible. So, I preferred to keep it simple but have in mind the autologon account will have its password stored in clear text in SetAutologon.bat.

Maybe I could set some ACLs on this file in a V2. 🤔

The script is pretty easy to call:

```PowerShell
.\SetAutologon.ps1 -Account leogetz -Password okay-okay-okay!
```

SetAutologon.ps1 -Account leogetz -Password okay-okay-okay!
As usual, I added some parameters to set the path where to store the scripts (default is C:\Script) and the output file for the log (default is the current directory).

Okay ! You'll find the [latest version on Github](https://github.com/SLemonier/CitrixAutomation/blob/main/SetAutologon.ps1), feel free to share your comments, your improvement's ideas or custom it your way!

```PowerShell
<#
 .Synopsis
 Configure local machine for autologon as described in James Rankin's article (https://james-rankin.com/articles/the-ultimate-guide-to-windows-logon-optimizations-part-3/).
 .Description
 Configure local machine autologon with given account (no need to specify the domain) and password.
 Configure logon script for the given account to logoff automatically.
 .Parameter Account
 Specifiy the account to use for autologon.
 This parameter is mandatory, ensure the account already exists in the domain.
 .Parameter Password
 Specifiy the account's password to use for autologon.
 This parameter is mandatory, ensure the password is correct.
 .Parameter Path
 Specify the folder where to store batches created by the script.
 This parameter is optionnal, by default, it will store in C:\Scripts.
 .Parameter Log
 Specifiy the output file for the logs.
 This parameter is optionnal, by default, it will create a file in the current directory.
 .Example
 # Configure local machine to autolog the user leogetz with the password P@ssw0rd
 SetAutologon.ps1 -Account leogetz -Password P@ssw0rd
 .Example
 # Configure local machine to autolog the user leogetz with the password P@ssw0rd and store the scripts in C:\Tmp
 SetAutologon.ps1 -Account leogetz -Password P@ssw0rd -Path C:\Tmp
 .Example
 # Configure local machine to autolog the user leogetz with the password P@ssw0rd and store the scripts in C:\Tmp
  and log the output in C:\Temp
 SetAutologon.ps1 -Account leogetz -Password P@ssw0rd -Path C:\Tmp -Log "C:\temp\test.log"
#>

[CmdletBinding()]
Param(
    # Declaring input variables for the script
    [Parameter(Mandatory=$true)] [string]$Account,
    [Parameter(Mandatory=$true)] [string]$Password,
    [Parameter(Mandatory=$false)] [string]$Path="C:\Scripts",
    [Parameter(Mandatory=$false)][ValidateNotNullOrEmpty()] [string]$LogFile=".\SetAutologon.log"
)

#Start logging
Start-Transcript -Path $LogFile

#Setting variables prior to their usage is not mandatory
Set-StrictMode -Version 2

################################################################################################
#Checking the parameters
################################################################################################

Write-Host "Checking Path..." -NoNewline
if(Test-Path -Path $Path){
    Write-Host "OK" -ForegroundColor Green
} else {
    New-Item -ItemType Directory -Path $Path | Out-Null
    Write-Host "OK" -ForegroundColor Green
}

################################################################################################
#Checking the pre-requisites
################################################################################################

Write-Host "Checking Sysinternal tool Autologon... " -NoNewline
try {
    autologon64 $Account $env:USERDOMAIN $Password /accepteula
    Write-Host "OK" -ForegroundColor Green
}
catch {
    Write-Host "Failed" -ForegroundColor Red
    Write-Host "Sysinternal tool Autologon is not available. Please download from https://docs.microsoft.com/en-us/sysinternals/downloads/autologon and unzip Autologon64.exe in default system Path (for instance ""C:\windows\system32\"")" -ForegroundColor Red
    Stop-Transcript 
    break
}

################################################################################################
#Creating Autologon
################################################################################################

Write-Host "Creating Autologon batch... " -NoNewline
if(!(Test-Path -Path C:\Windows\System32\GroupPolicy\Machine\SetAutologon.bat)){
    try {
        Write-Host "Adding autologon64 $Account $env:USERDOMAIN $Password /accepteula to C:\Windows\System32\GroupPolicy\Machine\SetAutologon.bat... " -NoNewline -ForegroundColor Yellow
        if ( -Not (Test-Path "C:\Windows\System32\GroupPolicy\Machine")){
            New-Item -Path "C:\Windows\System32\GroupPolicy\Machine" -ItemType Directory | out-null
        }  
        New-Item -Path C:\Windows\System32\GroupPolicy\Machine\ -Name SetAutologon.bat -ItemType File | Out-Null
        Set-Content -Path C:\Windows\System32\GroupPolicy\Machine\SetAutologon.bat  "autologon64 $Account $env:USERDOMAIN $Password /accepteula" -Encoding ASCII
        Write-Host "OK" -ForegroundColor Green
    }
    catch {
        Write-Host "Failed" -ForegroundColor Red
        Stop-Transcript 
        break
    }
} else {
    try {
        Remove-Item -Path C:\Windows\System32\GroupPolicy\Machine\SetAutologon.bat -Force | Out-Null
        if ( -Not (Test-Path "C:\Windows\System32\GroupPolicy\Machine")){
            New-Item -Path "C:\Windows\System32\GroupPolicy\Machine" -ItemType Directory | out-null
        }  
        New-Item -Path C:\Windows\System32\GroupPolicy\Machine\ -Name SetAutologon.bat -ItemType File | Out-Null
        Write-Host "Adding autologon64 $Account $env:USERDOMAIN $Password /accepteula to C:\Windows\System32\GroupPolicy\Machine\SetAutologon.bat... " -NoNewline -ForegroundColor Yellow
        Set-Content -Path C:\Windows\System32\GroupPolicy\Machine\SetAutologon.bat  "autologon64 $Account $env:USERDOMAIN $Password /accepteula" -Encoding ASCII
        Write-Host "OK" -ForegroundColor Green
    }
    catch {
        Write-Host "Failed" -ForegroundColor Red
        Stop-Transcript 
        break
    }
}

Write-Host "Creating Autologon Scheduled task... " -NoNewline
if(!(Get-ScheduledTask | Where-Object{$_.TaskName -match "SetAutologon"})){
    try {
        New-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\" -Name Scripts -Force | Out-Null
        New-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\Scripts\" -Name Shutdown -Force | Out-Null
        New-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\Scripts\Shutdown\" -Name 0 -Force | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\Scripts\Shutdown\0" -Name "GPO-ID" -Value "LocalGPO" | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\Scripts\Shutdown\0" -Name "SOM-ID" -Value "Local" | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\Scripts\Shutdown\0" -Name "FileSysPath" -Value "C:\Windows\System32\GroupPolicy\Machine" | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\Scripts\Shutdown\0" -Name "DisplayName" -Value "Local Group Policy" | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\Scripts\Shutdown\0" -Name "GPOName"  -Value "Local Group Policy" | Out-Null
        New-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\Scripts\Shutdown\0\" -Name 0 -Force | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\Scripts\Shutdown\0\0" -Name "Script" -Value "SetAutologon.bat" | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\Scripts\Shutdown\0\0" -Name "Parameters" -Value "" | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\Scripts\Shutdown\0\0" -Name "ExecTime" -Value ([byte[]](0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00)) | Out-Null
        New-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\" -Name Scripts -Force | Out-Null
        New-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\" -Name Shutdown -Force | Out-Null
        New-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Shutdown\" -Name 0 -Force | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Shutdown\0" -Name "GPO-ID" -Value "LocalGPO" | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Shutdown\0" -Name "SOM-ID" -Value "Local" | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Shutdown\0" -Name "FileSysPath" -Value "C:\Windows\System32\GroupPolicy\Machine" | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Shutdown\0" -Name "DisplayName" -Value "Local Group Policy" | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Shutdown\0" -Name "GPOName"  -Value "Local Group Policy" | Out-Null
        New-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Shutdown\0\" -Name 0 -Force | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Shutdown\0\0" -Name "Script" -Value "SetAutologon.bat" | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Shutdown\0\0" -Name "Parameters" -Value "" | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Shutdown\0\0" -Name "IsPowershell" -Value 0 -Type dword | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Shutdown\0\0" -Name "ExecTime" -Value ([byte[]](0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00)) | Out-Null
        if ( -Not (Test-Path "C:\Windows\System32\GroupPolicy\Machine\Scripts\Startup")){
            New-Item -Path "C:\Windows\System32\GroupPolicy\Machine\Scripts\Startup" -ItemType Directory | out-null
        }   
        if ( -Not (Test-Path "C:\Windows\System32\GroupPolicy\User\Scripts\Startup")){
            New-Item -Path "C:\Windows\System32\GroupPolicy\User\Scripts\Startup" -ItemType Directory | out-null
        }   
        if ( -Not (Test-Path "C:\Windows\System32\GroupPolicy\Machine\Scripts\Shutdown")){
            New-Item -Path "C:\Windows\System32\GroupPolicy\Machine\Scripts\Shutdown" -ItemType Directory | out-null
        }   
        if ( -Not (Test-Path "C:\Windows\System32\GroupPolicy\User\Scripts\Shutdown")){
            New-Item -Path "C:\Windows\System32\GroupPolicy\User\Scripts\Shutdown" -ItemType Directory | out-null
        }
        Write-Host "OK" -ForegroundColor Green
    }
    catch {
        Write-Host "Failed" -ForegroundColor Red
        Stop-Transcript 
        break
    }
}else {
    Write-Host "Task already exists" -ForegroundColor Yellow
    Write-Host "Replacing existing task... " -NoNewline
    try {
        New-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\" -Name Scripts -Force | Out-Null
        New-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\Scripts\" -Name Shutdown -Force | Out-Null
        New-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\Scripts\Shutdown\" -Name 0 -Force | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\Scripts\Shutdown\0" -Name "GPO-ID" -Value "LocalGPO" | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\Scripts\Shutdown\0" -Name "SOM-ID" -Value "Local" | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\Scripts\Shutdown\0" -Name "FileSysPath" -Value "C:\\Windows\\System32\\GroupPolicy\\Machine" | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\Scripts\Shutdown\0" -Name "DisplayName" -Value "Local Group Policy" | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\Scripts\Shutdown\0" -Name "GPOName"  -Value "Local Group Policy" | Out-Null
        New-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\Scripts\Shutdown\0\" -Name 0 -Force | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\Scripts\Shutdown\0\0" -Name "Script" -Value "SetAutologon.bat" | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\Scripts\Shutdown\0\0" -Name "Parameters" -Value "" | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\Scripts\Shutdown\0\0" -Name "ExecTime" -Value ([byte[]](0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00)) | Out-Null
        New-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\" -Name Scripts -Force | Out-Null
        New-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\" -Name Shutdown -Force | Out-Null
        New-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Shutdown\" -Name 0 -Force | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Shutdown\0" -Name "GPO-ID" -Value "LocalGPO" | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Shutdown\0" -Name "SOM-ID" -Value "Local" | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Shutdown\0" -Name "FileSysPath" -Value "C:\\Windows\\System32\\GroupPolicy\\Machine" | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Shutdown\0" -Name "DisplayName" -Value "Local Group Policy" | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Shutdown\0" -Name "GPOName"  -Value "Local Group Policy" | Out-Null
        New-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Shutdown\0\" -Name 0 -Force | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Shutdown\0\0" -Name "Script" -Value "SetAutologon.bat" | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Shutdown\0\0" -Name "Parameters" -Value "" | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Shutdown\0\0" -Name "IsPowershell" -Value 0 -Type dword | Out-Null
        Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Shutdown\0\0" -Name "ExecTime" -Value ([byte[]](0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00)) | Out-Null
        if ( -Not (Test-Path "C:\Windows\System32\GroupPolicy\Machine\Scripts\Startup")){
            New-Item -Path "C:\Windows\System32\GroupPolicy\Machine\Scripts\Startup" -ItemType Directory | out-null
        }   
        if ( -Not (Test-Path "C:\Windows\System32\GroupPolicy\User\Scripts\Startup")){
            New-Item -Path "C:\Windows\System32\GroupPolicy\User\Scripts\Startup" -ItemType Directory | out-null
        }   
        if ( -Not (Test-Path "C:\Windows\System32\GroupPolicy\Machine\Scripts\Shutdown")){
            New-Item -Path "C:\Windows\System32\GroupPolicy\Machine\Scripts\Shutdown" -ItemType Directory | out-null
        }   
        if ( -Not (Test-Path "C:\Windows\System32\GroupPolicy\User\Scripts\Shutdown")){
            New-Item -Path "C:\Windows\System32\GroupPolicy\User\Scripts\Shutdown" -ItemType Directory | out-null
        }
        Write-Host "OK" -ForegroundColor Green
    }
    catch {
        Write-Host "Failed" -ForegroundColor Red
        Stop-Transcript 
        break
    }
}

################################################################################################
#Allowing Autologon user to amend registry key HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon
################################################################################################

Write-Host "Allowing Autologon user to amend Autologon registry key... " -NoNewline
try {
    $acl = Get-Acl "HKLM:\Software\Microsoft\Windows NT\CurrentVersion\Winlogon"
    $rule = New-Object System.Security.AccessControl.RegistryAccessRule ("$env:USERDOMAIN\$account","FullControl","ContainerInherit,ObjectInherit","None","Allow")
    $acl.SetAccessRule($rule)
    $acl | Set-Acl -Path "HKLM:\Software\Microsoft\Windows NT\CurrentVersion\Winlogon"
    Write-Host "OK" -ForegroundColor Green
}
catch {
    Write-Host "Failed" -ForegroundColor Red
    Stop-Transcript 
    break
}

################################################################################################
#Creating Autologoff
################################################################################################

Write-Host "Creating Autologoff batch... "
if(!(Test-Path -Path $Path\SetAutologoff.bat)){
    try {
        New-Item -Path $Path -Name SetAutologoff.bat -ItemType File | Out-Null
        Set-Content -Path $Path\SetAutologoff.bat  "reg delete ""HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon"" /v DefaultUserName /f" -Encoding ASCII
        Write-Host "Adding reg delete ""HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon"" /v DefaultUserName /f to $Path\SetAutologoff.bat... " -ForegroundColor Yellow
        Set-Content -Path $Path\SetAutologoff.bat  "reg add ""HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon"" /v AutoAdminLogon /t REG_SZ /d 0 /f" -Encoding ASCII
        Write-Host "Adding reg add ""HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon"" /v AutoAdminLogon /t REG_SZ /d 0 /f to $Path\SetAutologoff.bat... " -ForegroundColor Yellow
        Set-Content -Path $Path\SetAutologoff.bat  "logoff" -Encoding ASCII
        Write-Host "Adding logoff to $Path\SetAutologoff.bat... " -ForegroundColor Yellow
        Write-Host "Creating Autologoff batch... " -NoNewline
        Write-Host "OK" -ForegroundColor Green
    }
    catch {
        Write-Host "Failed" -ForegroundColor Red
        Stop-Transcript 
        break
    }
} else {
    try {
        Remove-Item -Path $Path\SetAutologoff.bat -Force | Out-Null
        New-Item -Path $Path -Name SetAutologoff.bat -ItemType File | Out-Null
        Set-Content -Path $Path\SetAutologoff.bat  "reg delete ""HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon"" /v DefaultUserName /f" -Encoding ASCII
        Write-Host "Adding reg delete ""HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon"" /v DefaultUserName /f to $Path\SetAutologoff.bat... " -ForegroundColor Yellow
        Set-Content -Path $Path\SetAutologoff.bat  "reg add ""HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon"" /v AutoAdminLogon /t REG_SZ /d 0 /f" -Encoding ASCII
        Write-Host "Adding reg add ""HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon"" /v AutoAdminLogon /t REG_SZ /d 0 /f to $Path\SetAutologoff.bat... " -ForegroundColor Yellow
        Set-Content -Path $Path\SetAutologoff.bat  "logoff" -Encoding ASCII
        Write-Host "Adding logoff to $Path\SetAutologoff.bat... " -ForegroundColor Yellow
        Write-Host "Creating Autologoff batch... " -NoNewline
        Write-Host "OK" -ForegroundColor Green
    }
    catch {
        Write-Host "Failed" -ForegroundColor Red
        Stop-Transcript 
        break
    }
}

Write-Host "Creating Autologoff Scheduled task... " -NoNewline
if(!(Get-ScheduledTask | Where-Object{$_.TaskName -match "SetAutologoff"})){
    try {
        $taskAction = New-ScheduledTaskAction -Execute $Path\SetAutologoff.bat
        $taskTrigger = New-ScheduledTaskTrigger -AtLogOn -User "$env:USERDOMAIN\$account"
        Register-ScheduledTask -TaskName "SetAutologoff" -Description "Configure Autologff for $Account" -Action $taskAction -Trigger $taskTrigger -user "$env:USERDOMAIN\$Account" | Out-Null
        Write-Host "OK" -ForegroundColor Green
    }
    catch {
        Write-Host "Failed" -ForegroundColor Red
        Stop-Transcript 
        break
    }
}else {
    Write-Host "Task already exists" -ForegroundColor Yellow
    Write-Host "Replacing existing task... " -NoNewline
    try {
        Unregister-ScheduledTask -TaskName "SetAutologoff" -Confirm:$false | Out-Null
        $taskAction = New-ScheduledTaskAction -Execute $Path\SetAutologoff.bat
        $taskTrigger = New-ScheduledTaskTrigger -AtLogOn -User "$env:USERDOMAIN\$account"
        Register-ScheduledTask -TaskName "SetAutologoff" -Description "Configure Autologff for $Account" -Action $taskAction -Trigger $taskTrigger -user "$env:USERDOMAIN\$Account" | Out-Null
        Write-Host "OK" -ForegroundColor Green
    }
    catch {
        Write-Host "Failed" -ForegroundColor Red
        Stop-Transcript 
        break
    }
}

#Stop logging
Stop-Transcript
```