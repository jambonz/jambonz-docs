# Modifying the webapp to use https

The jambonz webapp is initially configured to run over http on port 3001 of the SBC.  These steps outline the changes needed to run this under https using nginx as the web server and letsencrypt TLS certificates.

### Install NGINX and allow traffic

* SSH into your Jambonz SBC instance
* Install Nginx and certbot
```shell
    $ sudo apt update
    $ sudo apt install nginx python3-certbot-nginx
```
* Navigate to your EC2 Dashboard in the AWS console
* Select your SBC instance
* In the bottom half of the screen, select the Security tab
* Under the Security Groups section select the SG link (allow_jambonz_sbc_sip_rtp) to make changes
* Under the inbound rules tab click the 'Edit inbound rules' button
* Click the 'Add rule' button to add the following rules:
```shell
    type: HTTP - source: Anywhere
    type: HTTPS - source: Anywhere
```

### Create DNS entries

* Navigate to Route 53, or wherever you have your domains hosted
* Add your desired domains and sub-domains (A records) and point them to the ip address of your SBC instance

	you will need to add a sub-domain for the api and the app, eg: 
```
  api.your-domain.com
  app.your-domain.com
```	

### Configure NGINX

* Return to your ssh session and navigate to your nginx sites-available folder
```shell
    $ cd /etc/nginx/sites-available
```
* Make a copy of the default config
```shell
    $ sudo cp default default.bak
```
* Edit the default config to look like below:
```shell
    $ sudo nano default

    server {
        listen 80;
        server_name your_domain.com;  # enter the app sub-domain that you setup in 11
        location / {
            proxy_pass http://localhost:3001; # point the reverse proxy to the webapp on port 3001
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }
    }
```
* Save your new configuration
* Create another config in your sites-available folder that you are currently in:
```shell
    $ sudo nano jambonz-api

    server {
        listen 80;
        server_name api.your_domain.com;  # enter the app sub-domain that you setup in 11
        location / {
            proxy_pass http://localhost:3000; # point the reverse proxy to the api server on port 3000
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }
    }
```
* Create a link from the sites-enabled directory to the new configuration
```shell
    $ sudo ln -s /etc/nginx/sites-available/jambonz-api /etc/nginx/sites-enabled/jambonz-api 
```

* Restart the nginx service
```shell
    $ sudo systemctl restart nginx
```

### Generate TLS certificates

* Install LetsEncrypt certificate using certbot
```shell
    $ certbot --nginx
```
* Follow the prompts to create certificates for both the 'app' and the 'api' domains
* At the end of the script choose option 2 to make all requests redirect to HTTPS
* Return to the security group settings as discussed above
* Change access to port 3000 and 3001 from everywhere to SG

### Modify webapp and restart

* Head back to your ssh terminal
* Change ownership of all the files in the app to admin
```
	$ cd ~/app
	$ sudo chown -R admin:admin . 
```
* Edit the jambonz webapp env
```
	$ cd ~/app/jambonz-webapp
	$ nano .env.local
	change http://ip:3000/v1 to
	https://api.your-domain.com/v1
```
> Note: make sure to change the api URL above from http to https.  If you leave it http requests will fail since the browser will prevent a page retrieved over https from making unsecured http calls.

* Rebuild the app
```
	$ cd /home/admin/apps/jambonz-webapp && sudo npm install --unsafe-perm && npm run build
```
* Restart the services using pm2
```
	$ pm2 restart all
```

Now navigate in your browser to https://app.yourdomain.com and test that your new domain works and is certified.

