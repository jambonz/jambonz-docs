# Overview
The platform allows sip clients to register, make and receive calls.  Managing sip registrations is a shared  activity between the platform and the customer application.  The platform handles the sip messaging aspects, but the determination of whether to authenticate a specific request is the responsibility of the application, which is notified of incoming REGISTER requests by means of the registration webhook.

When the platform receives an incoming sip register request, the registering sip domain is first checked to see if there is a register webhook provisioned for the that domain.  If there is no webhook provisioned for that domain, a 403 Forbidden response is sent back to the client.

Otherwise, the platform will challenge the REGISTER request with a 401 Unauthorized response containing a digest challenge.

If the sip client then sends a REGISTER request with and Authorization header, the platform generates an http POST request to the registered webhook.  The Content-Type of the POST is application/json and the body contains the following elements, as provided in  the Authorization sip header of the incoming REGISTER request.
```
{
  "method": "REGISTER",
  "realm": "example.com",
  "username": "foo",
  "expires": 3600,
  "nonce": "InFriVGWVoKeCckYrTx7wg==",
  "uri": "sip:example.com",
  "algorithm": "MD5",
  "qop": "auth",
  "cnonce": "03d8d2aafd5a975f2b07dc90fe5f4100",
  "nc": "00000001",
  "response": "db7b7dbec7edc0c427c1708031f67cc6"
}
```
The application, with knowledge of the password associated with the provided username and password, then performs [digest authentication](https://tools.ietf.org/html/rfc2617) to authenticate the request using the information provided, including the calculated response value.

Regardless of whether the request is authenticated or not, the application should respond with a 200 OK to the http POST and with a JSON body.

The JSON body in the response if the request is authenticated should simply contain a `status` attribute with a value of `ok`, e.g.:
```
{
  "status": "ok"
}
```

If the application wishes to enforce a shorter expires value, it may include that value in the response, e.g.:
```
{
  "status": "ok",
  "expires": 1800
}
```

The JSON body in the response if the request is _not_ authentication should contain a status of `fail`, and optionally a `msg` attribute, e.g.
```
{
  "status": "fail",
  "msg" : "invalid password"
}
```
