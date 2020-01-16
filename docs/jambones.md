# Overview 

jambones is a specification for issuing call control commands via JSON messages.  These messages are sent from your application in response to web callbacks, and they provide the jambones platform with your instructions on how it should handle a call.  

When an incoming call is received by the platform, jambones makes an HTTP request to the URL endpoint that is configured for that called number. Outbound calls that are initiated by the REST API are controlled in the same way -- when invoking the REST API to launch a call, you provide a web callback url and in your response to that callback you then return a JSON payload describing the application that should govern the outbound call.

## Basic JSON message structure
The JSON object (aka the "application") that you provide in response to a callback must be an array of objects, with each object describing a task that the platform shall perform.  These tasks are executed sequentially in the order they appear in the array.  Each task is identified by a verb (e.g. "dial", "gather", "hangup" etc) with associated detail and these verbs are described in more detail below.

If the caller (or called party, in the case of a REST-initiated outdial) hangs up during the execution of an application, the current task is allowed to complete and any remaining tasks in the application are ignored.

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
  "language": "en-US",
  "say": {
    "text": "Please say or enter your six digit card number now",
    "synthesizer": {
      "vendor": "google",
      "voice": "en-US-Wavenet-C"
    }
  }
}
```

Altogether then, a simple example application which provides the basics of a voicemail application with transcription included would look like this:
```
[
  {
    "verb": "say",
    "text": "Hi there!  Please leave a message at the tone and we will get back to you shortly.  Thanks, and have a great day!",
    "synthesizer": {
      "vendor": "google",
      "voice": "en-US-Wavenet-C"
    }
  },
  {
    "verb": "listen",
    "finishOnKey": "#",
    "metadata": {
      "topic": "voicemail"
    },
    "mixType": "mono",
    "playBeep": true,
    "timeout": 20,
    "url": "http://bd0bf038.ngrok.io",
    "transcribe": {
      "action": "https://00dd977a.ngrok.io/transcription",
      "recognizer": {
        "vendor": "google",
        "language": "en-US"
      }
    }
  }
]
```
#### Alternate JSON structure
For those who prefer the readability of seeing verbs appear as object keys, the following syntax is also allowed:
```
[
  {
    "say": {
      "text": "Hi there!  Please leave a message at the tone and we will get back to you shortly.  Thanks, and have a great day!",
      "synthesizer": {
        "vendor": "google",
        "voice": "en-US-Wavenet-C"
      }
    }
  },
  {
    "listen": {
      "finishOnKey": "#",
      "metadata": {
        "topic": "voicemail"
      },
      "mixType": "mono",
      "playBeep": true,
      "timeout": 20,
      "url": "http://bd0bf038.ngrok.io",
      "transcribe": {
        "action": "https://00dd977a.ngrok.io/transcription",
        "recognizer": {
          "vendor": "google",
          "language": "en-US"
        }
      }
    }
  }
]
```

## Initial state of incoming calls
When the jambones platform receives a new incoming call, it responds 100 Trying to the INVITE but does not automatically answer the call.  It is up to your application to decide how to finally respond to the INVITE:

- it can answer the call (i.e. generating a SIP 200 OK response, and connecting the call to a media endpoint that can play media and collect responses), 
- it can simply reject the call, with a specified non-success SIP final response
- it can perform a SIP redirect (i.e. generating a SIP 302 response), or
- it can establish an early media connection (i.e. generating a SIP 183 Session Progress response, and connecting the call to a media endpoint).

The last is interesting and worthy of further comment.  The intent is to let you play audio to callers without necessarily answering the call.  You signal this by including an "earlyMedia" property with a value of true in the application.  When receiving this, the jambones core will create an early media connection if possible as in the example below.

> Note: an early media connection will not be possible if the call has already been answered.  In such a scenario, the earlyMedia property is ignored.
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
The say, play, gather, listen, and transcribe verbs all support the "earlyMedia" property.  The dial verb supports a similar feature of not answering the inbound call unless/until the dialed call is answered via the "answerOnBridge" property.

## Speech 
The platform makes use of text-to-speech as well as real-time speech recognition.  Currently, only google is supported for both text to speech and speech to text.  Other speech vendors will be supported in the future.

A JSON service key file containing GCP credentials for cloud speech services must be downloaded and installed on the jambones feature servers to enable tts and speech recognition.

# Supported Verbs
Each of the supported verbs are described below.

## dial

The dial command is used to create a new call by dialing out to a number, a registered sip user, or sip endpoint.  
```json
{
  "verb": "dial",
  "action": "http://example.com/outdial",
  "callerId": "+16173331212",
  "answerOnBridge": true,
  "statusCallback": "http://example.com/callStatus",
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
As the example above illustrates, when you execute the 'dial' command you are making one or more outbound call attempts in an effort to create one new call, which can be bridged to a parent call. The `target` property specifies an array of call destinations (aka [endpoints](#target-types)) that will be attempted simultaneously.  If only a single destination is to be called, the value of the `target` key can simply be an object specifying a single endpoint.

If multiple endpoints are specified, the call will be connected to the first endpoint that answers the call or returns an early media response (i.e. 183 Session Progress).

There are three types of endpoints:

* a telephone phone number,
* a sip endpoint, identified by a sip uri, or
* a webrtc or sip client that has registered directly with your application.

You can use the following attributes in the `dial` command:

| option        | description | required  |
| ------------- |-------------| -----|
| action | webhook to invoke when call ends | no |
| answerOnBridge | If set to true, the inbound call will ring until the number that was dialed answers the call, and at that point a 200 OK will be sent on the inbound leg.  If false, the inbound call will be answered immediately as the outbound call is placed. <br/>Defaults to false. | no |
| callerId | The inbound caller's phone number, which is displayed to the number that was dialed. The caller ID must be a valid E.164 number. <br/>Defaults to caller id on inbound call. | no |
| dialMusic | URL to a .wav or .mp3 file to play to caller during outdial | no |
| headers | an object containing arbitrary sip headers to apply to the outbound call attempt(s) | no |
| listen | a nested [listen](#listen) action, which will cause audio from the call to be streamed to a remote server over a websocket connection | no |
| method | 'GET', 'POST' - http method to use on 'action' callback.  <br/>Defaults to POST.| no|
| statusCallback | url to send call status events to for any new call legs generated | no |
| statusCallbackMethod | 'GET', 'POST' - http method to use on statusCallback callback. <br/>Defaults to POST. | no |
| target | array of targeted [endpoints](#target-types) (may be a single endpoint).  If multiple, all endpoints are attempted simultaneously | yes |
| timeLimit | max length of call in seconds | no |
| timeout | ring no answer timeout, in seconds.  <br/>Defaults to 60. | no |
| transcribe | a nested [transcribe](#transcribe) action, which will cause the call to be transcribed | no |

##### target types

*PSTN number*

| option        | description | required  |
| ------------- |-------------| -----|
| type | must be "phone" | yes |
| url | A specified URL for a document that runs on the callee's end after the dialed number answers but before the call is connected. | no |
| method | 'GET', 'POST' - http method to use on url callback.  <br/>Defaults to POST.| no|
| number | a telephone numnber in E.164 number | yes |

*sip endpoint*

| option        | description | required  |
| ------------- |-------------| -----|
| type | must be "sip" | yes |
| url | A specified URL for a document that runs on the callee's end after the dialed number answers but before the call is connected. | no |
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
| url | A specified URL for a document that runs on the callee's end after the dialed number answers but before the call is connected. | no |
| method | 'GET', 'POST' - http method to use on url callback.  <br/>Defaults to POST.| no|
| name | registered sip user, including domain (e.g. "joeb@sip.jambones.org") | yes |


## gather

The gather command is used to collect dtmf or speech input.

```json
{
  "verb": "gather",
  "action": "http://example.com/collect",
  "input": ["dtmf"],
  "finishOnKey": "#",
  "numDigits": 5,
  "timeout": 8,
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
| action | web callback to invoke with collected digits or speech | yes |
| finishOnKey | dmtf key that signals the end of input | no |
| hints | array of words or phrases to assist speech detection | no |
| input | array, specifying allowed types of input: ['dtmf'], ['speech'], or ['dtmf', 'speech'].  Default: ['dtmf'] | no |
| language | language code to use for speech detection.  Default: 'en-US' | no |
| numDigits | number of dtmf digits expected to gather | no |
| partialResultCallback | url to send interim transcription results to. Partial transcriptions are only generated if this property is set. | no |
| play | nested [play](#play) command that can be used to prompt the user | no |
| profanityFilter | if true, filter profanity from speech transcription.  Default:  no| no |
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

jambones does not have a 'record' verb.    This is by design, for data privacy reasons.  

> _Recordings can contain sensitive and confidential information, and such data is never stored at rest in the jambones core._

Instead, jambones provides the 'listen' verb, where an audio stream(s) can be forked and sent in real-time to a customer application for processing.

The listen verb includes a `url` property which is the remote url of a websocket server to send the audio to. The audio format is 16-bit PCM encoding, with a user-specified sample rate.  The audio is sent in binary frames over the websocket connection.  Optionally, text frames can be sent as well -- these are used to send user-specified metadata at the start of the call, and DTMF entries during the call.

To utilize the listen verb, the customer must implement a websocket server to receive and process the audio.  The endpoint should be prepared to accept websocket connections with a subprotocol name of audio.jambones.org.  

(*TBD: link for more detail on the protocol, json metadata etc*)

The listen verb can also be nested in a [dial](#dial) verb, which allows the audio for a call between two parties to be sent to a remote websocket server.


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
| finishOnKey | The set of digits that can end the listen action | no |
| maxLength | the maximum length of the listened audio stream, in secs | no |
| metadata | arbitrary JSON payload to send to remote server when websocket connection is first connected | no |
| mixType | "mono" (send single channel), "stereo" (send dual channel of both calls in a bridge), or "mixed" (send audio from both calls in a bridge in a single mixed audio stream) Default: mono | no |
| passDtmf | if true, send JSON text messages over the websocket indicating when DMTF keys have been pressed.  Default: no | no |
| playBeep | true, false whether to play a beep at the start of the listen operation.  Default: false | no |
| sampleRate | sample rate of audio to send (allowable values: 8000, 16000, 24000, 48000, or 64000).  Default: 8000 | no |
| timeout | the number of seconds of silence that terminates the listen operation.| no |
| transcribe | a nested [transcribe](#transcribe) verb | no |
| url | url of remote server to connect to | yes |

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
  "url": "https://example.com/?foo=bar"
}
```

You can use the following options in the `redirect` action:

| option        | description | required  |
| ------------- |-------------| -----|
| url | url to retrieve document from | yes |

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
| synthesizer.vendor | speech vendor to use (currently only google supported)| yes |
| synthesizer.voice | voice to use  | yes |
| loop | the number of times a text is to be repeated; 0 means repeat forever.  Defaults to 1. | no |
| earlyMedia | if true and the call has not yet been answered, play the audio without answering call.  Defaults to false | no |

*TBD: list of speech vendors supported (currently only google)*

[list of google voices](#google-tts-voices)


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

## transcribe

The transcribe verb is used to send real time transcriptions of speech to a web callback.

The transcribe command is only allowed as a nested verb within a dial or listen verb.  Using transcribe in a dial command allows a long-running transcription of a phone call to be made, while nesting within a listen verb allows transcriptions of recorded messages (e.g. voicemail to be created.

```json
{
  "verb": "transcribe",
  "action": "http://example.com/transcribe",
  "language" : "en-US",
  "source" : "both",
  "interim": true,
  "vendor": "google"
}
```

You can use the following options in the `transcribe` command:

| option        | description | required  |
| ------------- |-------------| -----|
| action | web callback to invoke with transcriptions | yes |
| interim | if true interim transcriptions are sent | no (default: false) |
| language | language to use for speech transcription | yes |
| source | call uuid to perform transcription on, or "both" to transcribe both calls in a bridge | no (default: original call) |
| vendor | speech vendor to use (currently only google supported) | no |

