# Welcome to jambonz

jambonz is a self-hosted, "bring your own everything" open source CPaaS platform, developed by the creator of the [drachtio](https://drachtio.org) open source sip server.

Unlike those fancy-pants CPaaS services, jambonz is designed to be:

- **100% open source** - the entire project is public on github.com/jambonz (all of it!)
- **Easy to self-host** - one-click to run on the infrastructure of your choice
- **Privacy-centric** - customer data is never stored by the platform
- **Multi-tenant** - good for providers and larger orgs 

## Features

- All the usual telephony controls - dial, gather DTMF, leave, park, hangup
- Media control:
    - text-to-speech (TTS) and speech-to-text (STT) integrations, using your google and AWS accounts
    - Call controls for audio playback, speech input, transcription
    - Media forwarding via websocket
- SIP - registration, trunk connectivity, and dialing
- Applications - define and manage a set of call flows and behaviours
- JSON based call control 
- REST API for live call control and resource provisioning and management
- Hierarchical data structure that can handle a variety of deployment scenarios
- Registrar, Call control, Session Border Control (SBC), API server, and management infrastructure to handle all the features above
- EC2 AMIs and terraform scripts for launching a jambonz cluster on AWS
- Anything you want to add - it's an open source project!

## Who is jambonz for?

jambonz is made for:

- **Commercial CPaaS users** that want to save costs using their own SIP trunks and speech services rather than paying up-charged rates to a CPaaS provider.
- **Privacy-concerned Businesses** with stringent data privacy requirements who wish to avoid exposing their customers' sensitive data to third parties that they can't effectively audit.
- **Developers** that want greater control and the ability to add their own features to a CPaaS platform they control.
- **Enterprises** with capable IT departments that are already managing most of what is required for a hosted telephony solution (e.g. cloud storage, speech APIs, infrastructure as code, etc) and are starting to wonder why they are paying so much money to a third party for doing the same thing for them
- **Service providers** that want a white-label product that they can offer as a branded solution to their customers.

## Why does jambonz exist?

There are a lot of CPaaS providers on the market today, and they all provide a similar set of easy-to-use APIs that allow enterprises to manage their communication services in new and innovative ways. The concept was novel 10 years ago, but does it really require the high prices and loss of control that commercial solutions provide today?

jambonz differs from other solutions because it is:

a) **open source**.  <p style="margin-left:10px">Oh, and we mean completely open source (none of that "yes, we have open source, but you really need to think about upgrading to our commercial offering if you're going to be serious about this relationship", haha). <br/><br/> 
> All of the jambonz core software and drachtio is available on github under the [MIT License](https://choosealicense.com/licenses/mit/).</p>

b) a **self-hosted** solution: <p style="margin-left:10px">You run it on your own infrastructure.  Use your own SIP trunks.  Your own storage.  Your own cloud speech credentials.  Why pay someone to upcharge you for all of that when it's basically a one-click experience to provision all of those yourself in today's world.  You know how to click, right?<br/></br/>

> Let's put it this way: ask yourself -- what are you *really* getting of value from that fancy-pants CPAAS service you're paying for, when you take away all of the integrations that you can easily do yourself?  

<p style="margin-left:10px">Just a nice API and application processing engine, that's what.  So why not get your own telephony API engine (hint, hint: that's jambonz!) and bring your own everything else to the party?  Just sayin.</p> 

c) a **radical approach to privacy**.<p style="margin-left:10px">None of your customer's personally identifiable information (PII) is stored at rest within the jambonz platform itself.  Ever.</p>

> Recordings or transcriptions that might contain sensitive information such as credit card numbers, HIPAA-related information, or social security numbers are neever stored at rest within the platform itself.

<p style="margin-left:10px">How about SIP credentials for devices or webRTC clients that you want to be allow to register with the platform and make phone calls?  Sure, we allow all of that but we don't store the credentials -- you do.  We never store any SIP credentials that could be hacked or used by others to run up your bill.</p>

d) **white-labelable**.<p style="margin-left:10px">Is that even a word?  Well, in any case, jambonz is service-provider friendly -- it can operate in a multi-tenant configuration for service providers that want to provide a hosted service for customers who are interested in enjoying the privacy and other features of jambonz without running their own hardware.

## How do jambonz applications work?

Well, if you ***are*** using one of those fancy-pants CPAAS services, then you are already familiar with how this works:

A jambonz application controls calls via web callbacks and an HTTP API.  The jambonz platform notifies your application of incoming calls and call status changes via web callbacks.  Your application provides call control instructions by responding to web callbacks with [JSON payloads](/jambonz) that include instructions, or by invoking a [REST API](/rest).

Additionally, jambonz supports sip end-user devices and webRTC clients registering with the platform and making and receiving calls.

Come on people.  We can do this thing!  

Take the next step, and read on to review the [Call Control](/jambonz) and [management APIs](/rest).

## What's the name mean?

The origin of the name jambonz is unclear, but it is rumoured to either be an acronym for:

*just another mediocre boring object notational exercise in silliness*

<p style="margin-left:10px">or</p>

a nod to an obscure 1980s-era Boston slang term:

<div>
jambones [jam-b&#333;nz] <span style="font-style:italic;">(adverb)</span>: to move fast; with reckless and uncontrolled abandon.<p style="margin:5px 0px 20px 10px;font-style:italic">Geraint Thomas was going jambones on that descent!</p>
</div>
