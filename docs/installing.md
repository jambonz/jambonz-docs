# Installing jambonz

## AWS

The quickest way to deploy jambonz currently is on AWS using the provided [terraform scripts](https://github.com/jambonz/jambonz-infrastructure).  Review [this quickstart video](/tutorials/#quickstart-deploying-jambonz-on-aws-in-10-minutes-or-less) for details.

## Otherwise..

If you are using a different hosted provider, or have your own hardware, there is a little more elbow grease required!  Follow the instructions below to create a jambonz deployment consisting of one SBC and one Feature Server.  You will also be provisioning a mysql server and a redis server.

### A. Provision servers
You'll need two servers -- one will be the public-facing SBC, while the other will be the feature server.  The SBC must have a public address; the Feature Server does not necessarily need a public address, but of course will need connectivity to the SBC, the mysql database, the redis server, and outbound connectivity to the internet in order to complete the install.  

If desired, you can install mysql and redis on the SBC server, but as long as they are reachable from both the SBC and the Feature Server you'll be fine.  We will be using ansible to build up the servers, which means from your laptop you need ssh connectivity to both the SBC and the Feature Server.

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

In `/etc/systemd/system/rtpengine.service` change this line:

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
> Note: rtpengine logs to `/var/log/daemon.log`.

#### Install drachtio apps

Choose a user to install the drachtio applications under -- the instructions below assume the `admin` user; if you use a different user than edit the instructions accordingly (note: the user must have sudo priviledges).  

Execute the following commands from the home directory of the install user:

```
mkdir apps && cd $_
git clone https://github.com/jambonz/sbc-outbound.git 
git clone https://github.com/jambonz/sbc-inbound.git 
git clone https://github.com/jambonz/sbc-registrar.git
git clone https://github.com/jambonz/sbc-call-router.git 
git clone https://github.com/jambonz/jambonz-api-server.git 
git clone https://github.com/jambonz/jambonz-webapp.git

cd sbc-inbound && sudo npm install --unsafe-perm
cd ../sbc-outbound && sudo npm install --unsafe-perm
cd ../sbc-registrar && sudo npm install --unsafe-perm
cd ../sbc-call-router && sudo npm install --unsafe-perm
cd ../jambonz-api-server && sudo npm install --unsafe-perm
cd ../jambonz-webapp && sudo npm install --unsafe-perm && npm run build

sudo -u admin bash -c "pm2 install pm2-logrotate"
sudo -u admin bash -c "pm2 set pm2-logrotate:max_size 1G"
sudo -u admin bash -c "pm2 set pm2-logrotate:retain 5"
sudo -u admin bash -c "pm2 set pm2-logrotate:compress true"

sudo chown -R admin:admin  /home/admin/apps
```
Next, edit this file: `~/apps/jambonz-webapp/.env`.  Change this:
```
REACT_APP_API_BASE_URL=http://[ip]:[port]/v1
```
to this:
```
REACT_APP_API_BASE_URL=http://190.144.12.220:3000/v1
```
> Note: again, substitute the public IP of your own SBC in the above

Next, copy this file below into `~/apps/ecosystem.config.js`.  

**Note:** Make sure to edit the file to have the correct connectivity information for your mysql and redis servers, and also if you have installed under a user other than 'admin' make sure to update the file paths accordingly (e.g. in the properties below such as 'cwd', 'out_file' etc).

```js
module.exports = {
	apps: [{
			name: 'jambonz-api-server',
			cwd: '/home/admin/apps/jambonz-api-server',
			script: 'app.js',
			out_file: '/home/admin/.pm2/logs/jambonz-api-server.log',
			err_file: '/home/admin/.pm2/logs/jambonz-api-server.log',
			combine_logs: true,
			instance_var: 'INSTANCE_ID',
			exec_mode: 'fork',
			instances: 1,
			autorestart: true,
			watch: false,
			max_memory_restart: '1G',
			env: {
				NODE_ENV: 'production',
				JAMBONES_MYSQL_HOST: '<your-mysql-host>',
				JAMBONES_MYSQL_USER: 'admin',
				JAMBONES_MYSQL_PASSWORD: 'JambonzR0ck$',
				JAMBONES_MYSQL_DATABASE: 'jambones',
				JAMBONES_MYSQL_CONNECTION_LIMIT: 10,
				JAMBONES_REDIS_HOST: '<your-redis-host>',
				JAMBONES_REDIS_PORT: 6379,
				JAMBONES_LOGLEVEL: 'info',
				JAMBONE_API_VERSION: 'v1',
				JAMBONES_CLUSTER_ID: 'jb',
				HTTP_PORT: 3000
			},
		},
		{
			name: 'sbc-call-router',
			cwd: '/home/admin/apps/sbc-call-router',
			script: 'app.js',
			instance_var: 'INSTANCE_ID',
			out_file: '/home/admin/.pm2/logs/jambonz-sbc-call-router.log',
			err_file: '/home/admin/.pm2/logs/jambonz-sbc-call-router.log',
			exec_mode: 'fork',
			instances: 1,
			autorestart: true,
			watch: false,
			max_memory_restart: '1G',
			env: {
				NODE_ENV: 'production',
				HTTP_PORT: 4000,
				JAMBONES_INBOUND_ROUTE: '127.0.0.1:4002',
				JAMBONES_OUTBOUND_ROUTE: '127.0.0.1:4003',
				JAMBONZ_TAGGED_INBOUND: 1,
				JAMBONES_NETWORK_CIDR: '192.168.0.0/16'
			}
		},
		{
			name: 'sbc-registrar',
			cwd: '/home/admin/apps/sbc-registrar',
			script: 'app.js',
			instance_var: 'INSTANCE_ID',
			out_file: '/home/admin/.pm2/logs/jambonz-sbc-registrar.log',
			err_file: '/home/admin/.pm2/logs/jambonz-sbc-registrar.log',
			exec_mode: 'fork',
			instances: 1,
			autorestart: true,
			watch: false,
			max_memory_restart: '1G',
			env: {
				NODE_ENV: 'production',
				JAMBONES_LOGLEVEL: 'info',
				DRACHTIO_HOST: '127.0.0.1',
				DRACHTIO_PORT: 9022,
				DRACHTIO_SECRET: 'cymru',
				JAMBONES_MYSQL_HOST: '<your-mysql-host>',
				JAMBONES_MYSQL_USER: 'admin',
				JAMBONES_MYSQL_PASSWORD: 'JambonzR0ck$',
				JAMBONES_MYSQL_DATABASE: 'jambones',
				JAMBONES_MYSQL_CONNECTION_LIMIT: 10,
				JAMBONES_REDIS_HOST: '<your-redis-host>',
				JAMBONES_REDIS_PORT: 6379,
			}
		},
		{
			name: 'sbc-outbound',
			cwd: '/home/admin/apps/sbc-outbound',
			script: 'app.js',
			instance_var: 'INSTANCE_ID',
			out_file: '/home/admin/.pm2/logs/jambonz-sbc-outbound.log',
			err_file: '/home/admin/.pm2/logs/jambonz-sbc-outbound.log',
			exec_mode: 'fork',
			instances: 1,
			autorestart: true,
			watch: false,
			max_memory_restart: '1G',
			env: {
				NODE_ENV: 'production',
				JAMBONES_LOGLEVEL: 'info',
				DRACHTIO_HOST: '127.0.0.1',
				DRACHTIO_PORT: 9022,
				DRACHTIO_SECRET: 'cymru',
				JAMBONES_RTPENGINES: '127.0.0.1:22222',
				JAMBONES_MYSQL_HOST: '<your-mysql-host>',
				JAMBONES_MYSQL_USER: 'admin',
				JAMBONES_MYSQL_PASSWORD: 'JambonzR0ck$',
				JAMBONES_MYSQL_DATABASE: 'jambones',
				JAMBONES_MYSQL_CONNECTION_LIMIT: 10,
				JAMBONES_REDIS_HOST: '<your-redis-host>',
				JAMBONES_REDIS_PORT: 6379
			}
		},
		{
			name: 'sbc-inbound',
			cwd: '/home/admin/apps/sbc-inbound',
			script: 'app.js',
			instance_var: 'INSTANCE_ID',
			out_file: '/home/admin/.pm2/logs/jambonz-sbc-inbound.log',
			err_file: '/home/admin/.pm2/logs/jambonz-sbc-inbound.log',
			exec_mode: 'fork',
			instances: 1,
			autorestart: true,
			watch: false,
			max_memory_restart: '1G',
			env: {
				NODE_ENV: 'production',
				JAMBONES_LOGLEVEL: 'info',
				DRACHTIO_HOST: '127.0.0.1',
				DRACHTIO_PORT: 9022,
				DRACHTIO_SECRET: 'cymru',
				JAMBONES_RTPENGINES: '127.0.0.1:22222',
				JAMBONES_MYSQL_HOST: '<your-mysql-host>',
				JAMBONES_MYSQL_USER: 'admin',
				JAMBONES_MYSQL_PASSWORD: 'JambonzR0ck$',
				JAMBONES_MYSQL_DATABASE: 'jambones',
				JAMBONES_MYSQL_CONNECTION_LIMIT: 10,
				JAMBONES_REDIS_HOST: '<your-redis-host>',
				JAMBONES_REDIS_PORT: 6379,
				JAMBONES_CLUSTER_ID: 'jb'
			}
		},
		{
			name: 'jambonz-webapp',
			script: 'npm',
			cwd: '/home/admin/apps/jambonz-webapp',
			args: 'run serve'
		}
	]
};
```

Open the following ports on the server

**SBC traffic allowed in**

| ports  | transport | description | 
| ------------- |-------------| -- |
| 3000 |tcp| REST API|
| 3001 |tcp| provisioning GUI|
| 5060 |udp| sip over udp|
| 5060 |tcp| sip over tcp|
| 5061 |tcp| sip over tls|
| 4433 |tcp| sip over wss|
| 40000-60000| udp| rtp |

Next, ssh into the server and run the following command:

```
JAMBONES_MYSQL_HOST=<your-mysql-host> \
JAMBONES_MYSQL_USER=admin \
JAMBONES_MYSQL_PASSWORD=JambonzR0ck$ \
JAMBONES_MYSQL_DATABASE=jambones \
/home/admin/apps/jambonz-api-server/db/reset_admin_password.js"
```
This is a security measure to randomize some of the initial seed data in the mysql database.

Next, start the applications and configure them to restart on boot:

```
sudo -u admin bash -c "pm2 start /home/admin/apps/ecosystem.config.js"
sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u admin --hp /home/admin
sudo -u admin bash -c "pm2 save"
sudo systemctl enable pm2-admin.service
```

Check to be sure they are running:

```
pm2 list
```

You should see output similar to this:
```
admin@ip-172-31-32-10:~$ pm2 list
┌─────┬───────────────────────┬─────────────┬─────────┬─────────┬──────────┬────────┬──────┬───────────┬──────────┬──────────┬──────────┬──────────┐
│ id  │ name                  │ namespace   │ version │ mode    │ pid      │ uptime │ ↺    │ status    │ cpu      │ mem      │ user     │ watching │
├─────┼───────────────────────┼─────────────┼─────────┼─────────┼──────────┼────────┼──────┼───────────┼──────────┼──────────┼──────────┼──────────┤
│ 7   │ jambonz-api-server    │ default     │ 1.1.7   │ fork    │ 4494     │ 4s     │ 0    │ online    │ 30.4%    │ 104.7mb  │ admin    │ disabled │
│ 12  │ jambonz-webapp        │ default     │ N/A     │ fork    │ 4540     │ 4s     │ 0    │ online    │ 7.9%     │ 49.9mb   │ admin    │ disabled │
│ 8   │ sbc-call-router       │ default     │ 0.0.1   │ fork    │ 4500     │ 4s     │ 0    │ online    │ 3.7%     │ 43.8mb   │ admin    │ disabled │
│ 11  │ sbc-inbound           │ default     │ 0.3.5   │ fork    │ 4538     │ 4s     │ 0    │ online    │ 24.1%    │ 100.3mb  │ admin    │ disabled │
│ 10  │ sbc-outbound          │ default     │ 0.4.2   │ fork    │ 4515     │ 4s     │ 0    │ online    │ 13.9%    │ 83.3mb   │ admin    │ disabled │
│ 9   │ sbc-registrar         │ default     │ 0.1.7   │ fork    │ 4512     │ 4s     │ 0    │ online    │ 13.6%    │ 83.0mb   │ admin    │ disabled │
└─────┴───────────────────────┴─────────────┴─────────┴─────────┴──────────┴────────┴──────┴───────────┴──────────┴──────────┴──────────┴──────────┘
Module
┌────┬───────────────────────────────────────┬────────────────────┬───────┬──────────┬──────┬──────────┬──────────┬──────────┐
│ id │ module                                │ version            │ pid   │ status   │ ↺    │ cpu      │ mem      │ user     │
├────┼───────────────────────────────────────┼────────────────────┼───────┼──────────┼──────┼──────────┼──────────┼──────────┤
│ 0  │ pm2-logrotate                         │ 2.7.0              │ 28461 │ online   │ 1    │ 0.3%     │ 80.7mb   │ admin    │
└────┴───────────────────────────────────────┴────────────────────┴───────┴──────────┴──────┴──────────┴──────────┴──────────┘
```

Finally, in your browser, navigate to `http://<sbc-public-ip>:3001`.

You should get a login page to the SBC.  Log in with admin/admin.  You will be asked to change the password and then be guided through an initial 3-step setup process to configuring your account, application, and SIP trunking provider.

### F. Configure Feature Server