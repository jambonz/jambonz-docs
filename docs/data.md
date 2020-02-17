# jambones data architecture

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

## data element identifiers
Instances of data model entities are publicly identified with a unique value known as a "sid".  The documentation will therefore frequently refer to the following identifiers:

- *Account Sid* - identifies an account.
- *Application Sid* - identifies an application.
- *Call Sid* - identifies a specific call.
- *Parent Call Sid* - identifies the call that another call is bridged to.

This is not an exhaustive list, since all data elements have a similar unique identifier (e.g. a service provider sid uniqely identifies a service provider, a carrier sid identifies a specific voip carrier, etc).

## In-memory database

We are currently using redis as an in-memory database for transient data such as sip registrations and calls in progress.  The database structure is defined below.

### calls in progress
> `call:{accountSid}:{callSid}` - a hash of call data

> `active-call-sids` - sorted set of call keys, with score = time of entry

Calls in progress are tracked in two related data structures:

First, each call is represented in the database as a hash that is keyed by `call:{accountSid}:{callSid}`.  This allows for easy retrieval of individual calls, as well as all calls for an account.  The hash contains the same information that is provided with each webhook for the call.

Secondly, each call key is then added to a sorted set named `active-call-sids`.  The score associated with each key is its time of entry (`Date.now()`).  This allows for easy scanning of keys from oldest to newest when it is necessary to purge them.

The reason for having the sorted set is for efficiency when searching for a subset of calls matching a pattern -- most specifically when retrieving all of the calls associated with an account.  The SCAN/ZSCAN command in redis is the most efficient and recommended way to do this.

The hash data for a new call is initially added with an initial expiry of twelve hours, but when the call ends the expiry is then set to one hour.  Thus all call data naturally expires on its own from the database.  The call keys in the sorted set are purged periodically by the jambonz-api-server.

### sip registrations
> ``user:{name}@{domain}` - a hash of registration data

Active sip registrations are represented in the database as a has that is keyed by `user:{name}@{domain}`, where "domain" is the sip domain associated with an account.  The following information is stored in the hash for each registration:

- the Contact header provided in the REGISTER request
- the IP:port of the SBC that the user/device registered to
- the protocol that the user/device is using (i.e., udp, tcp, wss)
- the actual source IP and port that the SBC received the REGISTER from (usually different than the Contact header since sip devices are often behind a router)

The hash data for a registration has an expiry equal to the registration interval granted, so it will naturally expire unless the registration is renewed by the device.  

When devices are detected as being behind a router or nat device, the registration interval is decreased to 30 seconds to force the device to register frequently in order to keep a pinhole open on the customer router.  This is needed in order to to enable incoming calls sent to the device to be able to enter the customer network.

## mysql database

A mysql database is used to provision service information related to accounts, SIP gateways and phone numbers that 