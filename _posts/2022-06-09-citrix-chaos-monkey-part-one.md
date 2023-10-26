---
layout: post
title: "Citrix Chaos Monkey - Part One"
description: "First part of a serie about Chaos Engineering for experimenting your Citrix environment resiliency"
excerpt: "First part of a serie about Chaos Engineering for experimenting your Citrix environment resiliency"
date: 2022-06-29
author:  " Steven"
image: "/img/title_citrixchaosmonkey.jpg"
thumbnail: "/img/title_citrixchaosmonkey.jpg"
published: true 
tags: Citrix Chaos Monkey Chaos Engineering
permalink: "citrix-chaos-monkey-part-one"
categories: Citrix Chaos Monkey Chaos Engineering
---

If you’re not familiar with Chaos Engineering, let’s start with the definition:

Chaos engineering is the discipline of experimenting on a distributed system to build confidence in the system's capability to withstand turbulent and unexpected conditions in production, [according to Wikipedia](https://en.wikipedia.org/wiki/Chaos_engineering).

It might sound pretty similar to "Disaster Recovery Plan" but it’s actually far from it.
In such a situation, companies tend to test how they can restore quickly the service in case of a massive failure, like the loss of a data center or a building. 
They validate procedures and so on in a very controlled environment. They plan the simulation, agree with every team involved on when to start and they start the plan in another environment, split from the production. Without any "real-life pressure".

Even if I heard of a company than runs its production one week a month in its DRP-restored environment, this is uncommon.

And again, controlled, from the start to the finish line. And everyone is aware. Even partners like networking companies.

[Chaos Engineering](https://principlesofchaos.org/), on the other hand, is different.

Your production environment runs, and from time to time, an automated tool generates some issues. On his own. Without warning anyone about the failure it is about to generate or had generated.

It starts from shutting down a service or a virtual machine to even generating latency on the network or databases for the most advanced ones.

Only the Chaos Engineering team knows what their tool is capable of, but they don’t know when it will do something or what.

This is the ultimate confidence in your infrastructure and your response teams!

Chaos Engineering started with Netflix, releasing in 2013, after they migrated to the Cloud, their [Simian Army](https://github.com/Netflix/SimianArmy), evolving later on to the [Chaos Monkey](https://github.com/netflix/chaosmonkey). With this migration, they found their infrastructure was becoming more and more complex (something cloud providers are not telling you). And they were afraid of not being capable of identifying issues and resolving them without impacting millions of users watching their TV shows on Netflix.

They imagine the tool like “What if we let a crazy monkey do shitty things in our data center, like pulling cable, powering off machines, and so on, would we be able to survive?”
The tool then evolved to the Simian Army, with each species specified in a sort of failure they generate.

Guess what the Chaos Gorilla is supposed to do. 😁

Of course, the goal of all of this is to be sure the systems, and the services you build are resilient. But at any time. Without impacting the end-user experience.

But beyond that, why should you do Chaos Engineering?
- First, of course, is to test your monitoring.
    - Are proper alerts sent at the correct time? (I have in mind an SSL certificate about to expire, luckily, my colleague remembered it was at that time of the year he used to renew it, he looked at it and found the certificate was about to expire 4 days later. The monitoring server supposed to check that was “off”)
    - What is the point of breaking stuff if you’re not able to detect the failures? With Chaos Engineering, you ensure you properly monitor your infrastructure.
- Then, measure your response
    - Are people trained enough to know how to handle generated alerts?
    - Do they know how to respond?
        - Is the Documentation available?
        - Do they know where it is? It is easy to put your hands on?
        - Do they provide sufficient troubleshooting?
        - Do they know who to escalate to? Which information to provide to the escalation team?
    - Is it possible to do some auto-remediation?
- How many time do they need to fix a situation?
    - Are they able to find a relevant workaround to mitigate the issue? Can it be automated?
    - The more they are trained to respond, the more confident they are when they face a failure, and the more confident other people will be in your service’s quality/resilience to resolve an issue
    - The more you know what can happen to your infrastructure and how to respond, the less you will be subject to the stress of uncertainty
- And finally, you can continuously improve your processes
    - It allows you to improve monitors, threshold, documentation, and put in place auto-remediation when possible and test all of this!
    - You improve your Chaos Monkey and monitoring each time a new “root cause” is identified and reproducible
    - You identify dependencies and impacts, and put in place monitors/alerts to be “aware” of failure outside of your scope but impacting your service (in the rush, other teams don’t think to warn other impacted teams when they are entirely focused on the resolution of their faulty service)
    - You can imagine new ways to monitor your end-user experience (Check my Citrix Logon Simulator for instance ;))

Basically, it means creating trouble! And work to mitigate them.

You can start small, one step at a time:
- Generate CPU and Memory peaks
- Stop services
- Kill TCP connections
- Alter data

And later on, you’ll add latency and more complicated stuff. But only when you know your basics are trustworthy.

Of course, you should start by training your Chaos Monkey in a DEV environment first. So, if you break things, at least your end users won’t complain about your first tries (and your first un-properly monitored services going down 😁).

Let’s be honest, it does sound fun to break things on purpose, doesn’t it? 😉
Some companies event organize [a contest for their teams around chaos engineering](http://days-of-chaos.com/), with a commented live stream of the event.

In the next series of articles, I’ll share my Citrix Chaos Monkey and how to use them to train yourself or break things randomly in your Citrix infrastructure. We’ll start small, with the Licensing server and then, we’ll move to other components.

Stay tuned guys!