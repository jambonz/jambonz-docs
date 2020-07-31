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
    - name: "build_with_grpc"
      prompt: "Include the grpc modules (mod_google_transcribe, mod_google_tts, mod_dialogflow)?"
      private: no
      default: false
    - name: "cloud_provider"
      prompt: "CLoud provider: aws, gcp, azure, digital_ocean"
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


### C. Configure SBC

### D. Configure Feature Server