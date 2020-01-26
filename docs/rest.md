
# Overview
The jambonz REST API allows applications to query, create, and manage calls and other resources. 

**Base URL**

All calls should use the following base URL:
```
https://<domain>/api/jambonz/v1
```
where domain is set according to your installation.

**Authentication**

The REST api requires that you implement basic authentication by including an HTTP Authorization header in all requests, consisting of the application uuid concatenated to a valid api token for that application, separated by a colon (:).

**Dates and Times**

All dates and times are UTC, using RFC 2822 format.

**Phone Numbers**

All phone numbers are in E.164 format, starting with a plus sign ("+") and the country code.

## Applications
An application represents a set of unified behaviors to be applied to phone calls either made or received through the platform.  Applications can be created, queried, updated, and destroyed via the API.

## API Key
An api key is a token that is associated with an application and is used to authenticate requests on behalf of that application.  Api keys can be created and destroyed via the API.  When created, they are associated to one and only one application.

## Calls
A call is a voice connection made between the jambonz platform and another endpoint, which may be a phone or a sip endpoint. Inbound calls are those made from external numbers or devices towards the platform, while outbound calls are placed by the platform to an endpoint.  Inbound calls quite often are used to trigger outbound calls and in such a situation the outbound call will have a parent call uuid attribute that references the inbound call.

Calls may created, modified, and deleted through the API.

## Conference participants
Conference participants refer to calls that are actively connected to a conference. You can mute or remove participants from a conference as well as retrieve a list of all participants, along with detailed information about each participant, in an active conference.

## Conferences
Conferences represent a common endpoint that can mix the audio from multiple calls.  Conferences can be created, modified and deleted through the API.

## Phone numbers
Phone numbers represent phone numbers that route to the jambonz platform, and may be associated with an application.  A Phone number may be associated with zero or one Application.  Phone numbers can be created and destroyed through the API, as well as being modified to point to a different application.

## Queues
Queues represent an ordered collection of active calls that are parked (not connected to a far end).  Queues may be created and deleted through the API.

## Queued calls
Queued calls are calls that have been assigned to a queue.

