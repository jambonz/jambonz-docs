# Microsoft Teams Integration

The instructions below assume that you are starting with a working system, and that you would like to add Microsoft Teams directing routing support.  In other words, your jambonz system will act as an SBC to Microsoft Teams, providing direct routing support and allowing teams users to make and receive calls from mobile phones and landlines.

These instructions cover only the changes needed on the jambonz side of things; please refer to the [Microsoft docs](https://docs.microsoft.com/en-us/microsoftteams/direct-routing-plan) for information regarding changes that must be made by your Teams admin to enable direct routing.

## Changes to the jambonz SBC

#### Add additional listen port 5061/tls

Edit the `/etc/systemd/system/drachtio.service` file to add a contact to listen for TLS traffic on port 5061:
```
ExecStart=/usr/local/bin/drachtio --daemon \
--contact sip:${LOCAL_IP};transport=udp --external-ip ${PUBLIC_IP} \
--contact sips:${LOCAL_IP}:5061;transport=tls --external-ip ${PUBLIC_IP} \
--contact sip:${LOCAL_IP};transport=tcp \
 --address 0.0.0.0 --port 9022
```

#### Generate TLS certificate

You will need to generate a TLS certificate that has a SAN for _both_ the SBC domain name _as well as_ a wildcard SAN for customer tenants, which are subdomains of the SBC domain name. 

For instance, an SBC domain name might be `teams.example.com` and customer tenants could be `cust1.teams.example.com`, `cust2.teams.example.com`, etc.

Generating such a TLS certificate can be done for free using [letsencrypt](https://letsencrypt.org/) as follows:
```
certbot certonly --manual --preferred-challenges=dns --email me@example.com \
--server https://acme-v02.api.letsencrypt.org/directory --agree-tos \ 
-d *.teams.example.com -d teams.example.com
```

#### Update drachtio config
Edit `/etc/drachtio.conf.xml` to reference the TLS certificate, key, and chain file by adding a `<tls/>` section within the `<sip/>` element
```xml
<sip>
	<tls>
	    <key-file>/etc/letsencrypt/live/teams.example.com/privkey.pem</key-file>
	   <cert-file>/etc/letsencrypt/live/teams.example.com/cert.pem</cert-file>
	   <chain-file>/etc/letsencrypt/live/teams.example.com/chain.pem</chain-file>
  </tls>

</sip>
```

#### Restart drachtio server 
```
sudo systemctl daemon-reload
sudo systemctl restart drachtio
```

#### Verify configuration changes

Execute the command `sudo systemctl status drachtio` to verify that the drachtio server restarted correctly, and examine the `/var/log/drachtio/drachtio.log` file to verify that it is now listening on port 5061 for TLS traffic, in addition to port 5060.  

The log file after startup should look something like this, with logging indicating that it is now listening for tls traffic on port 5061 and has successfully loaded the TLS certificate:

```
2020-08-05 13:06:39.574425 Starting drachtio version v0.8.6-rc1-2-g5faa42c74
2020-08-05 13:06:39.574546 DrachtioController::run tls key file:         /etc/letsencrypt/live/teams.jambonz.us/privkey.pem
2020-08-05 13:06:39.574554 DrachtioController::run tls certificate file: /etc/letsencrypt/live/teams.jambonz.us/cert.pem
2020-08-05 13:06:39.574562 DrachtioController::run tls chain file:       /etc/letsencrypt/live/teams.jambonz.us/chain.pem
2020-08-05 13:06:39.580917 SipTransport::logTransports - there are : 3 transports
2020-08-05 13:06:39.580933 SipTransport::logTransports - tcp/172.31.32.10:5060 (sip:172.31.32.10;transport=tcp, external-ip: , local-net: )
2020-08-05 13:06:39.580942 SipTransport::logTransports - tls/172.31.32.10:5061 (sips:172.31.32.10:5061;transport=tls, external-ip: 54.174.72.99, local-net: )
2020-08-05 13:06:39.580953 SipTransport::logTransports - udp/172.31.32.10:5060 (sip:172.31.32.10;transport=udp, external-ip: 54.174.72.99, local-net: ), mtu size: 4096
```

## Provision tenant FQDNs

Log into the jambonz provisioning GUI for your system, click 'Settings' in the left sidebar menu and enable the checkbox labeled "Enable Microsoft Teams Direct Routing".  Enter your SBC domain name (e.g. "teams.example.com") in the field labeled "SBC Domain Name" and click the Save button.

Next, click "MS Teams Tenant" in the left sidebar menu and add at least one tenant domain (e.g., "cust1.teams.example.com" in the example above), associate that tenant with a jambonz account and click the "Add Microsoft Teams Tenant" button.

## Restart drachtio apps

As the 'admin' user, restart the applications using pm2:

```
pm2 restart all
```

#### Verify connectivity

The jambonz SBC should now be sending OPTIONS pings to Microsoft Teams every 30 seconds or so.  

To verify, tail the log file `/var/log/drachtio.log` to see the OPTIONS request being sent, and the response received.  If all is working, then a 200 OK should be received as in the sample output below

```
2020-08-05 14:36:22.539782 send 372 bytes to tls/[52.114.148.0]:5061 at 14:36:22.539678:
OPTIONS sip:sip.pstnhub.microsoft.com SIP/2.0
Via: SIP/2.0/TLS 54.174.72.99;branch=z9hG4bK1HQpp1BtBQ30N
Max-Forwards: 70
From: <sip:teams.jambonz.us:5061;transport=tls>;tag=SX7B39veap93c
To: <sip:sip.pstnhub.microsoft.com>
Call-ID: dfe4b29f-51cb-1239-3b99-06ce423ffcdb
CSeq: 23765299 OPTIONS
Contact: <sip:teams.jambonz.us:5061;transport=tls>
Content-Length: 0

2020-08-05 14:36:22.564556 recv 376 bytes from tls/[52.114.148.0]:5061 at 14:36:22.564475:
SIP/2.0 200 OK
FROM: <sip:teams.jambonz.us:5061;transport=tls>;tag=SX7B39veap93c
TO: <sip:sip.pstnhub.microsoft.com>
CSEQ: 23765299 OPTIONS
CALL-ID: dfe4b29f-51cb-1239-3b99-06ce423ffcdb
VIA: SIP/2.0/TLS 54.174.72.99;branch=z9hG4bK1HQpp1BtBQ30N
CONTENT-LENGTH: 0
ALLOW: INVITE,ACK,OPTIONS,CANCEL,BYE,NOTIFY
SERVER: Microsoft.PSTNHub.SIPProxy v.2020.7.31.1 i.USWE2.3
```

At this point, once the necessary configuration changes have been made in Microsoft Teams admin, you should be able to test sending and receiving calls.  

To test, create a simple app of some kind and assign it as the outbound calling application for the customer tenant by going into the jambonz provisioning GUI, clicking "MS Teams Tenant" and modifying the customer tenant to associate the application to this tenant.  Any outbound calls from the tenant will now execute the selected application.


