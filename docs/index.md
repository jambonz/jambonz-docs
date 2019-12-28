# Welcome to Jambones API documentation

jambones is an open source CPAAS project based on [drachtio](https://drachtio.org), [rtpengine](https://github.com/sipwise/rtpengine) and [freeswitch](https://freeswitch.com/).  The origin of the name is unclear, but is rumoured to either be an acronym for:

*just another mediocre boring object notational exercise in silliness*,

or a nod to an obscure 1980s-era Boston slang term:

jambones [jam-b&#333;nz]: to move fast, with reckless abandon: *Geraint Thomas was going jambones on that descent!*

## Overview

There are a lot of CPAAS providers on the market today, and they all provide a similar service: a set of easy-to-use APIs that customers can use to drive telephony applications. So..

How is jambones different and why do we need (yet another) CPAAS?

##### How does jambones differ from other CPAAS solutions?
jambones differs from other solutions because it is:

a) an **open source** CPAAS: all the jambones core software is available under the permissive [MIT License](https://choosealicense.com/licenses/mit/). 

b) a **self-hosted** solution: i.e., customers can run on their own infrastructure.  

c) a **privacy-first** solution: no sensitive customer information is held within the platform itself.

> Note: it can also be deployed in a multi-tenant configuration for use by service providers.


##### Who should be interested?
Those interested would include:

- customers that want to **save costs** vs a commercial CPAAS by using their own sip trunks rather than paying more expensive per-minute rates to a CPAAS provider.<br/><br/>
- customers that need to **achieve stringent data privacy requirements** and wish to avoid exposing their customers' sensitive data to third parties.<br/><br/>
- customers that want **greater control and the ability to add features** themselves to their CPAAS platform.<br/><br/>
- enterprises with **highly capable IT departments** that are already managing most of what is required for a hosted telephony solution (e.g. cloud storage, speech APIs, infrastructure as code, etc).<br/><br/>
- **service providers that want a white-label product** that they can to offer as a branded solution to their customers.

### What is a jambones application?
A jambones application controls calls via web callbacks and an HTTP API.  The jambones platform notifies applications of incoming calls and call status changes via web callbacks.  An application can provide call control instructions by responding to web callbacks with JSON payloads that include instructions, or by invoking a REST API.

Additionally, jambones supports sip end-user devices and webRTC clients interacting with the platform.  
> Note that in keeping with the data privacy design goal, the platform does not store customers' sip credentials.  Authentication of native sip clients is delegated to customer-side logic [as described here](/jambones-docs/register-hook).

## jambones data architecture

jambones is designed to be deployed either by an end-user customer on their own behalf or by a service provider that hosts a jambones platform and provides service to multiple enterprises.

The data model therefore distinguishes the following high-level data entities:

- *platform owner* - the entity that operates an instance of the jambones platform.
- *service provider* - an organization that provides service to one or more enterprises: a single instance of the platform may support multiple service providers.
- *account* - the credentials and information for a single enterprise that is using the platform for telephony service.
- *application* - a set of defined call control behaviors to apply to calls managed by an account.
- *carrier* - a VoIP carrier that provides call origination/termination services and DIDs (aka DDIs or telephone numbers) to customers.
- *sip gateways* - the signaling gateways for a carrier.
- *calls* - an instance of a phone call that is controlled via a jambones application.
- *registered user* - an authenticated sip client that belongs to an account.

### data element identifiers
Instances of data model entities are publicly identified with a unique value known as a "sid".  The documentation will therefore frequently refer to the following identifiers:

- *Account Sid* - identifies an account.
- *Application Sid* - identifies an application.
- *Call Sid* - identifies a specific call.
- *Parent Call Sid* - identifies the call that another call is bridged to.

This is not an exhaustive list, since all data elements have a similar unique identifier (e.g. a service provider sid uniqely identifies a service provider, a carrier sid identifies a specific voip carrier, etc).
