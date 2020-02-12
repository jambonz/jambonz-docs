# Welcome to jambonz API documentation

jambonz is an <strong style="text-decoration:underline">open source</strong> CPAAS project brought to you by the same people that brought you [drachtio](https://drachtio.org), the open source project for creating SIP server applications.  

The origin of the name jambonz is unclear, but it is rumoured to either be an acronym for:

*just another mediocre boring object notational exercise in silliness*

<p style="margin-left:10px">or</p>

a nod to an obscure 1980s-era Boston slang term:

<div>
jambones [jam-b&#333;nz]: to move fast; with reckless and uncontrolled abandon. <p style="margin:5px 0px 20px 10px;font-style:italic">Geraint Thomas was going jambones on that descent!</p>
</div>

## Overview

There are a lot of CPAAS providers on the market today, and they all provide the same thing:
<ul><li/>A set of easy-to-use APIs that allow enterprises to manage their communication services in new and innovative ways.</li></ul>

So then. Why do we need (yet another) CPAAS?

#### Well, since you asked..
jambonz differs from other solutions because it is:

a) **open source**.  <p style="margin-left:10px">Oh, and we mean completely open source (none of that "yes, we have open source, but you really need to think about upgrading to our commercial offering if you're going to be serious about this relationship", haha). <br/><br/> 
> All of the jambonz core software and drachtio is available under the [MIT License](https://choosealicense.com/licenses/mit/).</p>

b) a **self-hosted** solution: <p style="margin-left:10px">You run it on your own infrastructure.  Use your own SIP trunks.  Your own storage.  Your own cloud speech credentials.  Why pay someone to upcharge you for all of that when it's basically a one-click experience to provision all of those yourself in today's world.  You know how to click, right?<br/></br/>

> Let's put it this way: ask yourself -- what are you *really* getting of value from that fancy-pants CPAAS service you're paying for, when you take away all of the integrations that you can easily do yourself?  

<p style="margin-left:10px">Just a nice API and application processing engine, that's what.  So why not get your own telephony API engine (hint, hint: that's jambonz!) and bring your own everything else to the party?  Just sayin.</p> 

c) a **radical approach to privacy**.<p style="margin-left:10px">None of your customer's personally identifiable information (PII) is stored at rest within the jambonz platform itself.  Ever.</p>

> Recordings or transcriptions that might contain sensitive information such as credit card numbers, HIPAA-related information, or social security numbers are neever stored at rest within the platform itself.

<p style="margin-left:10px">How about SIP credentials for devices or webRTC clients that you want to be allow to register with the platform and make phone calls?  Sure, we allow all of that but we don't store the credentials -- you do.  We never store any SIP credentials that could be hacked or used by others to run up your bill.</p>

d) **white-labelable**.<p style="margin-left:10px">Is that even a word?  Well, in any case, jambonz is service-provider friendly -- it can operate in a multi-tenant configuration for service providers that want to provide a hosted service for customers who are interested in enjoying the privacy and other features of jambonz without running their own hardware. 

#### Who should be interested?
Those interested would include:

- customers that want to **save costs** vs a commercial CPAAS by using their own sip trunks rather than paying more expensive per-minute rates to a CPAAS provider.<br/><br/>
- customers that need to **achieve stringent data privacy requirements** and wish to avoid exposing their customers' sensitive data to third parties that they can't effectively audit.<br/><br/>
- customers that want **greater control and the ability to add features** themselves to their CPAAS platform.<br/><br/>
- enterprises with **highly capable IT departments** that are already managing most of what is required for a hosted telephony solution (e.g. cloud storage, speech APIs, infrastructure as code, etc) and are starting to wonder why they are paying so much money to a third party for doing the same thing for them.<br/><br/>
- **service providers that want a white-label product** that they can to offer as a branded solution to their customers.

### What is a jambonz application?

Well, if you ***are*** using one of those fancy-pants CPAAS services, then you are already familiar with how this works:

A jambonz application controls calls via web callbacks and an HTTP API.  The jambonz platform notifies your application of incoming calls and call status changes via web callbacks.  Your application provides call control instructions by responding to web callbacks with [JSON payloads](/jambonz-docs/jambonz) that include instructions, or by invoking a [REST API](/jambonz-docs/rest).

Additionally, jambonz supports sip end-user devices and webRTC clients registering with the platform and making and receiving calls.

Come on people.  We can do this thing!  

Take the next step, and read on to review our APIs.

