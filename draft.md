# Docker Local Repository

## Using `Dockerfile`

### Build and run

<p class="callout danger">Containers created with this method will be removed after exit.  
To keep container use commands below instead:  
`docker build -t IMAGE_NAME .docker run -it IMAGE_NAME`  
</p>

```
docker run --rm -it $(docker build -q .)
```

## Using `docker-compose.yml`

# Docker Volumes

## Inspect mapping

To inspect mapping between a *host directory* and a *volume*, first get the container's *id*:

```shell
docker ps
```

then run the `inspect` command like below:

```shell
docker inspect -f "{{ .Mounts }}" CONTAINER_ID
```

# Docker Compose

> Compose is a tool for defining and running multi-container Docker applications. With Compose, you use a YAML file to configure your application’s services. Then, with a single command, you create and start all the services from your configuration. To learn more about all the features of Compose, see [the list of features](https://docs.docker.com/compose/#features).
> 
> `docker compose up -d` builds, (re)creates, starts, and attaches to containers for a service. - [ref](https://docs.docker.com/engine/reference/commandline/compose_up "Link to official Docker documentation of 'docker compose up'")

# Docker Images

## Save Images Locally

### Save images in individual tar files (not tested)

```shell
docker images \
| grep -v '<none>' \
| awk '{if ($1 ~ /^(openshift|centos)/) print $1 " " $2 " " $3 }' \
| tr -c "a-z A-Z0-9_.\n-" "%" \
| while read REPOSITORY TAG IMAGE_ID
do
  echo "== Saving $REPOSITORY $TAG $IMAGE_ID =="
  docker save -o ./images/$REPOSITORY-$TAG-$IMAGE_ID.tar $IMAGE_ID
done
```

### Save all images in one file

```shell
# docker save $(docker images | sed '1d' | grep -v "<none>" | awk '{print $1 ":" $2 }') -o $(date).tar
docker save $(docker images -q) -o "$(date).tar"
```

## Load Images

<span style="background-color: #e03e2d;">todo: docker load</span>

# Docker API and Self-signed signatures

Enabling the API

Securing the API

# Docker TLS handshake error

## When `docker run` command returns errors:

```
Unable to find image '....' locally
docker: Error response from daemon: Get "https://registry-1.docker.io/v2/.../.../manifests/sha256:643...c9a0": net/http: TLS handshake timeout.
See 'docker run --help'.
```

## Try: Bypass proxy

```shell
unset http_proxy
unset https_proxy
```

# Docker Resource Limitations

```YAML
    deploy:
      resources:
        limits:
          memory: 64M
          cpus: '0.05'
```

# Docker SSH & API

> <article><div class="l"><div class="l"><section> You can execute any docker command through this API. You can read the [documentation](https://docs.docker.com/engine/api/v1.40/) to get more info about the available API methods. In this story we’ll learn how to use the [Docker Engine API](https://docs.docker.com/engine/api/v1.40/) through the network in a secure way. The Engine API is an HTTP API served by the Docker Engine. It’s the API the Docker client uses to communicate with the Engine, so everything the Docker client can do can also be done with the API. \[[ref](https://medium.com/trabe/using-docker-engine-api-securely-584e0882158e)\]
> 
> </section></div></div></article>

## Enabling the API

> In order to use the Docker Engine API, a TCP socket must be enabled when the engine daemon starts. By default, a `unix` domain socket (or IPC socket) is created at `/var/run/docker.sock`. However, we can configure the daemon to listen to multiple sockets at the same time using multiple `-H` options

### Method 1 (Temporary use)

```shell
dockerd -H unix:///var/run/docker.sock -H tcp://0.0.0.0:2375
```

### Method 2 (Permanent)

edit `/etc/systemd/system/docker.service.d/override.conf` and add the third line as below:

```shell
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H fd:// -H tcp://0.0.0.0:2375
```

## Reload deamon and service to enable new configuration

```shell
sudo systemctl daemon-reload
sudo systemctl restart docker.service
```

## Test API

```shell
curl -X GET http://localhost:2375/containers/json?all=1
```

## Securing the API (Self-signed certificate)

### CA Certificate

<p class="callout info">Certificate Authority is placed on the <span style="text-decoration: underline;">host machine</span>.</p>

```shell
openssl genrsa -aes256 -out ca-key.pem 4096
```

### CA File

<p class="callout info">Certificate Authority is placed on the <span style="text-decoration: underline;">host machine</span>.</p>

```shell
openssl req -new -x509 -days 365 -key ca-key.pem -sha256 \
-out ca.pem
```

### Server Key

<p class="callout info">Server Key is placed on the <span style="text-decoration: underline;">host machine</span>.</p>

<p class="callout warning">Keep this key a secret.</p>

```
openssl genrsa -out server-key.pem 4096
```

### Certificate Signing Request (CSR)

```
openssl req -subj "/CN=docker.tyyi.net" -sha256 -new \
-key server-key.pem -out server.csr
```

### Sign the CSR with CA (If not connecting with IP)

```
openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out server-cert.pem
```

### Sign the CSR with CA (If connecting with IP)

<p class="callout info">Replace *MY\_SERVER\_IP\_ADDRESS\_HERE* with the server's IP address.</p>

```
echo subjectAltName = DNS:docker.tyyi.net,IP:MY_SERVER_IP_ADDRESS_HERE,IP:10.0.0.100,IP:127.0.0.1 >> extfile.cnf
echo extendedKeyUsage = serverAuth >> extfile.cnf
openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out server-cert.pem -extfile extfile.cnf
```

### Enable TLS

edit `/etc/systemd/system/docker.service.d/override.conf` and update the third line as below:

```shell
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H fd:// -H tcp://0.0.0.0:2376 --tlsverify --tlscacert=/etc/ssl/certs/ca.pem --tlscert=/etc/ssl/certs/server-cert.pem --tlskey=/etc/ssl/private/server-key.pem
```

<p class="callout danger">Deamon must be reloaded as described before.</p>

### Generate Client Certificate

<p class="callout info">This file is placed on the client machine.</p>

```
openssl genrsa -out key.pem 4096
openssl req -subj '/CN=client' -new -key key.pem -out client.csr
echo extendedKeyUsage = clientAuth > extfile-client.cnf
openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out cert.pem -extfile extfile-client.cnf
```

## Connect to portainer

[![image-1669191749324.png](https://library.tyyi.net/uploads/images/gallery/2022-11/scaled-1680-/image-1669191749324.png)](https://library.tyyi.net/uploads/images/gallery/2022-11/image-1669191749324.png)

## Test Secured API

> For example, you may want to consume the API using Python and the [requests library](https://2.python-requests.org/en/master/). In the next snippet we’ll make a request to fetch the list of containers available in the docker host. We must include the certificates in the library calls:

```Python
import requests

CA_CERT = '/Users/martin/certs/ca.pem'

CLIENT_CERT = '/Users/martin/certs/cert.pem'
CLIENT_KEY = '/Users/martin/certs/key.pem'

URL = 'https://docker.acme.com:2376/containers/json?all=1'


def get_container_list():
    response = requests.get(url=URL, cert=(CLIENT_CERT, CLIENT_KEY),
                            verify=CA_CERT)
    return response.json()


def print_containers_data():
    for container in get_container_list():
        print('Container [Id: %s, Name: %s, Status: %s]' %
              (container['Id'], container['Names'][0], container['Status']))
        
        
print_containers_data()
```

### Data Center Firewall

Please note that you have to allow API's port on the data center's firewall.

[![image-1669191817869.png](https://library.tyyi.net/uploads/images/gallery/2022-11/scaled-1680-/image-1669191817869.png)](https://library.tyyi.net/uploads/images/gallery/2022-11/image-1669191817869.png)

# Docker Container Shell

To get `bash`/`sh` access from a container:

```
docker exec -it CONTAINER_NAME_HERE sh
```

# Changes & Version Control



# Git Installation

To install git simply command:

```
sudo apt install git
```

# Git Security

## Remember passwords

To store passwords in git client:

```shell
git config --global credential.helper store
git pull
```

# Git submodules

## What is submodule?

> It often happens that while working on one project, you need to use another project from within it. Perhaps it’s a library that a third party developed or that you’re developing separately and using in multiple parent projects. A common issue arises in these scenarios: you want to be able to treat the two projects as separate yet still be able to use one from within the other. [src](https://git-scm.com/book/en/v2/Git-Tools-Submodules)

## Submodules vs Subtrees

> - Git submodules have a smaller repository size since they are just links to a single commit in a subproject; whereas Git subtrees store the whole subproject, including its history.
> - Subtrees are decentralized, while Git submodules must be accessible on the server. [src](https://gitprotect.io/blog/managing-git-projects-git-subtree-vs-submodule)

## Add submodule

```shell
git submodule add https://git.tyyi.net/myproject/textyy app/assets/plugin/textyy
```

## Clone/Pull a project with it's submodules

```shell
git clone --recurse-submodules --remote-submodules https://git.tyyi.net/myproject/textyy
```

## Cloning a project's submodules

```shell
git submodule init
git submodule update
```

# File mode

## To ignore mode changes

```shell
git config core.fileMode false
```

# Cronjobs



# Crontab

## Edit Cronjobs for current user

```shell
crontab -e
```

# Cron.d

The main difference between `crontab -e` and directly editing `/etc/crond.d/<username>` in syntax is the necessity of adding related username to definition of the command. etc:

```C
*/30 * * * * ubuntu ./test.sh
```

# Automatically sync git

## sync\_git.sh

```shell
echo 'syncing myproject.api'
cd d
cd myproject.api
$(git stash)
$(git pull)
echo $(git status)
echo $(docker container restart myproject-api)

echo 'syncing myproject.front'
cd ../
cd myproject.front
$(git stash)
$(git pull)
echo $(git status)
```

## /etc/cron.d/root

<p class="callout info">Root username is `root` and current user is `ubuntu`</p>

<p class="callout info">`wall` command will broadcast output to all active users in bash</p>

```C
*/30 * * * * ubuntu wall $(~/sync_git.sh)
```

It's possible to use *crontab-ui* even [in docker](http://beta.tyyi.net:8080/books/devops/page/docker-compose "Docker compose for crontab-ui") to manage this. But please follow updates on [this issue](https://github.com/alseambusher/crontab-ui/issues/208 "dockerized crontab-ui jobs will not run without trick #208") on Github.

# Schedule Automatic Shutdown

Edit the `/etc/rc.local` as below:

```shell
#!/bin/bash

# schedule turnoff
sudo shutdown -P 12:58 "The system is going DOWN to maintenance mode at 12:58 GMT!"

# NECESSARY TO EXIT WITH 0
exit 0
```

# SSH



# Connect to remote server without entering password

```shell
ssh-keygen -t rsa -b 2048
ssh-copy-id <server>
ssh <server>
```

# Monitoring



# Up-time Monitoring

## Third-party services

- [uptimerobot.com](https://uptimerobot.com)
- 

# Speed test

You can use the command below to test the net speed using a third-party web server

```shell
wget --output-document=/dev/null http://speedtest.wdc01.softlayer.com/downloads/test500.zip
```

# Docker Monitoring

To get a report of resources used by containers run

```shell
docker stats --no-stream
```

# Server Softwares



# Portainer

## Installation

<details id="bkmrk-docker-compose.ymlve"><summary>docker-compose.yml</summary>

```YAML
version: "3"
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./data:/data
    ports:
      - 9000:9000
```

</details>## Reset Password

[\[docs.portainer.io\]](https://docs.portainer.io/v/ce-2.9/advanced/reset-admin) / [\[github.com\]](https://github.com/portainer/helper-reset-password)

```shell
# stop the existing Portainer container
docker container stop portainer

# run the helper using the same bind-mount/volume for the data volume
docker run --rm -v portainer_data:/data portainer/helper-reset-password
2020/06/04 00:13:58 Password succesfully updated for user: admin
2020/06/04 00:13:58 Use the following password to login: &_4#\3^5V8vLTd)E"NWiJBs26G*9HPl1

# restart portainer and use the password above to login
docker container start portainer
```

# MySQL

```YAML
version: '3.3'
services:
  db:
    image: mysql:5.7
    container_name: 'mysql_5.7'
    hostname: 'mysql'
    restart: always
    environment:
      MYSQL_DATABASE: 'db'
      # So you don't have to use root, but you can if you like
      # MYSQL_USER: 'root'
      # You can use whatever password you like
      MYSQL_PASSWORD: '123'
      # Password for root access
      MYSQL_ROOT_PASSWORD: '123'
    ports:
      #  : < MySQL Port running inside container>
      - '3306:3306'
    expose:
      # Opens port 3306 on the container
      - '3306'
      # Where our data will be persisted
    volumes:
      - ./data:/var/lib/mysql
# Names our volume
# volumes:
#  my-db:

```

# PhpMyAdmin

```YAML
version: "2"
services:
    phpmyadmin:
        image: phpmyadmin/phpmyadmin
        container_name: 'phpmyadmin'
        restart: unless-stopped
        environment:
            PMA_HOST: mysql
            PMA_PORT: 3306
#        depends_on:
#            - mysql
        ports:
            - "83:80"
        networks:
            - mysql_default
networks:
     mysql_default:
            external: true

```

# BookStack

```YAML
version: '2'
services:
  bookstack_mysql:
    image: mysql:8.0
    container_name: 'bookstack_mysql'
    restart: always
    environment:
    - MYSQL_ROOT_PASSWORD=secret
    - MYSQL_DATABASE=bookstack
    - MYSQL_USER=bookstack
    - MYSQL_PASSWORD=secret
    volumes:
    - ./data/mysql:/var/lib/mysql

  bookstack:
    image: solidnerd/bookstack:22.04.02
    restart: always
    container_name: 'bookstack'
    depends_on:
    - bookstack_mysql
    environment:
    - DB_HOST=bookstack_mysql:3306
    - DB_DATABASE=bookstack
    - DB_USERNAME=bookstack
    - DB_PASSWORD=secret
    - APP_URL=
    volumes:
    - ./data/uploads:/var/www/bookstack/public/uploads
    - ./data/storage:/var/www/bookstack/storage/uploads
    ports:
    - 82:8080
```

# MongoDB

```YAML

version: '3'
services:
  database:
    image: mongo:4.4.6
    container_name: 'mongo'
    restart: always
    environment:
      - MONGO_INITDB_DATABASE=bot
      - MONGO_INITDB_ROOT_USERNAME=root
      - MONGO_INITDB_ROOT_PASSWORD=123
    volumes:
#      - ./init-mongo.js:/docker-entrypoint-initdb.d/mongo-init.js:ro
      - ./data:/data/db
    ports:
      - '27017-27019:27017-27019'

```

# Kanboard

```YAML
version: '2'
services:
  kanboard:
    container_name: kanboard
    restart: always
    image: kanboard/kanboard:latest
    ports:
     - "85:80"
#     - "443:443"
    volumes:
     - ./data/db:/var/www/app/data
     - ./data/plugins:/var/www/app/plugins
     - ./data/ssl:/etc/nginx/ssl
volumes:
  kanboard_data:
    driver: local
  kanboard_plugins:
    driver: local
  kanboard_ssl:
    driver: local

```

# ProcessMaker

```YAML
version: "2.1"

services:
   processmaker_mysql:
     container_name: 'processmaker_mysql'
     image: mysql:5.6
     restart: always
     environment:
       MYSQL_ROOT_PASSWORD: 123
       MYSQL_USER: pm
       MYSQL_DATABASE: pm
       MYSQL_PASSWORD: 123
     ports:
       - 3309:3306
     volumes:
       - ./data/mysql:/var/lib/mysql
     networks:
       - bpm

   processmaker:
     container_name: 'processmaker'
     depends_on:
       - processmaker_mysql
     image: eltercera/docker-processmaker
     volumes:
       - ./data/processmaker:/opt/processmaker
     ports:
       - "86:80"
       - "87:8080"
     restart: always
     environment:
       URL: "127.0.0.1"
     networks:
      - bpm

networks:
  bpm:
    driver: bridge
    ipam:
      driver: default
      config:
      -
        subnet: 172.16.150.0/24
```

# Gitea

```YAML
version: "3"

networks:
  gitea:
    external: true

services:
  server:
    image: gitea/gitea:1.16.8
    container_name: gitea
    environment:
      - USER_UID=1000
      - USER_GID=1000
    restart: always
    volumes:
      - ./data:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "81:3000"
      - "23:22"
networks:
    default:
      driver: bridge
```

# Virola

<details id="bkmrk-download_and_run.shw"><summary>download\_and\_run.sh</summary>

```shell
wget https://virola.io/downloads/1.0.18.22091906/virola-server-docker-1.0.18.22091906.tar.gz;
docker load < virola-server-docker-1.0.18.22091906.tar.gz
```

</details><details id="bkmrk-run.shcurrent_dir%3D%24%28"><summary>run.sh</summary>

```shell
CURRENT_DIR=$( cd -- "$( dirname -- "${BASH_SOURCE[0]}" )" &> /dev/null && pwd )
docker run \
	--name virola \
	--volume=$CURRENT_DIR/data:/virola \
	--restart=always \
	-d \
	-p 8888:7777/tcp \
	-p 8888:7777/udp \
	-it providesupport/virola-server:latest

```

</details>

# Chatwoot

<details id="bkmrk-.env%23-used-to-verify"><summary>.env</summary>

```INI
# Used to verify the integrity of signed cookies. so ensure a secure value is set
SECRET_KEY_BASE=replace_with_lengthy_secure_hex

# Replace with the URL you are planning to use for your app
FRONTEND_URL=https://0.0.0.0:3000
# To use a dedicated URL for help center pages
# HELPCENTER_URL=https://0.0.0.0:3333

# If the variable is set, all non-authenticated pages would fallback to the default locale.
# Whenever a new account is created, the default language will be DEFAULT_LOCALE instead of en
# DEFAULT_LOCALE=en

# If you plan to use CDN for your assets, set Asset CDN Host
ASSET_CDN_HOST=

# Force all access to the app over SSL, default is set to false
FORCE_SSL=false

# This lets you control new sign ups on your chatwoot installation
# true : default option, allows sign ups
# false : disables all the end points related to sign ups
# api_only: disables the UI for signup, but you can create sign ups via the account apis
ENABLE_ACCOUNT_SIGNUP=false

# Redis config
REDIS_URL=redis://redis:6379
# If you are using docker-compose, set this variable's value to be any string,
# which will be the password for the redis service running inside the docker-compose
# to make it secure
REDIS_PASSWORD=123
# Redis Sentinel can be used by passing list of sentinel host and ports e,g. sentinel_host1:port1,sentinel_host2:port2
REDIS_SENTINELS=
# Redis sentinel master name is required when using sentinel, default value is "mymaster".
# You can find list of master using "SENTINEL masters" command
REDIS_SENTINEL_MASTER_NAME=

# Redis premium breakage in heroku fix
# enable the following configuration
# ref: https://github.com/chatwoot/chatwoot/issues/2420
# REDIS_OPENSSL_VERIFY_MODE=none

# Postgres Database config variables
# You can leave POSTGRES_DATABASE blank. The default name of 
# the database in the production environment is chatwoot_production
# POSTGRES_DATABASE=
POSTGRES_HOST=postgres
POSTGRES_USERNAME=postgres
POSTGRES_PASSWORD=123
RAILS_ENV=development
RAILS_MAX_THREADS=5

# The email from which all outgoing emails are sent
# could user either  `email@yourdomain.com` or `BrandName <email@yourdomain.com>`
MAILER_SENDER_EMAIL="Support <support@tyyi.net>"

#SMTP domain key is set up for HELO checking
SMTP_DOMAIN=chatwoot.com
# the default value is set "mailhog" and is used by docker-compose for development environments,
# Set the value as "localhost" or your SMTP address in other environments
SMTP_ADDRESS=mailhog
SMTP_PORT=1025
SMTP_USERNAME=
SMTP_PASSWORD=
# plain,login,cram_md5
SMTP_AUTHENTICATION=
SMTP_ENABLE_STARTTLS_AUTO=true
# Can be: 'none', 'peer', 'client_once', 'fail_if_no_peer_cert', see http://api.rubyonrails.org/classes/ActionMailer/Base.html
SMTP_OPENSSL_VERIFY_MODE=peer
# Comment out the following environment variables if required by your SMTP server
# SMTP_TLS=
# SMTP_SSL=

# Mail Incoming
# This is the domain set for the reply emails when conversation continuity is enabled
MAILER_INBOUND_EMAIL_DOMAIN=
# Set this to appropriate ingress channel with regards to incoming emails
# Possible values are :
# relay for Exim, Postfix, Qmail
# mailgun for Mailgun
# mandrill for Mandrill
# postmark for Postmark
# sendgrid for Sendgrid
RAILS_INBOUND_EMAIL_SERVICE=
# Use one of the following based on the email ingress service
# Ref: https://edgeguides.rubyonrails.org/action_mailbox_basics.html
RAILS_INBOUND_EMAIL_PASSWORD=
MAILGUN_INGRESS_SIGNING_KEY=
MANDRILL_INGRESS_API_KEY=

# Storage
ACTIVE_STORAGE_SERVICE=local

# Amazon S3
# documentation: https://www.chatwoot.com/docs/configuring-s3-bucket-as-cloud-storage
S3_BUCKET_NAME=
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=

# Log settings
# Disable if you want to write logs to a file
RAILS_LOG_TO_STDOUT=true
LOG_LEVEL=info
LOG_SIZE=500

### This environment variables are only required if you are setting up social media channels

# Facebook
# documentation: https://www.chatwoot.com/docs/facebook-setup
FB_VERIFY_TOKEN=
FB_APP_SECRET=
FB_APP_ID=

# https://developers.facebook.com/docs/messenger-platform/instagram/get-started#app-dashboard
IG_VERIFY_TOKEN=

# Twitter
# documentation: https://www.chatwoot.com/docs/twitter-app-setup
TWITTER_APP_ID=
TWITTER_CONSUMER_KEY=
TWITTER_CONSUMER_SECRET=
TWITTER_ENVIRONMENT=

#slack integration
SLACK_CLIENT_ID=
SLACK_CLIENT_SECRET=

### Change this env variable only if you are using a custom build mobile app
## Mobile app env variables
IOS_APP_ID=L7YLMN4634.com.chatwoot.app
ANDROID_BUNDLE_ID=com.chatwoot.app

# https://developers.google.com/android/guides/client-auth (use keytool to print the fingerprint in the first section)
ANDROID_SHA256_CERT_FINGERPRINT=AC:73:8E:DE:EB:56:EA:CC:10:87:02:A7:65:37:7B:38:D4:5D:D4:53:F8:3B:FB:D3:C6:28:64:1D:AA:08:1E:D8

### Smart App Banner
# https://developer.apple.com/library/archive/documentation/AppleApplications/Reference/SafariWebContent/PromotingAppswithAppBanners/PromotingAppswithAppBanners.html
# You can find your app-id in https://itunesconnect.apple.com
#IOS_APP_IDENTIFIER=1495796682

## Push Notification
## generate a new key value here : https://d3v.one/vapid-key-generator/
# VAPID_PUBLIC_KEY=
# VAPID_PRIVATE_KEY=
#
# for mobile apps
# FCM_SERVER_KEY=

## Bot Customizations
USE_INBOX_AVATAR_FOR_BOT=true

### APM and Error Monitoring configurations
## Elastic APM
## https://www.elastic.co/guide/en/apm/agent/ruby/current/getting-started-rails.html
# ELASTIC_APM_SERVER_URL=
# ELASTIC_APM_SECRET_TOKEN=

## Sentry
# SENTRY_DSN=

## Scout
## https://scoutapm.com/docs/ruby/configuration
# SCOUT_KEY=YOURKEY
# SCOUT_NAME=YOURAPPNAME (Production)
# SCOUT_MONITOR=true

## NewRelic
# https://docs.newrelic.com/docs/agents/ruby-agent/configuration/ruby-agent-configuration/
# NEW_RELIC_LICENSE_KEY=
# Set this to true to allow newrelic apm to send logs.
# This is turned off by default.
# NEW_RELIC_APPLICATION_LOGGING_ENABLED=

## Datadog
## https://github.com/DataDog/dd-trace-rb/blob/master/docs/GettingStarted.md#environment-variables
# DD_TRACE_AGENT_URL=

## IP look up configuration
## ref https://github.com/alexreisner/geocoder/blob/master/README_API_GUIDE.md
## works only on accounts with ip look up feature enabled
# IP_LOOKUP_SERVICE=geoip2
# maxmindb api key to use geoip2 service
# IP_LOOKUP_API_KEY=

## Rack Attack configuration
## To prevent and throttle abusive requests
# ENABLE_RACK_ATTACK=true

## Running chatwoot as an API only server
## setting this value to true will disable the frontend dashboard endpoints
# CW_API_ONLY_SERVER=false

## Development Only Config
# if you want to use letter_opener for local emails
# LETTER_OPENER=true
# meant to be used in github codespaces
# WEBPACKER_DEV_SERVER_PUBLIC=

# If you want to use official mobile app,
# the notifications would be relayed via a Chatwoot server
ENABLE_PUSH_RELAY_SERVER=true

# Stripe API key
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=

# Set to true if you want to upload files to cloud storage using the signed url
# Make sure to follow https://edgeguides.rubyonrails.org/active_storage_overview.html#cross-origin-resource-sharing-cors-configuration on the cloud storage after setting this to true.
DIRECT_UPLOADS_ENABLED=

```

</details><p class="callout warning">Prepare the database by running the migrations: `docker compose run --rm rails bundle exec rails db:chatwoot_prepare`</p>

<details id="bkmrk-docker-compose.ymlve"><summary>docker-compose.yml</summary>

```YAML
version: '3'

services:
  base: &base
    image: chatwoot/chatwoot:latest
    env_file: .env ## Change this file for customized env variables
    volumes:
      - /data/storage:/app/storage

  rails:
    <<: *base
    depends_on:
      - postgres
      - redis
    ports:
      - '0.0.0.0:4444:3000'
#      - '0.0.0.0:3333:3333'
    environment:
      - NODE_ENV=production
      - RAILS_ENV=production
      - INSTALLATION_ENV=docker
    entrypoint: docker/entrypoints/rails.sh
    command: ['bundle', 'exec', 'rails', 's', '-p', '3000', '-b', '0.0.0.0']

  sidekiq:
    <<: *base
    depends_on:
      - postgres
      - redis
    environment:
      - NODE_ENV=production
      - RAILS_ENV=production
      - INSTALLATION_ENV=docker
    command: ['bundle', 'exec', 'sidekiq', '-C', 'config/sidekiq.yml']

  postgres:
    hostname: postgres
    image: postgres:12
    restart: always
    ports:
      - '127.0.0.1:5432:5432'
    volumes:
      - ./data/postgres:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=chatwoot
      - POSTGRES_USER=postgres
      # Please provide your own password.
      - POSTGRES_PASSWORD=123

  redis:
    hostname: redis
    image: redis:alpine
    restart: always
    command: ["sh", "-c", "redis-server --requirepass \"$REDIS_PASSWORD\""]
    env_file: .env
    volumes:
      - ./data/redis:/data
    ports:
      - '127.0.0.1:6379:6379'

```

</details>

# crontab-ui

<details id="bkmrk-variables.envbasic_a"><summary>variables.env</summary>

```INI
BASIC_AUTH_USER=tayyebi
BASIC_AUTH_PWD=123
```

</details><details id="bkmrk-dockerfile%23-docker-r"><summary>Dockerfile</summary>

```Powershell
# docker run -d -p 8000:8000 alseambusher/crontab-ui

FROM alpine:3.15.3

ENV   CRON_PATH /etc/crontabs

RUN   mkdir /crontab-ui; touch $CRON_PATH/root; chmod +x $CRON_PATH/root

WORKDIR /crontab-ui

LABEL maintainer "@alseambusher"
LABEL description "Crontab-UI docker"

RUN   apk --no-cache add \
      wget \
      curl \
      nodejs \
      npm \
      supervisor \
      tzdata

COPY supervisord.conf /etc/supervisord.conf
COPY . /crontab-ui

RUN   npm install

ENV   HOST 0.0.0.0

ENV   PORT 8000

ENV   CRON_IN_DOCKER true

EXPOSE $PORT

CMD ["supervisord", "-c", "/etc/supervisord.conf"]

```

</details><details id="bkmrk-docker-compose.ymlve"><summary>docker-compose.yml</summary>

```YAML
version: '3.7'

services:
  crontab-ui:
    container_name: crontab-ui
    build: .
    image: alseambusher/crontab-ui
    network_mode: bridge
    ports:
      - 3030:8000
    env_file: variables.env
    volumes:
      # - /var/spool/cron/crontabs/ubuntu:/etc/crontabs/root
      - /etc/cron.d:/etc/crontabs
      - ~/d:/d
      - ./data/crontabs:/crontab-ui/crontabs
      - /tmp/crontab-ui:/tmp
    working_dir: /d
```

</details>

# Mailu

<details id="bkmrk-%23-this-file-is-auto-"><summary>docker-compose.yml</summary>

```YAML
# This file is auto-generated by the Mailu configuration wizard.
# Please read the documentation before attempting any change.
# Generated for compose flavor

version: '2.2'

services:

  # External dependencies
  redis:
    image: redis:alpine
    restart: always
    volumes:
      - "./data/redis:/data"
    depends_on:
      - resolver
    networks:
      - mail
    dns:
      - 10.100.0.10

  # Core services
  front:
    image: ${DOCKER_ORG:-mailu}/${DOCKER_PREFIX:-}nginx:${MAILU_VERSION:-1.9}
    restart: always
    env_file: mailu.env
    logging:
      driver: json-file
    ports:
      - "0.0.0.0:80:80"
      - "0.0.0.0:443:443"
      - "0.0.0.0:25:25"
      - "0.0.0.0:465:465"
      - "0.0.0.0:587:587"
      - "0.0.0.0:110:110"
      - "0.0.0.0:995:995"
      - "0.0.0.0:143:143"
      - "0.0.0.0:993:993"
    volumes:
      - "./data/certs:/certs"
      - "./data/overrides/nginx:/overrides:ro"
    depends_on:
      - resolver
    networks:
      - mail
    dns:
      - 10.100.0.10

  resolver:
    image: ${DOCKER_ORG:-mailu}/${DOCKER_PREFIX:-}unbound:${MAILU_VERSION:-1.9}
    env_file: mailu.env
    restart: always
    networks:
      mail:
        ipv4_address: 10.100.0.10

  admin:
    image: ${DOCKER_ORG:-mailu}/${DOCKER_PREFIX:-}admin:${MAILU_VERSION:-1.9}
    restart: always
    env_file: mailu.env
    volumes:
      - "./data/data:/data"
      - "./data/dkim:/dkim"
    depends_on:
      - redis
      - resolver
    networks:
      - mail
    dns:
      - 10.100.0.10

  imap:
    image: ${DOCKER_ORG:-mailu}/${DOCKER_PREFIX:-}dovecot:${MAILU_VERSION:-1.9}
    restart: always
    env_file: mailu.env
    volumes:
      - "./data/mail:/mail"
      - "./data/overrides/dovecot:/overrides:ro"
    depends_on:
      - front
      - resolver
    networks:
      - mail
    dns:
      - 10.100.0.10

  smtp:
    image: ${DOCKER_ORG:-mailu}/${DOCKER_PREFIX:-}postfix:${MAILU_VERSION:-1.9}
    restart: always
    env_file: mailu.env
    volumes:
      - "./data/mailqueue:/queue"
      - "./data/overrides/postfix:/overrides:ro"
    depends_on:
      - front
      - resolver
    networks:
      - mail
    dns:
      - 10.100.0.10

  antispam:
    image: ${DOCKER_ORG:-mailu}/${DOCKER_PREFIX:-}rspamd:${MAILU_VERSION:-1.9}
    hostname: antispam
    restart: always
    env_file: mailu.env
    volumes:
      - "./data/filter:/var/lib/rspamd"
      - "./data/overrides/rspamd:/etc/rspamd/override.d:ro"
    depends_on:
      - front
      - resolver
    networks:
      - mail
    dns:
      - 10.100.0.10

  # Optional services

  webdav:
    image: ${DOCKER_ORG:-mailu}/${DOCKER_PREFIX:-}radicale:${MAILU_VERSION:-1.9}
    restart: always
    env_file: mailu.env
    volumes:
      - "./data/dav:/data"
    depends_on:
      - resolver
    networks:
      - mail
    dns:
      - 10.100.0.10

  fetchmail:
    image: ${DOCKER_ORG:-mailu}/${DOCKER_PREFIX:-}fetchmail:${MAILU_VERSION:-1.9}
    restart: always
    env_file: mailu.env
    volumes:
      - "./data/data/fetchmail:/data"
    depends_on:
      - resolver
    networks:
      - mail
    dns:
      - 10.100.0.10

  # Webmail
  webmail:
    image: ${DOCKER_ORG:-mailu}/${DOCKER_PREFIX:-}roundcube:${MAILU_VERSION:-1.9}
    restart: always
    env_file: mailu.env
    volumes:
      - "./data/webmail:/data"
      - "./data/overrides/roundcube:/overrides:ro"
    depends_on:
      - imap
      - resolver
    networks:
      - mail
    dns:
      - 10.100.0.10

networks:
  mail:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 10.100.0.0/24

```

</details><p class="callout info">This file is generated from wizard available on mailu website [\[link\]](https://setup.mailu.io/1.9/setup/c1a09085-2df5-433f-ae3f-f5bd15a0432a "Mailu Setup").</p>

<details id="bkmrk-%23-mailu-main-configu"><summary>mailu.env</summary>

```YAML
# Mailu main configuration file
#
# This file is autogenerated by the configuration management wizard for compose flavor.
# For a detailed list of configuration variables, see the documentation at
# https://mailu.io

###################################
# Common configuration variables
###################################

# Set to a randomly generated 16 bytes string
SECRET_KEY=44IMD0FACJD9S8D8

# Subnet of the docker network. This should not conflict with any networks to which your system is connected. (Internal and external!)
SUBNET=10.100.0.0/24

# Main mail domain
DOMAIN=tyyi.net

# Hostnames for this server, separated with comas
HOSTNAMES=mail.tyyi.net,mail.tyyi.net

# Postmaster local part (will append the main mail domain)
POSTMASTER=admin

# Choose how secure connections will behave (value: letsencrypt, cert, notls, mail, mail-letsencrypt)
TLS_FLAVOR=letsencrypt

# Authentication rate limit per IP (per /24 on ipv4 and /56 on ipv6)
AUTH_RATELIMIT_IP=60/hour

# Authentication rate limit per user (regardless of the source-IP)
AUTH_RATELIMIT_USER=100/day

# Opt-out of statistics, replace with "True" to opt out
DISABLE_STATISTICS=False

###################################
# Optional features
###################################

# Expose the admin interface (value: true, false)
ADMIN=true

# Choose which webmail to run if any (values: roundcube, rainloop, none)
WEBMAIL=roundcube

# Dav server implementation (value: radicale, none)
WEBDAV=radicale

# Antivirus solution (value: clamav, none)
ANTIVIRUS=none

###################################
# Mail settings
###################################

# Message size limit in bytes
# Default: accept messages up to 50MB
# Max attachment size will be 33% smaller
MESSAGE_SIZE_LIMIT=50000000

# Message rate limit (per user)
MESSAGE_RATELIMIT=200/day

# Networks granted relay permissions
# Use this with care, all hosts in this networks will be able to send mail without authentication!
RELAYNETS=

# Will relay all outgoing mails if configured
RELAYHOST=

# Fetchmail delay
FETCHMAIL_DELAY=600

# Recipient delimiter, character used to delimiter localpart from custom address part
RECIPIENT_DELIMITER=+

# DMARC rua and ruf email
DMARC_RUA=admin
DMARC_RUF=admin

# Welcome email, enable and set a topic and body if you wish to send welcome
# emails to all users.
WELCOME=false
WELCOME_SUBJECT=Welcome to your new email account
WELCOME_BODY=Welcome to your new email account, if you can read this, then it is configured properly!

# Maildir Compression
# choose compression-method, default: none (value: gz, bz2, lz4, zstd)
COMPRESSION=
# change compression-level, default: 6 (value: 1-9)
COMPRESSION_LEVEL=

# IMAP full-text search is enabled by default. Set the following variable to off in order to disable the feature.
# FULL_TEXT_SEARCH=off

###################################
# Web settings
###################################

# Path to redirect / to
WEBROOT_REDIRECT=/mail

# Path to the admin interface if enabled
WEB_ADMIN=/admin

# Path to the webmail if enabled
WEB_WEBMAIL=/webmail

# Website name
SITENAME=myproject Mail Center

# Linked Website URL
WEBSITE=https://mail.tyyi.net



###################################
# Advanced settings
###################################

# Log driver for front service. Possible values:
# json-file (default)
# journald (On systemd platforms, useful for Fail2Ban integration)
# syslog (Non systemd platforms, Fail2Ban integration. Disables `docker-compose log` for front!)
# LOG_DRIVER=json-file

# Docker-compose project name, this will prepended to containers names.
COMPOSE_PROJECT_NAME=mailu

# Number of rounds used by the password hashing scheme
CREDENTIAL_ROUNDS=12

# Header to take the real ip from
REAL_IP_HEADER=

# IPs for nginx set_real_ip_from (CIDR list separated by commas)
REAL_IP_FROM=

# choose wether mailu bounces (no) or rejects (yes) mail when recipient is unknown (value: yes, no)
REJECT_UNLISTED_RECIPIENT=

# Log level threshold in start.py (value: CRITICAL, ERROR, WARNING, INFO, DEBUG, NOTSET)
LOG_LEVEL=WARNING

# Timezone for the Mailu containers. See this link for all possible values https://en.wikipedia.org/wiki/List_of_tz_database_time_zones
TZ=Asia/Tehran

###################################
# Database settings
###################################
DB_FLAVOR=sqlite

```

</details><details id="bkmrk-first_run.shdocker-c"><summary>first\_run.sh</summary>

```shell
docker compose -p mailu up -d
docker compose -p mailu exec admin flask mailu admin tayyebi tyyi.net 3.1415926535##
```

</details>

# OpenVPN

## Method 1

Ref: [https://medium.com/@gurayy/set-up-a-vpn-server-with-docker-in-5-minutes-a66184882c45](https://medium.com/@gurayy/set-up-a-vpn-server-with-docker-in-5-minutes-a66184882c45)

> Our **OpenVPN** server will also be capable of handling multiple user accounts and different port options thanks to Docker’s easy port exporting options. We will start with **UDP 3000** port which is different than its default port (UDP 1194). We will not use pre-built image and make our own image from a *Dockerfile* . We will clone [this repository](https://github.com/kylemanna/docker-openvpn) and build our image. This image is also ready-to-use on DockerHub ([link](https://hub.docker.com/r/kylemanna/openvpn/)).

```shell
git clone https://github.com/kylemanna/docker-openvpn.git
cd docker-openvpn/
docker build -t myownvpn .
cd ..
mkdir vpn-data && touch vpn-data/vars
```

### Generate OpenVPN config file

```shell
docker run -v $PWD/vpn-data:/etc/openvpn --rm myownvpn ovpn_genconfig -u udp://IP_ADDRESS:3000
```

### PKI

> This covers generating our CA certificate and we will have a private key belong to the PKI

```shell
docker run -v $PWD/vpn-data:/etc/openvpn --rm -it myownvpn ovpn_initpki
```

### Run VPN Server

```
docker run -v $PWD/vpn-data:/etc/openvpn -d -p 3000:1194/udp --cap-add=NET_ADMIN myownvpn
```

### Create user

```
docker run -v $PWD/vpn-data:/etc/openvpn --rm -it myownvpn easyrsa build-client-full user1 nopass
docker run -v $PWD/vpn-data:/etc/openvpn --rm myownvpn ovpn_getclient user1 > user1.ovpn
```

# Developer Tools & Extensions



# Redirector

```
Redirect:	http://localhost:3000/*
to:	https://git.tyyi.net/$1
Hint:	Any word after example.com leads to google search for that word.
Example:	http://localhost:3000/myproject → https://git.tyyi.net/myproject
Applies to:	Main window (address bar)
```

# Ventoy

> Ventoy is an open source tool to create bootable USB drive for ISO/WIM/IMG/VHD(x)/EFI files.  
> With ventoy, you don't need to format the disk over and over, you just need to copy the ISO/WIM/IMG/VHD(x)/EFI files to the USB drive and boot them directly.  
> You can copy many files at a time and ventoy will give you a boot menu to select them.  
> You can also browse ISO/WIM/IMG/VHD(x)/EFI files in local disks and boot them.  
> x86 Legacy BIOS, IA32 UEFI, x86\_64 UEFI, ARM64 UEFI and MIPS64EL UEFI are supported in the same way.

[![image-1667128345299.png](https://library.tyyi.net/uploads/images/gallery/2022-10/scaled-1680-/image-1667128345299.png)](https://library.tyyi.net/uploads/images/gallery/2022-10/image-1667128345299.png)

# zip (upgraded - bash script)

Ref:

[https://askubuntu.com/questions/909918/how-to-show-unzip-progress](https://askubuntu.com/questions/909918/how-to-show-unzip-progress)

Add following lines to *.bashrc* and then run `source .bashrc` :

```shell
# Show progress bar with unzip (based on stackoverflow#909918); usage: punzip file.zip
function plunzip {
    for f in $(unzip -Z -1 $1 | grep -v '/$');
    do
        [[ "$f" =~ "/" ]] && mkdir -p ${f%/*}
        echo "Extracting $f"
        unzip -o -c $1 $f \
            | pv -s $(unzip -Z $1 $f | awk '{print $4}') \
            > $f
    done
}
```

# .htpasswd

> `htpasswd` is used to create and update the flat-files used to store usernames and password for basic authentication of HTTP users. If `htpasswd` cannot access a file, such as not being able to write to the output file or not being able to read the file in order to update it, it returns an error status and makes no changes.

## Create a new password file

```
htpasswd -c /home/pwww/.htpasswd laleh
```

## Change or update password  


```
htpasswd /home/pwww/.htpasswd-users ladan
```

# OpenSSL

## How to generate a self-signed SSL certificate using OpenSSL?

[\[ref\]](https://stackoverflow.com/questions/10175812/how-to-generate-a-self-signed-ssl-certificate-using-openssl)

```shell
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -sha256 -days 365
```

> You can also add `-nodes` (short for "no DES") if you don't want to protect your private key with a passphrase. Otherwise it will prompt you for "at least a 4 character" password.
> 
> The `days` parameter (365) you can replace with any number to affect the expiration date. It will then prompt you for things like "Country Name", but you can just hit <kbd>Enter</kbd> and accept the defaults.
> 
> Add `-subj '/CN=localhost'` to suppress questions about the contents of the certificate (replace `localhost` with your desired domain).
> 
> Self-signed certificates are not validated with any third party unless you import them to the browsers previously. If you need more security, you should use a certificate signed by a certificate authority (CA).

# Domain Name Server (DNS)



# DNS flush

Ref:

[https://devconnected.com/how-to-flush-dns-cache-on-linux](https://devconnected.com/how-to-flush-dns-cache-on-linux)

```shell
sudo systemd-resolve --flush-caches
sudo resolvectl flush-caches
sudo systemd-resolve --statistics
```

# PTR AKA rDNS (Reverse DNS) Records; Necessary for E-Mail Validation

## What is a PTR record?

> A PTR record, also known as a Pointer Record, is a piece of information (a record) that is attached to an email message. The purpose of the PTR record is to verify that the sender matches the IP address it claims to be using.
> 
> This email ID check process is also known as reverse DNS lookup. It’s the reverse of the forward DNS lookup process that browsers use to convert a domain name to a numerical address: the IP address. - \[[ref](https://www.intermedia.com/blog/what-is-a-ptr-record-do-i-need-one/#:~:text=A%20PTR%20record%2C%20also%20known,known%20as%20reverse%20DNS%20lookup. "Intermedia.com")\]

## How rDNS Works?

> This process happens with email too, but in reverse.
> 
> - Every email address is attached to a domain name – it’s what follows the @ symbol. For example, @gmail.com or @tyyi.net.
> - That domain name is matched to an IP address.
> - When you enter an email address, the mail server (the equivalent of a postal mail carrier) matches the receiver’s IP address to the destination. It’s the same thing as a mail carrier matching the address on the outside of an envelope with an actual street address, except this is happening digitally.
> - The mail server also verifies the identity of the sender by using the IP address to obtain the domain name. This is called reverse DNS lookup, and it’s where PTR records are involved.
> 
> The PTR record is the data verifying that the IP address matches the domain name, and it’s the reverse of the “A record,” which provides the IP address associated with the domain.
> 
> - Here’s what the A record for a forward DNS record might look like:
> 
> **tyyi.net &gt; 185.235.41.59**
> 
> - And here’s what a reverse DNS lookup PTR record might look like:
> 
> **185.235.41.59 &gt; tyyi.net**
> 
> So, a PTR record is simply a normal DNS lookup record in reverse. You can use an external tool like [MxToolbox](https://mxtoolbox.com/SuperTool.aspx?action=ptr%3a185.235.41.59&run=toolpage) to look up your PTR record.
