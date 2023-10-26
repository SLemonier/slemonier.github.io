---
layout: post
title: "Citrix Logon Simulator"
description: "A Free and Open Source Logon Simulator for Citrix to test the launch of any published application or desktop, through a Citrix Gateway or Citrix Storefront"
excerpt: "A Free and Open Source Logon Simulator for Citrix to test the launch of any published application or desktop, through a Citrix Gateway or Citrix Storefront"
date: 2022-05-12
author:  " Steven"
image: "/img/title_logonsimulator.jpg"
thumbnail: "/img/title_logonsimulator.jpg"
published: true 
tags: Citrix Logon Simulator Gateway Storefront Automation
permalink: "citrix-logon-simulator/"
categories: Citrix Logon Simulator Automation Python
---

***Note:** I recommend you to read the following post but if you're only interested by the script: [here it is](https://github.com/SLemonier/Citrix-Logon-Simulator).*

*Edit 6/27/2022: Sometimes, selenium can be a bit annoying. I found version 3.8 of the package resolves the issue. Pyautogui as well has somme issue at the installation on some machines, I use now a method from Pillow to do a screenshot.*

# An introduction

I always wanted to monitor correctly the end-user experience and especially application availability. Like, at any time of the day, can my business-critical resource be opened without any issue? And if there is, at which stage does it fail? Log on to the portal? Application enumeration? At the application launch?

I know there are some logon simulators available on the market. But they usually come with three majors caveats:
- you have to pay for it
- they are bound to a suite of applications (that you have to pay for...)
- I was not able to have them work (and you know, when it does not work the first time, I usually decide to do it on my own)

One day, [I read about a guy automating an MFA form by doing OCR on his Mac](https://tyler.io/lets-reimplement-an-amazing-first-party-feature-in-the-dumbest-way-possible/). Thought it was genius and wondered if Windows could do something like that (I already had the logon simulator in mind but didn't know how to do it). It does, by default! Windows 10/11 can do a screenshot in PowerShell and run some OCR on it.

It was great. Then I tried on a Windows Server (because I use a server for some monitoring scripts). It became a nightmare. OCR is bound to a component only available on desktop OS. I was so pissed about it!

I know, I could have used Windows 10 but I wanted to have it on a Windows Server!

So I let it on a corner of the table for a better day. It came when a former colleague told me:

> Python is way better in automation than PowerShell. Do Python Steven. At least for this project.

And I tried. And finally got it down in Python. Way more reliable and not OS-dependent.

So here it is!

[You can find the script here](https://github.com/SLemonier/Citrix-Logon-Simulator).

# Pre-requisites

To run the script, you must have the following pre-requisites:

- Python 3 ([Official website](https://www.python.org/downloads/))
- Tesseract, for OCR ([Windows build](https://github.com/UB-Mannheim/tesseract/wiki))
- Tesseract installation folder added to Windows PATH environment variable
- Firefox ([Official website](https://www.mozilla.org/en-US/firefox/new/)) 
- Geckodriver, aligned with Firefox version you installed ([Github repository](https://github.com/mozilla/geckodriver/releases))
- you **MUST** enable HTML5 receiver as the script does not rely on Citrix Workspace App

Once all those pre-requisites are installed, you must run the following command in PowerShell to download the modules required to run Citrix Logon Simulator (the machine you running the script on must have internet access):

```Python
pip install selenium==3.8 pypiwin32 requests Pillow pytesseract
```

To write events in Windows Event Log, the Event Source must be created first (value can be modified within the Python script, just edit the variable 'EventSource'). In Powershell, execute the following commands, as an Administrator:
```PowerShell
$eventsource = "Citrix.LogonSimulator" #Align the value with the one you edited in the Python script
$EventLogSourceParams = @{
    LogName = 'Application'
    Source = $eventsource
}
New-EventLog @EventLogSourceParams
```

# Edit the variables

Before running the logon simulator, you must edit some variables to adapt it to your environment:
- EventSource (The event's source of the events that will be written in Windows Event Log)
- username (do I need to explain?)
- password (really?)
- URL (doesn't matter if it's a Citrix Gateway or a Storefront, https or not, just enter the URL you want to log to)
- ResourceToTest (the published resource to launch, it must be added to the favorites for the testing user before running the script)
- Screenshotfile (where and how to name the screenshot file to do OCR to, can stay as is)
- TextToFind (to validate the application has been properly launched, the script takes a screenshot and runs OCR on it to find this text)
- LogFile (path for the log file, so you can share it with me if something does not work as expected)

# Run the script

Running the python script ([available on github](https://github.com/SLemonier/Citrix-Logon-Simulator)) is pretty straightforward, just run the following command in Powershell:

```PowerShell
python CitrixLogonSimulator.py
```

You'll get the following output (at least, if it goes to end!):

![Script output](/assets/img/logonsimulator/Script_output.png "Script output... when the resource launched properly!")

What the script does.

It tries to resolve the URL, no need to go deeper if it's not accessible right!

Then, it launches an instance of Firefox and goes to the URL. It will check if it's a Citrix Gateway or a Storefront and will adapt the logon process according to its findings.

It logs on, looks for the resource to test once application enumeration is finished, and launches it.

It waits for 60 seconds and takes a screenshot.

And finally runs OCR to check if it launched the resources properly.

At each stage, it will generate some events (and logs in the logfile, of course):
- Event ID 12001, information, in case of success, with some details
- Event ID 12002, error, in case of a failure, with some details

![Logon failure generates an error event.](/assets/img/logonsimulator/Logon-Failure.png "Logon failure generates an error event.")

And if it worked properly, it will generate an Event ID 12000!

Thus, if you understood correctly, monitor the machine the script is running on for Event ID 12002 (as an alert trigger) to be proactive with your applications/desktops availability!

Here is a quick demo:

{{< youtube p16roep2Cps >}}

# Limitations  

As the logon simulator is more an MVP (standing for Minimum Viable Product), it comes with some limitations I hope I'll address in a near future.

Here they are (don't worry, they're not that much big of a deal):
- to install python modules (using 'pip install' command), the computer/virtual machine must have access to the internet; for offline modules installation, it's trickier, I'm looking for a reliable way to redistribute those
- using a proxy must be tested, it may depend on the type of proxy you are using; if you have an issue with the proxy, just let me know on [Twitter](https://twitter.com/StevenLemonier) or [open an issue on the Github repo](https://github.com/SLemonier/Citrix-Logon-Simulator), I'll be glad to help
- supported logon process is limited to login/password
- password is not encrypted, use a test account with very limited rights!
- the resource to test (application or desktop) must be added to the favorites for the testing user before running the script as told earlier
- it's not yet able to manage the logon duration, thus, it waits for 60 seconds and takes a screenshot; if your logon duration is longer (Booooh 🤭), increase the value of *'Sleep(60)'*
- the script does basic OCR (I can add more processing on the image to improve the OCR), it's better to look for some text with high contrast (like black strings on white background) and to not have too much text displayed on the page
- events generated in Windows Event Log are pretty basic, and messages can be improved (I'm open to suggestions!)
- and as I write Windows Event Log, the script is running only on Windows, even if python runs on every OS; if you'd need the script for Linux, I can do a specific version 😉

One last limitation, if you're running the script for monitoring purposes, you have to use auto logon on your machine and run the script as a logon script. It does some interactive stuff (like launching Firefox and doing a screenshot) and it cannot run as an unattended script. Sorry 😟

# Upcoming improvements

Here are some improvements I have in mind but feel free to share your suggestions, I'll be more than happy to add them:

- Improve OCR processing
- Improve messages in Windows Event Log
- Do a screenshot at every stage and store them for history
- Look for the ResourceToTest in other tabs than the landing one once application enumeration is finished (like looking in apps or desktops tabs if the resource is not added to the favorites)
- Support MFA, but this will require your help, so, please contact me if you need such a feature!
- Encrypt password (I have a workaround with OS variables to now have it stored in plain text in the script but this solution is not perfect)