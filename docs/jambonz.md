# Overview 

jambonz is a specification for issuing call control commands via JSON payloads over HTTP connections.  These messages are sent from your application in response to web callbacks from a jambonz call control server, and they provide jambonz with your instructions on how to handle a call.  

When an incoming call is received by the platform, jambonz makes an HTTP request to the URL endpoint that is configured for that called number. Outbound calls that are initiated by the REST API are controlled in a similar way -- when invoking the REST API to launch a call, you provide a web callback url and in your response to the subsequent HTTP GET or POST to that url you then return a JSON payload describing the application that should govern the outbound call.

## Basic JSON message structure
The JSON payload (aka the "application") that you provide in response to a callback must be an array of objects, with each object describing a task that the platform shall perform.  These tasks are executed sequentially in the order they appear in the array.  Each task is identified by a verb (e.g. "dial", "gather", "hangup" etc) with associated detail and these verbs are described in more detail below.

If the caller hangs up during the execution of an application for that call, the current task is allowed to complete and any remaining tasks in the application are ignored.

Through additional callbacks that may be invoked during the execution of an application (typically as a result of those verbs which have an "action" property callback), the current application may be replaced with a new JSON application document.  In such a case, the new document begins executing, and any remaining tasks in the original document are discarded.

Each task object in the JSON array must include a "verb" property that describes the action to take.  Any additional information that the task needs to operate are provided as properties as well, e.g.:

```
{
  "verb": "say",
  "text": "Hi there!  Please leave a message at the tone and we will get back to you shortly.",
  "synthesizer": {
    "vendor": "google",
    "voice": "en-US-Wavenet-C"
  }
}
```

Some verbs allow other verbs to be nested; e.g. "gather" can have a nested "say" command in order to play a prompt and collect a response in one command:

```
{
  "verb": "gather",
  "action": "https://00dd977a.ngrok.io/gather",
  "input": ["speech", "dtmf"],
  "timeout": 16,
  "numDigits": 6,
  "recognizer": {
    "vendor": "google",
    "language": "en-US"
  },
  "say": {
    "text": "Please say or enter your six digit card number now",
    "synthesizer": {
      "vendor": "google",
      "voice": "en-US-Wavenet-C"
    }
  }
}
```

Altogether then, a simple example application which provides the basics of a voicemail application with transcription would look like this:
```
[
  {
    "verb": "say",
    "text": "Hi there!  Please leave a message at the tone and we will get back to you shortly.  Thanks, and have a great day!"
  },
  {
    "verb": "listen",
    "action": "http://example.com/voicemail",
    "url": "http://ws.example.com",
    "finishOnKey": "#",
    "metadata": {
      "topic": "voicemail"
    },
    "mixType": "mono",
    "playBeep": true,
    "timeout": 20,
    "transcribe": {
      "transcriptionCallback": "http://example.com/transcription"
    }
  },
  {
    "verb": "say",
    "text": "Thanks for your message.  We'll get back to you"
  }
]
```

Hey, did you see what we did there?  It's a voicemail application, but where is the recording url?  How do you retrieve the recording after the call?

There is none, and you don't.

Let's rethink this: 

Instead of making a recording -- which exposes your customer's PII since we now have to store sensitive data at rest in the platform -- how's about we instead send you a real-time audio stream over a secure websocket connection while the call is proceeding.  Annotate it with any metadata you need for tracking on your end, and we'll send that along as well.

Thus, you get the audio in real-time and we don't ever store your customer's sensitive data at rest.  Bam. Done.

## HTTP connection details
Each HTTP request that jambonz makes to one of your callbacks will include (at least) the following information either as query arguments (in a GET request) or in the body of the response as a JSON payload (in a POST request):

- callSid: a unique identifier for the call, in a [uuid](https://en.wikipedia.org/wiki/Universally_unique_identifier) format.
- applicationSid: a unique identifier for the jambonz application controlling this call
- accountSid: a unique identifier for the jambonz account associated with the application
- direction: the direction of the call, either 'inbound' or 'outbound'
- from: the calling party number
- to: the called party number
- callerId: the caller name, if known
- callStatus: current status of the call, see table below
- sipStatus: the most recent sip status code received or generated for the call

Additionally, the request **MAY** include

- parentCallSid: the callSid of a parent call to this call, if this call is a child call

And the initial webhook for a new incoming call will have:

- originatingSipTrunkName: name of the SIP trunk that originated the call to the platform
- originatingSipIP: the ip address and port of the sip gateway that originated the call

Finally, if you specify to use a POST method for the initial webhook for an incoming call, the JSON payload in that POST will also contain the entire incoming SIP INVITE request details in a 'sip' property (this is not provided if a GET request is used).  This can be useful if you need a detailed look at all of the SIP headers or the Session Description Protocol being offered.

Note also that the information that jambonz sends you with each HTTP request can be augmented by your application by using the [tag](#tag) verb.

You may optionally use [HTTP Basic Authentication](https://en.wikipedia.org/wiki/Basic_access_authentication) to protect your endpoints.

| call status value  | description | 
| ------------- |-------------| 
| trying | a new incoming call has arrived or an outbound call has just been sent|
| ringing | a 180 Ringing response has been sent or received |
| early-media | an early media connection has been established prior to answering the call (183 Session Progress) |
| in-progress | call has been answered |
| completed | an answered call has ended |
| failed | a call attempt failed |
| busy | a call attempt failed because the called party returned a busy status |
| no-answer | a call attempt failed because it was not answered in time |

Refer to the [Example messages](#example-messages) section to see further details.

## Initial state of incoming calls
When the jambonz platform receives a new incoming call, it responds 100 Trying to the INVITE but does not automatically answer the call.  It is up to your application to decide how to finally respond to the INVITE.  Your application can:

- answer the call, which connects the call to a media endpoint that can perform IVR functions on the call,
- outdial a new call, and bridge the two calls together, 
- reject the call, with a specified SIP status code and reason,
- redirect the call (i.e. generating a SIP 302 response), or
- establish an early media connection without answering the call.

The last is interesting and worthy of further comment.  The intent is to let you play audio to callers without necessarily answering the call.  You signal this by including an "earlyMedia" property with a value of true in the application.  When receiving this, the jambonz core will create an early media connection (183 Session Progress) if possible, as shown in the example below.

> Note: an early media connection will not be possible if the call has already been answered by an earlier verb in the application.  In such a scenario, the earlyMedia property is ignored.
```
[
  {
    "verb": "say",
    "earlyMedia": true,
    "text": "Please call back later, we are currently at lunch"
    "synthesizer": {
      "vendor": "google",
      "voice": "en-US-Wavenet-F"
    },
    {
      "verb": "sip:decline",
      "status": 480,
      "headers": {
        "Retry-After": 1800
      }
    }
  }
]
```
The say, play, gather, listen, and transcribe verbs all support the "earlyMedia" property.  

The dial verb supports a similar feature of not answering the inbound call unless/until the dialed call is answered via the "answerOnBridge" property.

## Speech integration
The platform makes use of text-to-speech as well as real-time speech recognition.  Currently, only google is supported for both text to speech and speech to text.  Other speech vendors will be supported in the future.

A JSON service key file containing GCP credentials for cloud speech services must be downloaded and installed on the jambonz feature servers to enable tts and speech recognition.

As part of the definition of an application, you can set defaults for the voice to use for speech synthesis as well as the language to use for speech recognition.  These can then be overridden by verbs in the application, by using the 'synthesizer' and 'recognizer' properties

# Supported Verbs
Each of the supported verbs are described below.

## dial

The dial command is used to create a new call by dialing out to a number, a registered sip user, or sip endpoint.  
```json
{
  "verb": "dial",
  "action": "/outdial",
  "callerId": "+16173331212",
  "answerOnBridge": true,
  "target": [
    {
      "type": "phone",
      "number": "+15083084809"
    },
    {
      "type": "sip",
      "sipUri": "sip:1617333456@sip.trunk1.com",
      "auth": {
        "user": "foo",
        "password": "bar"
      }
    },
    {
      "type": "user",
      "name": "spike@sip.example.com"
    }
  ]
}
```
As the example above illustrates, when you execute the 'dial' command you are making one or more outbound call attempts in an effort to create one new call, which can be bridged to a parent call. The `target` property specifies an array of call destinations (aka [endpoints](#target-types)) that will be attempted simultaneously.  

If multiple endpoints are specified in the `target` array, all targets are outdialed at the same time (e.g., "simring", or "blast outdial" as some folks call it) and the call will be connected to the first endpoint that answers the call and, optionally, completes a call screening application as specified in the `url` property.

There are three types of endpoints:

* a telephone phone number,
* a sip endpoint, identified by a sip uri, or
* a webrtc or sip client that has registered directly with your application.

You can use the following attributes in the `dial` command:

| option        | description | required  |
| ------------- |-------------| -----|
| action | URL of webhook to invoke when the call ends.  The 'action' URL may be either an absolute or relative URL.  If the latter, it is applied to the base URL of the original application, and if basic auth was used for the original request, then it will be for this request as well | no |
| answerOnBridge | If set to true, the inbound call will ring until the number that was dialed answers the call, and at that point a 200 OK will be sent on the inbound leg.  If false, the inbound call will be answered immediately as the outbound call is placed. <br/>Defaults to false. | no |
| callerId | The inbound caller's phone number, which is displayed to the number that was dialed. The caller ID must be a valid E.164 number. <br/>Defaults to caller id on inbound call. | no |
| confirmMethod | 'GET', 'POST' - http method to use on 'confirmUrl' callback. | no |
| confirmUrl | A specified URL for a document that runs on the callee's end after the dialed number answers but before the call is connected. This allows the caller to provide information to the dialed number, giving them the opportunity to decline the call, before they answer the call.  Note that if you want to run different applications on specific destinations, you can specify the 'url' property on the nested [target](#target-types) object.  | no |
| dialMusic | url that specifies a .wav or .mp3 audio file of custom audio or ringback to play to the caller while the outbound call is ringing. | no |
| headers | an object containing arbitrary sip headers to apply to the outbound call attempt(s) | no |
| listen | a nested [listen](#listen) action, which will cause audio from the call to be streamed to a remote server over a websocket connection | no |
| method | 'GET', 'POST' - http method to use on 'action' callback.  <br/>Defaults to POST.| no|
| target | array of to 10 [destinations](#target-types) to simultaneously dial. The first person (or entity) to answer the call will be connected to the caller and the rest of the called numbers will be hung up.| yes |
| timeLimit | max length of call in seconds | no |
| timeout | ring no answer timeout, in seconds.  <br/>Defaults to 60. | no |
| transcribe | a nested [transcribe](#transcribe) action, which will cause the call to be transcribed | no |

##### target types

*PSTN number*

| option        | description | required  |
| ------------- |-------------| -----|
| type | must be "phone" | yes |
| url | A specified URL for a document that runs on the callee's end after the dialed number answers but before the call is connected. This will override the confirmUrl property set on the parent dial verb, if any.| no |
| method | 'GET', 'POST' - http method to use on url callback.  <br/>Defaults to POST.| no|
| number | a telephone numnber in E.164 number | yes |

*sip endpoint*

| option        | description | required  |
| ------------- |-------------| -----|
| type | must be "sip" | yes |
| url | A specified URL for a document that runs on the callee's end after the dialed number answers but before the call is connected. This will override the confirmUrl property set on the parent dial verb, if any.| no |
| method | 'GET', 'POST' - http method to use on url callback.  <br/>Defaults to POST.| no|
| sipUri | sip uri to send call to | yes |
| auth | authentication credentials | no |
| auth.user | sip username | no |
| auth.password | sip password | no |

Using this approach, it is possible to send calls out a sip trunk.  If the sip trunking provider enforces username/password authentication, then supply the credentials in the `auth` property.

**a registered webrtc or sip user**

| option        | description | required  |
| ------------- |-------------| -----|
| type | must be "user" | yes |
| url | A specified URL for a document that runs on the callee's end after the dialed number answers but before the call is connected. This will override the confirmUrl property set on the parent dial verb, if any.| no |
| method | 'GET', 'POST' - http method to use on url callback.  <br/>Defaults to POST.| no|
| name | registered sip user, including domain (e.g. "joeb@sip.jambonz.org") | yes |

The `url` property that can be optionally specified as part of a target is a web callback that will be invoked when the outdial call is answered.  That callback should return an application that will run on the outbound call before bridging it to the inbound call.  If the application completes with the outbound call still in a stable/connected state, then the two calls will be bridged together.

This allows you to easily implement call screening applications (e.g. "You have a call from so-and-so.  Press 1 to decline").

## gather

The gather command is used to collect dtmf or speech input.

```json
{
  "verb": "gather",
  "action": "http://example.com/collect",
  "input": ["digits", "speech"],
  "finishOnKey": "#",
  "numDigits": 5,
  "timeout": 8,
  "recognizer": {
    "vendor": "google",
    "language": "en-US"
  },
  "say": {
    "text": "To speak to Sales press 1.  To speak to customer support press 2.",
    "synthesizer": {
      "vendor": "google",
      "voice": "en-US-Wavenet-F"
    }
  }
}
```

You can use the following options in the `gather` command:

| option        | description | required  |
| ------------- |-------------| -----|
| action | URL of webhook to invoke collected digits or speech.  The 'action' URL may be either an absolute or relative URL.  If the latter, it is applied to the base URL of the original application, and if basic auth was used for the original request, then it will be for this request as well | yes |
| finishOnKey | dmtf key that signals the end of input | no |
| input | array, specifying allowed types of input: ['digits'], ['speech'], or ['digits', 'speech'].  Default: ['digits'] | no |
| method | 'GET', 'POST' - http method to use on action callback.  <br/>Defaults to POST.| no|
| numDigits | number of dtmf digits expected to gather | no |
| partialResultCallback | url to send interim transcription results to. Partial transcriptions are only generated if this property is set. | no |
| play | nested [play](#play) command that can be used to prompt the user | no |
| recognizer.hints | array of words or phrases to assist speech detection | no |
| recognizer.language | language code to use for speech detection.  Defaults to the application level setting, or 'en-US' if not set | no |
| recognizer.profanityFilter | if true, filter profanity from speech transcription.  Default:  no| no |
| recognizer.vendor | speech vendor to use (currently only google supported) | no |
| say | nested [say](#say) command that can be used to prompt the user | no |
| timeout | The number of seconds of silence or inaction that denote the end of caller input.  The timeout timer will begin after any nested play or say command completes.  Defaults to 5 | no |

> Note: an HTTP POST will be used for both the `action` and the `partialResultCallback` since the body may need to contain nested JSON objects for speech details.

> Note: the `partialResultCallback` web callback should not return content; any returned content will be discarded.

## hangup

The hangup command terminates the call and ends the application.
```json
{
  "verb": "hangup",
  "headers": {
    "X-Reason" : "maximum call duration exceeded"
  }
}
```

You can use the following options in the `hangup` action:

| option        | description | required  |
| ------------- |-------------| -----|
| headers | an object containing SIP headers to include in the BYE request | no |


## listen

jambonz does not have a 'record' verb.    This is by design, for data privacy reasons.  

> _Recordings can contain sensitive and confidential information, and such data is never stored at rest in the jambonz core._

Instead, jambonz provides the 'listen' verb, where an audio stream(s) can be forked and sent in real-time to a customer application for processing.

The listen verb can also be nested in a [dial](#dial) verb, which allows the audio for a call between two parties to be sent to a remote websocket server.

To utilize the listen verb, the customer must implement a websocket server to receive and process the audio.  The endpoint should be prepared to accept websocket connections with a subprotocol name of 'audio.jambonz.org'.  

The listen verb includes a `url` property which is the url of the remote websocket server to send the audio to. The url may be an absolute or relative URL. HTTP Basic Authentication can optionally be used to protect the websocket endpoint by using the 'wsAuth' property.

The format of the audio data sent over the websocket is 16-bit PCM encoding, with a user-specified sample rate.  The audio is sent in binary frames over the websocket connection.  

Additionally, one text frame is sent immediately after the websocket connection is established.  This text frame contains a JSON string with all of the call attributes normally sent on an HTTP request (e.g. callSid, etc), plus 'sampleRate' and 'mixType' properties describing the audio sample rate and stream(s).  Additional metadata can also be added to this payload using the 'metadata' property as described in the table below.  Once the intial text frame containing the metadata has been sent, the remote side should expect to receive only binary frames, containing audio.  The remote side is not expected to send any data back over the websocket.

```json
{
  "verb": "listen",
  "url": "wss://myrecorder.example.com/calls/271314e6-b463-4980-b007-80defc181058:4433",
  "mixType" : "stereo"
}
```

You can use the following options in the `listen` action:

| option        | description | required  |
| ------------- |-------------| -----|
| action | URL of webhook to invoke when listen operation ends.  The information will include the duration of the audio stream, and also a 'digits' property if the recording was terminated by a dtmf key.  The 'action' URL may be either an absolute or relative URL.  If the latter, it is applied to the base URL of the original application, and if basic auth was used for the original request, then it will be for this request as well | yes |
| auth.username | HTTP basic auth username to use on action callback | no |
| auth.password | HTTP basic auth password to use on action callback | no |
| finishOnKey | The set of digits that can end the listen action | no |
| maxLength | the maximum length of the listened audio stream, in secs | no |
| metadata | arbitrary data to add to the JSON payload sent to the remote server when websocket connection is first connected | no |
| method | 'GET', 'POST' - http method to use on action callback.  <br/>Defaults to POST.| no|
| mixType | "mono" (send single channel), "stereo" (send dual channel of both calls in a bridge), or "mixed" (send audio from both calls in a bridge in a single mixed audio stream) Default: mono | no |
| playBeep | true, false whether to play a beep at the start of the listen operation.  Default: false | no |
| sampleRate | sample rate of audio to send (allowable values: 8000, 16000, 24000, 48000, or 64000).  Default: 8000 | no |
| timeout | the number of seconds of silence that terminates the listen operation.| no |
| transcribe | a nested [transcribe](#transcribe) verb | no |
| url | url of remote server to connect to | yes |
| wsAuth.username | HTTP basic auth username to use on websocket connection | no |
| wsAuth.password | HTTP basic auth password to use on websocket connection | no |

## play

The play command is used to stream recorded audio to a call.
```json
{
  "verb": "play",
  "url": "https://example.com/example.mp3"
}
```

You can use the following options in the `play` action:

| option        | description | required  |
| ------------- |-------------| -----|
| url | a single url or array of urls (will play in sequence) to a wav or mp3 file | yes |
| loop | number of times to play the url(s) | no (default: 1) |
| earlyMedia | if true and the call has not yet been answered, play the audio without answering call.  Defaults to false | no |

## redirect

The redirect action is used to transfer control to another JSON document taht is retrieved from the specified url.  All actions after `redirect` are unreachable and ignored.

```json
{
  "verb": "redirect",
  "action": "https://example.com/?foo=bar",
  "method": "GET",
  "auth": {
    "username": "foo",
    "password": "bar"
  }
}
```

You can use the following options in the `redirect` action:

| option        | description | required  |
| ------------- |-------------| -----|
| action | URL of webhook to retrieve document from.  The 'action' URL may be either an absolute or relative URL.  If the latter, it is applied to the base URL of the original application, and if basic auth was used for the original request, then it will be for this request as well | yes |
| method | 'GET', 'POST' - http method to use on url callback.  <br/>Defaults to POST.| no|
| auth.user | HTTP Basic Authorization username | no|
| auth.password | HTTP Basic Authorization password | no|

## say

The say command is used to send synthesized speech to the remote party. The text provided may be either plain text or may use SSML tags.  

```json
{
  "verb": "say",
  "text": "hi there!",
  "synthesizer" : {
    "vendor": "google",
    "voice": "en-AU-Wavenet-B"
  }
}
```

You can use the following options in the `say` action:

| option        | description | required  |
| ------------- |-------------| -----|
| text | text to speak; may contain SSML tags | yes |
| synthesizer.vendor | speech vendor to use (currently only google supported)| no |
| synthesizer.voice | voice to use.  Defaults to application setting.  | no |
| loop | the number of times a text is to be repeated; 0 means repeat forever.  Defaults to 1. | no |
| earlyMedia | if true and the call has not yet been answered, play the audio without answering call.  Defaults to false | no |

## sip:decline

The sip:decline action is used to reject an incoming call with a specific status and, optionally, a reason and SIP headers to include on the response.  

This action must be the first and only action returned in the JSON payload for an incoming call.  

The sip:decline action is a non-blocking action and the session ends immediately after the action is executed.

```json
{
  "verb": "sip:decline",
  "status": 480,
  "reason": "Gone Fishing",
  "headers" : {
    "Retry-After": 1800
  }
}
```

You can use the following options in the `sip:decline` action:

| option        | description | required  |
| ------------- |-------------| -----|
| status | a valid SIP status code in the range 4XX - 6XX | yes |
| reason | a brief description | no (default: the well-known SIP reasons associated with the specified status code |
| headers | SIP headers to include in the response | no

## sip:notify

The sip:notify action is used to a SIP NOTIFY request to one or more registered users.  The sip:notify action is a non-blocking action.

```json
{
  "verb": "sip:notify",
  "user": ["spike"],
  "contentType": "application/simple-message-summary",
  "content": [
    "Messages-Waiting: yes",
    "Message-Account: sip:spike@sip.example.com",
    "Voice-Message: 2/8 (0/2)"
  ],
  "headers": {
    "Subscription-State": "active"
  }
}
```

You can use the following options in the `sip:notify` action:

| option        | description | required  |
| ------------- |-------------| -----|
| user | user, or array of users, to send NOTIFY requests to | yes |
| contentType | value of SIP Content-Type header in NOTIFY | yes |
| content | body of NOTIFY request | yes |
| headers | object specifying arbitrary headers to apply to SIP NOTIFY request | no

> **Note**: the sip:notify verb is not currently implemented.

## sip:redirect

The sip:redirect command is used to redirect an incoming call to another sip server or network.  This must be the first and only command returned in the web callback for an incoming call.  A SIP 302 Moved response will be sent back to the caller.  

The sip:redirect command is a non-blocking and the session ends immediately after the command is executed.

```json
{
  "verb": "sip:redirect",
  "sipUri": "sip:123@gateway.example.com"
}
```

You can use the following options in the `sip:decline` command:

| option        | description | required  |
| ------------- |-------------| -----|
| sipUri | a sip uri that will appear in the Contact header of the 302 response to the caller | yes |

> **Note**: the sip:redirect verb is not currently implemented.

## tag

The tag verb is used to add properties to the standard call attributes that jambonz includes on every action or call status HTTP POST request.

> Note: because of the possible richness of the data, only subsequent POST requests will include this data.  It will not be included in HTTP GET requests.

The purpose is to simplify applications by eliminating the need to store state information if it can simply be echoed back to the application on each HTTP request for the call.

For example, consider an application that wishes to apply some privacy settings on outdials based on attributes in the initial incoming call.  The application could parse information from the SIP INVITE provided in the web callback when the call arrives, and rather than having to store that information for later use it could simply use the 'tag' verb to associate that information with the call.  Later, when an action or call status triggers the need for the application to outdial it can simply access the information from the HTTP POST body, rather than having to retrieve it from the cache of some sort.

Note that every time the tag verb is used, the collection of customer data is completely replaced with the new data provided.  This information will be provided back in all action or status notifications if POST method is used.  It will appear in property named 'customerData' in the JSON payload. 

```json
{
  "verb": "tag",
  "data" {
		"foo": "bar",
		"counter": 100,
		"list": [1, 2, "three"]
	}
}
```

After the above 'tag' verb has executed, web callbacks using POST would have a payload similar to this:
```
{
	"callSid": "df09e8d4-7ffd-492b-94d9-51a60318552c",
	"direction": "inbound",
	"from": "+15083084809",
	"to": "+15083728299",
	"callId": "f0414693-bdb6-1238-6185-06d91d68c9b0",
	"sipStatus": 200,
	"callStatus": "in-progress",
	"callerId": "f0414693-bdb6-1238-6185-06d91d68c9b0",
	"accountSid": "fef61e75-cec3-496c-a7bc-8368e4d02a04",
	"applicationSid": "0e0681b0-d49f-4fb8-b973-b5a3c6758de1",
	"originatingSipIP": "54.172.60.1:5060",
	"originatingSipTrunkName": "twilio",
	"customerData": {
		"foo": "bar",
		"counter": 100,
		"list": [1, 2, "three"]
	}
}
```

You can use the following options in the `tag` command:

| option        | description | required  |
| ------------- |-------------| -----|
| data | a JSON object containing values to be saved and included in future action or call status notifications (HTTP POST only) for this call | yes |

## transcribe

The transcribe verb is used to send real time transcriptions of speech to a web callback.

The transcribe command is only allowed as a nested verb within a dial or listen verb.  Using transcribe in a dial command allows a long-running transcription of a phone call to be made, while nesting within a listen verb allows transcriptions of recorded messages (e.g. voicemail).

```json
{
  "verb": "transcribe",
  "transcriptionCallback": "http://example.com/transcribe",
  "recognizer": {
    "vendor": "google",
    "language" : "en-US",
    "interim": true
  }
}
```

You can use the following options in the `transcribe` command:

| option        | description | required  |
| ------------- |-------------| -----|
| recognizer.dualChannel | if true, transcribe the parent call as well as the child call | no |
| recognizer.interim | if true interim transcriptions are sent | no (default: false) |
| recognizer.language | language to use for speech transcription | yes |
| recognizer.profanityFilter | if true, filter profanity from speech transcription.  Default:  no| no |
| recognizer.vendor | speech vendor to use (currently only google supported) | no |
| transcriptionCallback | url to invoke with transcriptions.  An HTTP POST will be sent to this url. | yes |

> **Note**: the `dualChannel` property is not currently implemented.

# Example messages

An example JSON payload for a webhook for an incoming call using a POST method:

```
{
	"direction": "inbound",
	"callSid": "1fe62f7c-ebb9-4b96-b75b-7d04ff2b195d",
	"accountSid": "fef61e75-cec3-496c-a7bc-8368e4d02a04",
	"applicationSid": "0e0681b0-d49f-4fb8-b973-b5a3c6758de1",
	"from": "+15083084809",
	"to": "+15083728299",
	"callerName": "+15083084809",
	"callId": "252a93d3-bdb2-1238-6185-06d91d68c9b0",
	"sipStatus": 100,
	"callStatus": "trying",
	"originatingSipIP": "54.172.60.2:5060",
	"originatingSipTrunkName": "twilio",
	"sip": {
		"headers": {
			"via": "SIP/2.0/UDP 3.10.235.99;rport=5060;branch=z9hG4bKgeBy6Fg863Z8N;received=172.31.3.33",
			"max-forwards": "70",
			"from": "<sip:+15083084809@3.10.235.99:5060>;tag=vQXQ3g5papXpF",
			"to": "<sip:+15083728299@172.31.3.33:5070>",
			"call-id": "252a93d3-bdb2-1238-6185-06d91d68c9b0",
			"cseq": "15623387 INVITE",
			"contact": "<sip:+15083084809@3.10.235.99:5060>",
			"user-agent": "Twilio Gateway",
			"allow": "INVITE, ACK, CANCEL, BYE, REFER, NOTIFY, OPTIONS",
			"content-type": "application/sdp",
			"content-length": "264",
			"X-CID": "f9221ea5e66a1d1f10a0b556933dc0c2@0.0.0.0",
			"X-Forwarded-For": "54.172.60.2:5060",
			"X-Originating-Carrier": "twilio",
			"Diversion": "<sip:+15083728299@public-vip.us1.twilio.com>;reason=unconditional"
		},
		"body": "v=0\r\no=root 1999455157 1999455157 IN IP4 3.10.235.99\r\ns=Twilio Media Gateway\r\nc=IN IP4 3.10.235.99\r\nt=0 0\r\nm=audio 49764 RTP/AVP 0 101\r\na=maxptime:150\r\na=rtpmap:0 PCMU/8000\r\na=rtpmap:101 telephone-event/8000\r\na=fmtp:101 0-16\r\na=sendrecv\r\na=rtcp:49765\r\na=ptime:20\r\n",
		"payload": [{
			"type": "application/sdp",
			"content": "v=0\r\no=root 1999455157 1999455157 IN IP4 3.10.235.99\r\ns=Twilio Media Gateway\r\nc=IN IP4 3.10.235.99\r\nt=0 0\r\nm=audio 49764 RTP/AVP 0 101\r\na=maxptime:150\r\na=rtpmap:0 PCMU/8000\r\na=rtpmap:101 telephone-event/8000\r\na=fmtp:101 0-16\r\na=sendrecv\r\na=rtcp:49765\r\na=ptime:20\r\n"
		}],
		"method": "INVITE",
		"version": "2.0",
		"uri": "sip:+15083728299@172.31.3.33:5070",
		"raw": "INVITE sip:+15083728299@172.31.3.33:5070 SIP/2.0\r\nVia: SIP/2.0/UDP 3.10.235.99;rport=5060;branch=z9hG4bKgeBy6Fg863Z8N;received=172.31.3.33\r\nMax-Forwards: 70\r\nFrom: <sip:+15083084809@3.10.235.99:5060>;tag=vQXQ3g5papXpF\r\nTo: <sip:+15083728299@172.31.3.33:5070>\r\nCall-ID: 252a93d3-bdb2-1238-6185-06d91d68c9b0\r\nCSeq: 15623387 INVITE\r\nContact: <sip:+15083084809@3.10.235.99:5060>\r\nUser-Agent: Twilio Gateway\r\nAllow: INVITE, ACK, CANCEL, BYE, REFER, NOTIFY, OPTIONS\r\nContent-Type: application/sdp\r\nContent-Length: 264\r\nX-CID: f9221ea5e66a1d1f10a0b556933dc0c2@0.0.0.0\r\nX-Forwarded-For: 54.172.60.2:5060\r\nX-Originating-Carrier: twilio\r\nDiversion: <sip:+15083728299@public-vip.us1.twilio.com>;reason=unconditional\r\nX-Twilio-AccountSid: AC58f23d38858ac262d6ee2e554b30c561\r\nX-Twilio-CallSid: CA708d85d118aacfcc794b730fa02bc40c\r\n\r\nv=0\r\no=root 1999455157 1999455157 IN IP4 3.10.235.99\r\ns=Twilio Media Gateway\r\nc=IN IP4 3.10.235.99\r\nt=0 0\r\nm=audio 49764 RTP/AVP 0 101\r\na=maxptime:150\r\na=rtpmap:0 PCMU/8000\r\na=rtpmap:101 telephone-event/8000\r\na=fmtp:101 0-16\r\na=sendrecv\r\na=rtcp:49765\r\na=ptime:20\r\n"
	}
}
```

An example JSON payload for a call status webhook for an incoming call using a POST method:
```
 {
 	"direction": "inbound",
 	"callSid": "1fe62f7c-ebb9-4b96-b75b-7d04ff2b195d",
 	"accountSid": "fef61e75-cec3-496c-a7bc-8368e4d02a04",
 	"applicationSid": "0e0681b0-d49f-4fb8-b973-b5a3c6758de1",
 	"from": "+15083084809",
 	"to": "+15083728299",
 	"callerName": "+15083084809",
 	"callId": "252a93d3-bdb2-1238-6185-06d91d68c9b0",
 	"sipStatus": 200,
 	"callStatus": "in-progress",
 	"originatingSipIP": "54.172.60.2:5060",
 	"originatingSipTrunkName": "twilio"
 }
```

An example JSON payload for a call status webhook for an outbound call using a POST method:

```
{
	"direction": "outbound",
	"callSid": "ddd6d4b2-ba3f-42fb-9845-8abdac047097",
	"parentCallSid": "1fe62f7c-ebb9-4b96-b75b-7d04ff2b195d",
	"accountSid": "fef61e75-cec3-496c-a7bc-8368e4d02a04",
	"applicationSid": "0e0681b0-d49f-4fb8-b973-b5a3c6758de1",
	"from": "+15083084809",
	"to": "+15084901000",
	"callerName": "+15083084809",
	"callId": "a5726393-bdaf-1238-9483-06d91d68c9b0",
	"callStatus": "in-progress",
	"sipStatus": 200
}
```
