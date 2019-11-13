# Welcome to Jambones API documentation

jambones is an open source CPAAS project based on [drachtio](https://drachtio.org), [rtpengine](https://github.com/sipwise/rtpengine) and [freeswitch](https://freeswitch.com/)

The origin of the name is unclear, but it is rumoured to either be an acronym for:

*just another mediocre boring object notational exercise in silliness*

or a nod to an obscure piece of 1980s-era Boston slang, meaning:

jambones [jam-b&#333;nz]: to move fast, with reckless abandon: *Geraint Thomas was going jambones on that descent!*

## Overview

A jambones application controls calls via web callbacks and an HTTP API.  The jambones platform notifies applications of incoming calls and call status changes via web callbacks containing JSON payloads.  An application can provide call control instructions both by responding to web callbacks, as well as invoking an HTTP REST API.

Each application has an identifier (the application uuid) and an api token that is used for authentication.  When registering an application you provide the information below, and receive in return an application uuid and api token.

| application option | description | required
| ------------- |-------------|-------------|
| incoming call| web callback that is invoked when a new call arrives | yes |
| call status change| web callback that is invoked when a call changes state (e.g. from 'ringing' to 'answered') | no |
| registration attempt | web callback that is invoked when a sip device attempts to register or unregister | no |
| subscription attempt | web callback that is invoked when a sip device sends a SUBSCRIBE request | no |
| sip domain | a sip domain to be used for user registration with this application | no |
| incoming call early media support | whether to playback media over an early media connection (i.e. 183) rather than answering and then playing media | no (default: no)|

## Object Identifiers

There are several types of objects that are created or managed in jambones, and which need unique identifiers.  These are summarized below.

| uuid | description | when used |
| ------------- |-------------|-------------|
| application uuid | identifies a jambones application | when authenticating to REST api or configuring callbacks |
| call uuid | identifies a specific incoming or outbound call | when creating calls using the REST api or managing them using callbacks |
| parent call uuid | identifies the call that created this call | when creating an outbound call from an inbound call | 
| conference uuid | identifies an audio bridge | when creating conferences or placing calls into conference |


