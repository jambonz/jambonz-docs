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
