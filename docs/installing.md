# Installing jambonz

## AWS

The quickest way to deploy jambonz currently is on AWS using the provided [terraform scripts](https://github.com/jambonz/jambonz-infrastructure).  Review [this quickstart video](/tutorials/#quickstart-deploying-jambonz-on-aws-in-10-minutes-or-less) for details.

## Otherwise..

If you are using a different hosted provider, or have your own hardware, there is a little more elbow grease required!  Follow the instructions below to create a jambonz deployment consisting of one SBC and one Feature Server.  You will also be provisioning a mysql server and a redis server.

### A. Provision servers
You'll need two servers -- one will be the public-facing SBC, while the other will be the feature server.  The SBC must have a public address; the Feature Server does not necessarily need a public address, but of course will need connectivity to the SBC, the mysql database, and the redis server.  If desired, you can install mysql and redis on the SBC server, but as long as they are reachable from both the SBC and the Feature Server you'll be fine.  We will be using ansible to build up the servers, which means from your laptop you need ssh connectivity to both the SBC and the Feature Server.

The base software distribution for both the SBC and the Feature Server should be Debian 9.  A vanilla install that includes sudo and python is all that is needed (python is used by [ansible](https://www.ansible.com/), which we will be using to build up the servers in the next step).

### B. Use ansible to install base software
If you don't have ansible installed on your laptop, install it now [following these instructions](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).

Check out the following github repos to your laptop:

[ansible-role-drachtio](https://github.com/davehorton/ansible-role-drachtio)

[ansible-role-fsmrf](https://github.com/davehorton/ansible-role-fsmrf)

[ansible-role-nodejs](https://github.com/davehorton/ansible-role-nodejs)

[ansible-role-rtpengine](https://github.com/davehorton/ansible-role-rtpengine)

For the SBC, create an ansible playbook that looks like this, and run it:
```yaml
---
- hosts: all
  become: yes
  vars: 
    drachtioBranch: develop
    rtp_engine_version: mr8.5
  vars_prompt:
    - name: "cloud_provider"
      prompt: "Cloud provider: aws, gcp, azure, digital_ocean"
      default: none
      private: no
  roles:
    - ansible-role-drachtio
    - ansible-role-nodejs
    - ansible-role-rtpengine
```

and for the Feature Server, create an ansible playbook that looks like this, and run it:
```yaml
---
- hosts: all
  become: yes
  vars:
    drachtioBranch: develop
    build_with_grpc: true
  vars_prompt:
    - name: "cloud_provider"
      prompt: "Cloud provider: aws, gcp, azure, digital_ocean"
      default: none
      private: no
  roles:
    - ansible-role-drachtio
    - ansible-role-nodejs
    - ansible-role-fsmrf
```
### C. Create mysql database
You need to install a mysql database server.  Example instructions for installing mysql are provided [here](https://dev.mysql.com/downloads/).

Once the mysql server is installed, create a new database named 'jambones' with an associated username 'admin' and a password of your choice.  For the remainder of these instructions, we'll assume a password of 'JambonzR0ck$' was assigned, but you may create a password of your own choosing.

Once the database and user has been created, then create [this schema](https://github.com/jambonz/jambonz-api-server/blob/master/db/jambones-sql.sql).

Once the database schema has been created, run [this database script](https://github.com/jambonz/jambonz-api-server/blob/master/db/create-admin-token.sql) as well as [this database script](https://github.com/jambonz/jambonz-api-server/blob/master/db/create-default-account.sql) to seed the database with initial data.

### D. Create redis server
Install redis somewhere in your network by following [these instructions](https://redis.io/topics/quickstart) and save the redis hostname that you will use to connect to it.

### E. Configure SBC
Your SBC should have both a public IP and a private IP.  The public IP needs to be reachable from the internet, while the private IP should be on the internal subnet, and thus reachable by the Feature Server.

> In the examples below, we assume that the public IP is 190.144.12.220 and the private IP is 192.168.3.11.  Your IPs will be different of course, so substitute the correct IPs in the changes below.

#### drachtio configuration

In `/etc/systemd/system/drachtio.service` change this line:

```
ExecStart=/usr/local/bin/drachtio --daemon
```
to this:

```
ExecStart=/usr/local/bin/drachtio --daemon \
--contact sip:192.168.3.11;transport=udp --external-ip 190.144.12.220 \
--contact sip:192.168.3.11;transport=tcp \
--address 0.0.0.0 --port 9022
```
**or**, if you plan on enabling Microsoft Teams routing, to this:
```
ExecStart=/usr/local/bin/drachtio --daemon \
--contact sip:192.168.3.11;transport=udp --external-ip 190.144.12.220 \
--contact sips:192.168.3.11:5061;transport=tls --external-ip 190.144.12.220 \
--contact sip:192.168.3.11;transport=tcp \
--address 0.0.0.0 --port 9022
```
Then, reload and restart the drachtio server
```
systemctl daemon-reload
systemctl restart drachtio
```
After doing that, run `systemctl status drachtio` and check `/var/log/drachtio/drachtio.log` to verify that the drachtio server started properly and is listening on the specified IPs and ports.

#### rtpengine configuration

In `/etc/systemd/system/drachtio.service` change this line:

```
ExecStart=/usr/local/bin/rtpengine --interface 192.168.3.11!192.168.3.11 \
```
to this:
```
ExecStart=/usr/local/bin/rtpengine \
--interface private/192.168.3.11 \
--interface public/192.168.3.11!190.144.12.220 \
```
Then, reload and restart rtpengine
```
systemctl daemon-reload
systemctl restart rtpengine
```
After doing that, run `systemctl status rtpengine` to verify that rtpengine is running with the defined interfaces.

### F. Configure Feature Server