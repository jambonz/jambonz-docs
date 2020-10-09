# Sending and receiving SMS

The jambonz REST API and webhooks support sending and receiving SMS messages using HTTP-based APIs from SMS providers.  Since there is no such standard HTTP-based API across providers, it is necessary to build a small plug-in library that provides the glue needed to integrate with a specific provider.  

Currently, plug-ins have been created for [Peerless Communications](https://github.com/jambonz/messaging-peerless), [382 Communications](https://github.com/jambonz/messaging-382com), and [Simwood](https://github.com/jambonz/messaging-simwood), so if you use any of these providers for SIP trunking you can also take advantage of sending and receiving SMS through them.  

If you use any other provider you can create a plug-in for that, and instructions are given on how to do that at the end of this article.

## Configuration

The jambonz session border controller (SBC) handles sending and receiving all SMS traffic to and from SMS providers.

For inbound SMS, you will need to tell the SMS provider the HTTP endpoint/URL to make an HTTP(s) POST to when an incoming SMS is received to one of your numbers.

For outbound SMS, the provider will provide you with a URL to make an HTTP(S) POST to when you want to send out an SMS, and optionally may provide you with username/password authentication credentials.

This information is encoded into an environment variable that made available to the jambonz-api-server Node.js application.

To configure the SBC to support SMS, then, you would perform the following steps:

1. Instruct your provider to send to an HTTP(s) POST requests to an endpoint on the SBC with path `/v1/messaging/{providerName}`.  
For example, a full URL might be `https://api.mycompany.com/v1/messaging/peerless`, if you have set up DNS records for your SBC and secured it with https.

2. Add an environment variable that is passed to the `jambonz-api-server` Node.js application that specifies which plugin module to invoke when an incoming SMS is received for the named partner, and which URL to send outgoing SMS to.  

The environment variable will be a JSON string representing an object with keys named after the provider(s), and value providing the needed information; e.g.
```
{
	"peerless": {
		"module": "@jambonz/messaging-peerless",
		"options": {
			"url": "https://BASEURL/partners/messageReceiving/your-account-no/submitMessage",
			"auth": {
				"username": "your-user",
				"password": "your-password"
			}
		}
	}
}
```
The stringified environment variable should appear in the `~admin/apps/ecosystem.config.js` configuration file on your SBC, and assigned to the JAMBONES_MESSAGING environment variable, e.g.:

```
  env: {
    JAMBONES_MESSAGING: '{"peerless":{"module":"@jambonz/messaging-peerless","options":{"url":"https://peerless-based-url:443/partners/messageReceiving/your-account-no/submitMessage","auth":{"username":"your-user","password":"your-password"}}}}'
  }
```
It is possible to support multiple SMS providers by including each with a different key and configuration in the JSON string.

Once this is done, and the jambonz-api-server is restarted with the new configuration, you will be able to receive and send SMS with your provider.

## Receiving SMS via web callback
When an incoming SMS is received on one of your DIDs, the application associated with that DID is retrieved and the messaging webhook, if defined, is invoked with a payload like this:
```
{
  provider: 'peerless',
  from: '15083084809',
  to: ['15085710838'],
  text: 'Your reservation has been booked.'
}
```

## Sending SMS via 'message' verb
You can send an SMS from an application by using the `message` verb, e.g.
```
{
  verb: 'message'
  provider: 'peerless',
  from: '15085710838',
  to: '15083084809',
  text: 'Thank you.'
}
```

## Sending SMS via REST api
You can send an SMS via the REST api as per this curl example:
```bash
curl -X POST "https://api.jambonz.us/v1/Accounts/9351f46a-678c-43f5-b8a6-d4eb58d131af/Messages" \
-H  "accept: application/json" \
-H  "Authorization: Bearer 32006475-d3f5-44b6-a4bd-8aaa682d735b" \
-H  "Content-Type: application/json" \
-d "{\"from\":\"13322060388\",\"to\":\"+15083084809\",\"provider\":\"peerless\",\"text\":\"please call when you can\"}
```
The response payload will provide a unique messageSid along with the raw response from the SMS provider:
```
{
  "sid":"6cc512e7-1dea-4cf6-bcca-e5a638884524",
  "providerResponse": {
    "responseText":"OK. Message Submitted Successfully"
  }
}
```

## Adding support for an SMS provider
To add support for a new SMS provider you must create a npmjs package that exports several functions:

```
fromProviderFormat(opts, url, payload)
```
translates an incoming SMS from the provider into a standard format for application processing.

```
formatProviderResponse(messageSid)
```
this function is optional; it should be exported if the provider requires any special content in the 200 OK response to an incoming SMS.

```
sendSms(opts, payload)
```
this function sends an outgoing SMS, translating the payload from standard application format to the provider-specified format.

Please review the examples to get a sense of what is required:

- [https://github.com/jambonz/messaging-peerless](https://github.com/jambonz/messaging-peerless)
- [https://github.com/jambonz/messaging-382com](https://github.com/jambonz/messaging-382com)
- [https://github.com/jambonz/messaging-simwood](https://github.com/jambonz/messaging-simwood)
