
# Overview
The jambonz REST API allows applications to query, create, and manage calls and other resources. 

**Base URL**

All calls should use the following base URL:
```
https://{serviceUrl}/v1
```
where serviceUrl is set according to your own installation.

**Authentication**

The REST api uses HTTP Bearer Authentication which requires that you include an HTTP Authorization header containing a valid api token.

**Dates and Times**

All dates and times are UTC, using RFC 2822 format.

**Phone Numbers**

All phone numbers are in E.164 format, starting with a plus sign ("+") and the country code.

## Applications
An application represents a set of unified behaviors to be applied to phone calls either made or received through the platform.  Applications can be created, queried, updated, and destroyed via the API by using an api_key with Account level scope.

Applications have the following properties:

| property        | description |
| ------------- |------------- |
| account_sid | identifies the Account this application belongs to |
| application_sid | unique identifier for the application | 
| name | user-specified name of application |
| call_hook | web callback to invoke for new calls |
| call_status_hook | web callback to invoke for call status notifications |
| speech_recognizer_vendor | vendor to use for speech recognition.  Default: google|
| speech_recognizer_language | default language to use for speech recognition, if not specified in the 'gather' or 'transcribe' verbs.  Default: en-US |
| speech_synthesis_vendor | vendor to use for text to speech.  Default: google|
| speech_synthesis_voice | default voice to use for text to speech, if not specified in the 'say' verb.  Default: en-US-Wavenet-C |

### Retrieving an Application

**GET /v1/Applications/{ApplicationSid}**

```xml
curl -X GET "http://{serviceUrl}/v1/Applications/0e0681b0-d49f-4fb8-b973-b5a3c6758de1" \
  -H "accept: application/json" \
  -H "Authorization: Bearer 38700987-c7a4-4685-a5bb-af378f9734de"

200 response
{
	"application_sid": "0e0681b0-d49f-4fb8-b973-b5a3c6758de1",
	"name": "daveh test app",
	"account_sid": "fef61e75-cec3-496c-a7bc-8368e4d02a04",
	"speech_synthesis_vendor": "google",
	"speech_synthesis_voice": "en-US-Wavenet-C",
	"speech_recognizer_vendor": "google",
	"speech_recognizer_language": "en-US",
	"call_hook": {
		"webhook_sid": "f227508f-dbb0-4fe8-a7e8-2974f54e24a1",
		"url": "http://vibe-test.drachtio.org.ngrok.io",
		"method": "post",
		"username": "drachtio",
		"password": "3298fn238hf9n823hf93h24398gj93n84h"
	},
	"call_status_hook": {
		"webhook_sid": "8226d33c-6a46-40d4-8fb3-02ef01199085",
		"url": "http://vibe-test.drachtio.org.ngrok.io/callStatus",
		"method": "post",
		"username": "drachtio",
		"password": "3298fn238hf9n823hf93h24398gj93n84h"
	}
}
```

### Listing Applications

**GET /v1/Applications**

Lists all Applications an Account (if an account scope api token is used).

### Deleting an Application

**DELETE /v1/Applications/{ApplicationSid}**

Deletes a specific Applications.

### Creating an Application

**POST /v1/Applications**

Creates a new Application.

Upon success, a 201 response is returned with a JSON body providing the account_sid ("sid") of the application that was created.

```xml
curl -X POST "http://{serviceUrl}/v1/Applications" \
-H "accept: application/json" \
-H "Authorization: Bearer 38700987-c7a4-4685-a5bb-af378f9734de" \
-H "Content-Type: application/json" \
-d "{\"name\":\"forward to mobile\",\"account_sid\":\"fef61e75-cec3-496c-a7bc-8368e4d02a04\",\"call_hook\":{\"url\":\"http://vibe-test.drachtio.org.ngrok.io/mobile\"},\"call_status_hook\":{\"url\":\"http://vibe-test.drachtio.org.ngrok.io/mobileStatus\"}}"

201 response
{
  "sid": "65cddc02-6cf8-476c-97a8-1941a96e143b"
}
```

## Calls
A call is a voice connection made between the jambonz platform and another endpoint, which may be a phone or a sip endpoint. Inbound calls are those made from external numbers or devices towards the platform, while outbound calls are placed by the platform to an endpoint.  Inbound calls quite often are used to trigger outbound calls and in such a situation the outbound call will have a Parent Call Sid that references the inbound call.

Calls may created, modified, and deleted through the API.

### Create a Call
Calls are created from the REST API by sending an HTTP POST request. A successful HTTP 201 response will contain the Call Sid of the call attempt that has been launched.

An example is shown below:
```xml
POST /v1/Accounts/fef61e75-cec3-496c-a7bc-8368e4d02a04/Calls HTTP/1.1
Content-Length: 175
Accept: application/json
Authorization: Bearer 9404e5f7-9a77-4bcc-b0fa-5665ace28ab3
Content-Type: application/json

{
  "application_sid": "0e06a1b0-d49f-4fb8-b973-b5a3c6758de1",
  "from": "+15083728299",
  "to": {
    "type": "phone",
    "name": "+16172375089"
  }
}

HTTP/1.1 201 Created
Content-Type: application/json; charset=utf-8
Content-Length: 46

{
  "sid":"9210add6-9573-4860-a003-648c7829faaa"
}
```

The Request-URI of the POST contains the Account Sid of the caller and JSON payload  may contain the following properties

| property      | description | required  |
| ------------- |-------------| -----|
| application_sid | The unique identifier of the application used to handle the call | either call_hook or application_sid must be supplied |
| call_hook | an object specifying a web callback that will be invoked when the call is answered | either call_hook or application_sid must be supplied |
| call_hook.url | web callback url | yes |
| call_hook.method | 'GET' or 'POST'.  Defaults to 'POST' | no |
| call_hook.username | username for HTTP Basic Authentication | no |
| call_hook.password | password for HTTP Basic Authentication | no |
| call_status_hook | an object specifying a  a web callback that will be invoked with call status notifications.  Object properties the same as 'call_hook' property above. | no |
| from | the calling party number | yes |
| timeout | the number of seconds to wait for the call to be answered.  Defaults to 60. | no |
| to | specifies the destination of the call. See description of [target types](/jambonz-docs/jambonz#target-types) in jambonz call control language. | yes | 

At the time that the 201 response is returned to the caller, the call attempt has been launched (i.e., the SIP INVITE has been sent) but no ringing or call answer has yet occurred.  The caller will receive call status notifications via the call_status_hook (either that supplied in the POST request, or if an application_sid is supplied then via the configured call_status_hook for that application).

## Live Call Control

TBD


## Management API

### Accounts
Accounts allow a subscriber to set some useful defaults that apply to all their Applications.

Accounts have the following properties:

| property        | description |
| ------------- |------------- |
| account_sid | unique identifier for the account | 
| device_calling_hook | web callback invoked when a registered sip user places a call |
| is_active | indicates whether account is active or not |
| name | account name |
| registration_hook | web callback to use to authenticate sip users/devices |
| sip_realm | the sip realm (domain) that groups all sip users within the Account |

#### Retrieving an Account

**GET /v1/Accounts/{AccountSid}**

```xml
curl -X GET "https://{serviceUrl}/v1/Accounts/fef61e75-cec3-496c-a7bc-8368e4d02a04" \
   -H "accept: application/json" \
   -H "Authorization: Bearer 6604e5f7-9a77-42cc-b0fa-5665ace28ab3"

200 response
{
  "account_sid": "fef61e75-cec3-496c-a7bc-8368e4d02a04",
  "name": "pactolus",
  "sip_realm": "pactolus.com",
  "service_provider_sid": "a44985ae-3b03-4d23-a602-4971cfd60a65",
  "is_active": 1,
  "registration_hook": {
    "url": "https://app.pactolus.com/reg"
  },
  "device_calling_hook": {
    "url": "https://app.pactolus/deviceCall"
  }
}
```

#### Listing Accounts

**GET /v1/Accounts**

Lists all Accounts under a Service Provider (if a service provider scope api token is used) or under all Service Providers (if an admin scope api token is used).

#### Deleting Accounts

**DELETE /v1/Accounts/{AccountSid}**

Deletes a specific Account.

#### Creating an Account

**POST /v1/Accounts**

Creates a new Account.  This operation can only be peformed using a Service Provider or Admin scope api token.

Upon success, a 201 response is returned with a JSON body providing the account_sid ("sid") of the account that was created.

```xml
curl -X POST "https://{serviceUrl}/v1/Accounts" \
-H "accept: application/json" \
-H "Authorization: Bearer 38702987-c7a4-4685-a5bb-af378f9734de" \
-H "Content-Type: application/json" \
-d "{\"name\":\"brandon's account\",\"sip_realm\":\"sip.brandon.com\",\"registration_hook\":{\"url\":\"https://brandon.com/reg\",\"method\":\"POST\"},\"device_calling_hook\":{\"url\":\"https://brandon.com/dev\",\"method\":\"POST\"},\"service_provider_sid\":\"a44985ae-3b03-4d23-a602-4971cfd60a65\"}"

201 response
{
  "sid": "65cddc02-6cf8-476c-97a8-1941a96e143b"
}
```

### API Key
An api key is a token that is associated with an application and is used to authenticate requests on behalf of that application.  Api keys can be created and destroyed via the API.  API keys can have different scopes: Admin scope, service provider scope, and account scope.  

- Admin scope allows the bearer to make changes to global system properties and to create Service Providers.
- Service provider scope allows the bearer to create and manage Accounts under the Service Providerxs.
- Account scope allows the bearer to create applications and calls associated with the Account.


#### Conference participants
Conference participants refer to calls that are actively connected to a conference. You can mute or remove participants from a conference as well as retrieve a list of all participants, along with detailed information about each participant, in an active conference.

#### Conferences
Conferences represent a common endpoint that can mix the audio from multiple calls.  Conferences can be created, modified and deleted through the API.

#### Phone numbers
Phone numbers represent phone numbers that route to the jambonz platform, and may be associated with an application.  A Phone number may be associated with zero or one Application.  Phone numbers can be created and destroyed through the API, as well as being modified to point to a different application.

#### Queues
Queues represent an ordered collection of active calls that are parked (not connected to a far end).  Queues may be created and deleted through the API.

#### Queued calls
Queued calls are calls that have been assigned to a queue.

