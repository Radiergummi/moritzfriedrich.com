---
title: "Portable Development Setup"
date: 2020-08-23T09:52:08Z
draft: false
tags: 
    - blog
    - chromebook
    - development
---
Due to switching jobs, and my beloved 2013-ish MacBook Pro finally failing on me for good, I'm in a
somewhat unsuspected situation: The only computing devices available to me are my smartphone and an
old Chromebook which Google gifted to me at MessengerPeople, so we could have a Meet coversation with
their sales people.  
It is a rather old and cheap model, with a slow dual core CPU and 3GB of RAM--all in all, the last
thing you'd probably want to do any reasonable software development on. 

Now, what is _also_ special about the current situation is that I absolutely have the time to 
follow stupid ideas all the way through! Assessing the situation, I have a low-performance local
device and a high(er)-performance VPS with Hetzner. Pretty much what the picture looked like to
mainframe operators in prior decades: They solved the demand for workstations by using cheap thin
clients connecting to the expensive mainframe backend, and this is exactly what we're going to
do--_build a portable development environment, accessible from any type of low-power device over the
internet._

TL;DR The code is available on GitHub: 
[Radiergummi/portable-development-environment](https://github.com/Radiergummi/portable-development-environment).

Choosing the components
-----------------------
Usually, I work with [Jetbrains' PhpStorm](https://www.jetbrains.com/phpstorm/) exclusively, which is
pretty much the gold-standard in IDEs to me. Its heaps of features and convenient auto-completion of
course come at a price, though, which is the amount of compute resources and memory it guzzles.
Intellij, the Jetbrains platform, is built on Java (probably Kotlin, by now), and its definitely 
showing. Therefore, PhpStorm is out of the question.  
Looking around for lighter solutions, I first took a look at web-based code editors, available as Chrome
apps, such as [Caret](http://thomaswilburn.net/caret/). That might be a viable solution if you're only 
looking to write markdown documentation or something, but it's certainly unsuitable for actual software
development.  
Finally, I learned about [`code-server`](https://github.com/cdr/code-server), which is basically just a 
headless Visual Studio Code, accessible via HTTP, rendering in a client browser instead of Electron.
This makes a lots of sense in a way--if an application works exclusively using web tech already, there's
no reason not to make use of it. `code-server` is maintained by [Coder](https://coder.com/), which more
or less provide a battle-tested, commercial variant of what I'm attempting to do, so check it out if 
you're interested.  
While `code-server` has sort-of authentication built in, I'm not going to rely on it. Having an IDE with
a shell and file system access exposed over the internet is risky enough already, so instead, we'll need
some sort of authentication proxy between the web and the application. To model the flow of HTTP 
requests, I usually rely [nginx](https://nginx.org/), which has us covered here: By using the 
[`auth-request` module](https://nginx.org/en/docs/http/ngx_http_auth_request_module.html), we can 
authorize all requests for a given location. This module merely queries another HTTP endpoint, passes
the original request, and evaluates the status code of the authorization server response to decide
whether to deny or forward the client request.  
There are sveral solutions available for such an authentication proxy, but I decided to go with 
[Vouch Proxy](https://github.com/vouch/vouch-proxy), as it is straight-forward to setup and does exactly
what I want it to, namely forward authentication to an OAuth2 provider of my choice!  

As soon as the number of applications involved in a system grows, I host them with Docker. This might 
seem overkill, but after years of setting up and maintaining Linux servers, I'm no longer inclined to 
waste time here when I could simply define the full stack with docker-compose and move on. Therefore, my
VPS is essentially just a (hardened) Docker host.

So to sum up, we need a compose stack with:
 - **code-server**, a web-based IDE
 - **vouch-proxy**, an authentication proxy
 - **nginx**, a reverse proxy server
 - A plethora of development tools such as node, php, composer etc., which we'll get to later

Finishing off, we will also need a domain for our IDE and one for vouch, both of which require an SSL 
certificate to be served securely. This will actually be pretty expensive, because... ha, just kidding!
It's 2020, Let's Encrypt is everywhere and for the life of me I will _never_ buy a certificate again.  

Preparing the Chromebook
------------------------
I'm going to skip the introduction to Chromebooks here, suffice it to say, they run a heavily 
customized, Google derivate of Gentoo, basically only exposing the Chrome browser as its user interface. 
For some reason, I suspect users bugging Google, ChromeOS got the capability to run a Linux _"VM"_ 
(there's a super-interesting description of its internals 
[available here](https://chromium.googlesource.com/chromiumos/docs/+/master/containers_and_vms.md), 
which also explains why VM has quotes around it). This is integrated elegantly into the system, enabled
with only a few clicks in the UI, and amounts to a Debian shell with password-less sudo.

To enable the Linux VM, simply open the settings app, navigate to _Linux (Beta)_, and follow the wizard. 
As soon as the shell window pops up, we're ready to continue to the server side of things. Let's quickly 
generate an SSH key and dial in to the box:
```bash
ssh-keygen -o -a 100 -t ed25519 -f ~/.ssh/id_ed25519 -C "Chromebook"
ssh-copy-id moritz@my-vps.example.com
ssh moritz@my-vps.example.com
```

Preparing the server
--------------------
I'm assuming a fresh, uninitialized server, updates installed and hardened security. I'm probably
going to do a write-up of that in another post soon.

### Setting up Docker
First up, let's install Docker _the recommended way_. Despite the 
[documentation being quite clear](https://docs.docker.com/engine/install/ubuntu/) on this, I've 
seen people do an `apt install docker` way too often. So doing this correctly involves exactly the
following steps:
```bash
# Install base dependencies
sudo apt-get update && sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

# Add Docker’s official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

# Add the Docker apt repository
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

# Finally, install the Docker packages
sudo apt-get update && sudo apt-get install -y \
    docker-ce \
    docker-ce-cli \
    containerd.io
```

Now you've got Docker installed, but you're still not able to use it without root privileges. The 
internet contains approximately a gazillion issues on this, but the solution is quite simple! We
just need a group for Docker, and our user needs to be a part of it:
```bash
# Add the Docker OS group
sudo groupadd docker

# Add your own user to the Docker group
sudo usermod -aG docker $USER

# Apply group changes without having to log off and back in again
newgrp docker
```

We probably want Docker to start on boot:
```bash
sudo systemctl enable docker
```


### Designing the compose stack
What I like a lot about docker-compose stacks is that we can design it iteratively - get the basics
to work, refine the services, restart. So let's start by defining the raw services:
```yaml
version: '3.7'
services:
    front-proxy:
        image: 'nginx:latest'
        restart: always
        ports:
            - 80
            - 443

    auth-proxy:
        image: 'voucher/vouch-proxy:latest'
        restart: always
        expose:
            - 8080

    ide:
        build: .
        restart: always
        expose:
            - 8080
```

Here, we define our three components and the way network communication is intended to flow: The front
proxy (nginx) will listen on ports `80` and `443` of the host itself, so it's the entry point into our
system, from the internet.  

### Configuring nginx
We will configure nginx to forward requests for the IDE to the `ide` service, and authentication requests
to `auth-proxy` next. To do so, we add our configuration files:
```yaml
    front-proxy:
        image: 'nginx:latest'
        restart: always
        volumes:
            - './nginx.conf:/etc/nginx/nginx.conf'
        ports:
            - 80
            - 443
 ```

Here, we override the main nginx configuration file with our own. It should look like the following, 
with the default values omitted for brevity:
```nginx
http {
    # basic settings, logging, gzip etc.

    server {
        server_name vouch.example.com;

        location / {
            proxy_pass http://auth-proxy:8080;
            proxy_set_header Host $http_host;
        }

        listen 80;
    }

    server {
        server_name ide.example.com;

        location / {
            proxy_pass                         http://ide:8080;
            proxy_set_header Host              $http_host;
            proxy_set_header X-Real-IP         $remote_addr;
            proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Upgrade           $http_upgrade;
            proxy_set_header Connection        upgrade;
            proxy_set_header Accept-Encoding   gzip;
        }
        
        listen 80;

        # Any request to this server will first be sent to this URL
        auth_request /vouch-validate;

        location = /vouch-validate {
            # This address is where Vouch will be listening on
            proxy_pass http://auth-proxy:8080/validate;
            proxy_pass_request_body off;

            proxy_set_header Content-Length    "";
            proxy_set_header X-Real-IP         $remote_addr;
            proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Host              $http_host;

            auth_request_set $auth_resp_jwt          $upstream_http_x_vouch_jwt;
            auth_request_set $auth_resp_err          $upstream_http_x_vouch_err;
            auth_request_set $auth_resp_failcount    $upstream_http_x_vouch_failcount;
            auth_request_set $auth_resp_x_vouch_user $upstream_http_x_vouch_user;
        }

        error_page 401 = @error401;

        # If the user is not logged in, redirect them to Vouch's login URL
        location @error401 {
            return 302 https://vouch.example.com/login?url=https://$http_host$request_uri&vouch-failcount=$auth_resp_failcount&X-Vouch-Token=$auth_resp_jwt&error=$auth_resp_err;
        }
    }
}
```
 
This configuration defines two virtual servers: One for vouch, our authentication proxy, and one for the
actual application. The vouch server simply passes requests to the vouch service defined in our 
docker-compose configuration--we can take advantage of the fact that Docker even handles the name 
resolution for us.  
The second server for our application also passes requests to the application service, sets a bunch of
request headers, but more importantly: It enables the `auth_request` module and configures nginx to pass
all requests to the vouch validation endpoint first. Vouch checks whether the request is authenticated,
and either redirects them to the login page or signals nginx to let the request pass through to our app.

If you've looked carefully, you might notice both blocks listen on port 80, with no mention of SSL yet.
Getting Let's Encrypt certificates and their renewal to play nicely with Docker requires some effort, so
we'll get back to that later on.

### Configuring vouch
For vouch to actually validate requests for us, we'll need to tell it how to do so. While vouch actually 
supports environment variable configuration, I've found it to be more convenient to use a configuration
file instead. The first step is updating our compose configuration with the path to this file:
```yaml
    auth-proxy:
        image: 'voucher/vouch-proxy:latest'
        restart: always
        environment:
            VOUCH_CONFIG: '/vouch.yaml'
        volumes:
            - './vouch.yaml:/vouch.yaml'
        expose:
            - 8080
 ```

We set an environment variable that points to our configuration file (in the context of our container 
file system), then mount the file from the working directory to the configured container path.  
The contents look akin to like the following:
```yaml
vouch:
    logLevel: info
    testing: false

    # We want to listen to all interfaces, as we're in a Docker
    # context. This will not expose vouch directly.
    listen: '0.0.0.0'
    port: 8080

    domains:
        - vouch.example.com
        - ide.example.com
        - example.com

    # Make sure to point the cookie domain to your domain name.
    # It must be shared by the auth domain and the app domain.
    cookie:
        domain: example.com

    # Approve user account names here as necessary. Vouch has
    # more options to configure this, take a look at its docs.
    whiteList:
        - Radiergummi

oauth:
    provider: github
    client_id: ffffffff1c511ca9ebc9
    client_secret: ffffffffffe314b9315d4a1b9af487a75e72f32a
    callback_url: 'https://vouch.example.com/auth'
    scopes:
        - user
```

It's important to get the domain list and OAuth provider configuration right. You'll probably want to
set this up according to your liking, but I've found maintaining an OAuth app with GitHub the easiest in
the past--no extra hoops to jump through, and straight-forward documentation.  
Take a look at the 
[example configuration file](https://github.com/vouch/vouch-proxy/blob/master/config/config.yml_example) 
for detailed information on the available directives.

### Building the IDE image
Now that we've got the front proxy and the authentication proxy working, we can finally get to the most
interesting part: The `ide` service and its Docker image. There's something important to understand about
our setup: Visual Studio Code will, for all intents and purposes, run inside this Docker container. Any
terminal opened in the UI will spawn inside this Docker container. Any files we'll be working with will
live inside this Docker container.  
This means that we are not just building a container for `code-server` to run in, but also an interactive
development environment we should adjust to our requirements. The nice thing about that is that we're 
going to define our complete workspace with code!

So the first thing we're going to do is, again, adjusting the docker-compose configuration to mount our
data properly:
```yaml
    ide:
        build: .
        restart: always
        expose:
            - 8080
        volumes:
            - './code-server.yaml:/code-server.yaml'
            - './user-home:/user-home'
            - './user-data:/user-data'
            - './projects:/projects'
```

#### Mounted volumes
As this container is our development environment, we want certain data to persist across restarts. Our volumes are used as follows:

 - `code-server.yaml`: A configuration file for code-server. It allows us to specify command-line flags to the
   executable without having to modify the `Dockerfile` later on.
 - `user-home/`: The home directory of our developer user account _in the Docker image_. A bunch of 
   important configuration files will live there.
 - `user-data/`: The Visual Studio Code data directory, containing configuration files, installed 
   extensions and so on.
 - `projects/`: The project root directory for our version controlled project directories.

We'll get to each of those in more detail later on. First, let's focus on the `Dockerfile`.

#### The `Dockerfile`
It has actually two purposes: First is building `code-server`, second defining our runtime environment.
That fits nicely with multi-stage builds! So first, let's define the build image only.
```dockerfile
FROM node:latest AS code-server-builder
WORKDIR /opt

# Install code-server build dependencies
RUN set -eux; \
    apt-get update; \
    apt-get install -y \
        build-essential \
        pkg-config \
        libx11-dev \
        libxkbfile-dev \
        libsecret-1-dev

# Install (and build) code-server
RUN yarn add code-server
```

This will install the `code-server` package and subsequently build it, resulting in a `node_modules/`
directory containing the server binary and all dependencies.  
As the next step, we want to define the runtime environment. This is heavily specific for your own 
workflow, so I'll keep this as generic as possible:
```dockerfile
FROM node:latest AS code-server-builder
WORKDIR /opt

# Install code-server build dependencies
RUN set -eux; \
    apt-get update; \
    apt-get install -y \
        build-essential \
        pkg-config \
        libx11-dev \
        libxkbfile-dev \
        libsecret-1-dev

# Install (and build) code-server
RUN yarn add code-server

#
# Actual runtime environment definition
#
FROM ubuntu:latest
WORKDIR /opt

# Install development dependencies
RUN set -eux; \
    apt-get update; \
    apt-get install -y \
    apt-transport-https \
    git \
    curl \
    gnupg \
    nano;

# Install node.js
RUN set -eux; \
    curl -sL https://deb.nodesource.com/setup_14.x | bash -; \
    apt-get update; \
    apt-get -y install nodejs

# Copy the actually built code-server
COPY --from=code-server-builder /opt/node_modules /opt/node_modules

# Add the home directory of our user as a volume
VOLUME /user-home

# Create the "developer" user in the container
RUN groupadd --gid 5000 developer && \
    useradd --home-dir /user-home \
        --uid 5000 \
        --gid 5000 \
        --shell /bin/bash \
        developer

# Switch to the new user account
USER developer

# Start code-server with our config file
CMD [ "/opt/node_modules/.bin/code-server", "--config", "/code-server.yaml" ]
```
My setup starts from the `php-cli` image instead, installs xdebug, composer, yarn and a few other 
packages in addition to what you see above. As I said, this is very much the stuff you require for your
own workflow, so customize this image as much as necessary.  

Before we get to finally boot the stack, we'll have to adjust file system permissions on the host:
```bash
mkdir user-home/ user-data/ projects/
chown -R 5000:5000 user-home/ user-data/ projects/
chmod -R 755 user-home/ user-data/ projects/
```

And create the configuration file for `code-server`:
```yaml
# Again, we bind to all interfaces of the container only here
bind-addr: '0.0.0.0:8080'

# As we have vouch taking care of authentication, we can skip it here
auth: none
user-data-dir: /user-data
```

### Configuring nginx, part 2: HTTPS
If you're using Cloudflare or a similar service, you might want to consider skipping the Let's Encrypt 
part and simply configure your CDN to terminate HTTPS for you. Otherwise, read on for automatic renewal
from within a Docker stack.

There is a guide by [@wmnnd](https://github.com/wmnnd/) which descibes this process in greater detail available 
[on Medium](https://medium.com/@pentacent/nginx-and-lets-encrypt-with-docker-in-less-than-5-minutes-b4b8a60d3a71).

First off, we'll need to modify our front proxy service:
```yaml
    front-proxy:
        image: 'nginx:latest'
        restart: always
        volumes:
            - './nginx.conf:/etc/nginx/nginx.conf'
            - './certbot/conf:/etc/letsencrypt'
            - './certbot/www:/var/www/certbot'
        ports:
            - 80
            - 443
        command: "/bin/sh -c 'while :; do sleep 6h & wait $${!}; nginx -s reload; done & nginx -g \"daemon off;\"'"
```

This adds a volume for certbot configuration and one for its web root to complete challenges, and overrides the nginx
command so it will reload every six hours and apply eventually renewed certs.

The nginx configuration must be modified to have a new, HTTP-only server block:
```nginx
http {
    # ...

    server {
        listen 80 default_server;
        server_name _;

        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }

        location / {
            return 301 https://$host$request_uri;
        }
    }

     # ...other server blocks
}
```

Modify the virtual server blocks for auth and ide with the usual SSL stuff (make sure to update the certificate 
path!):
```nginx
server {
    listen 443 ssl http2;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    include             /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam         /etc/letsencrypt/ssl-dhparams.pem;

    # ...other directives
}
```
Don't worry, I've got the final versions of the configuration files 
[in the accompanying GitHub repository](https://github.com/Radiergummi/portable-development-environment), too!

To actually request and renew certificates, we'll need a certbot service:
```yaml
  certbot:
    image: certbot/certbot
    restart: unless-stopped
    volumes:
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"
```

Finally, you can either do the last bit of footwork manually using the certbot image or just adapt @wmnnd's 
excellent script to your setup:  
[init-letsencrypt.sh](https://github.com/wmnnd/nginx-certbot/blob/master/init-letsencrypt.sh).

Wrapping it up
--------------
What does this leave us with? 
 - A running Visual Studio Code installation running headlessly
 - A secure, OAuth-powered login with a provider of our choice
 - An up-to-date Ubuntu container with all our daily drivers installed within
 - A persistent home directory with our tool configurations
 - A persistent VS Code configuration directory

We have just created our own, code-defined development environment. It is able to run everywhere Docker 
is running. It is hosted securely, only accessible to us. Its power is only limited by the server 
hardware we put below it. It can be powered down and back up again at any time!

As soon as you fire up the stack with `docker-compose up -d`, the services will start. If you've configured
everything correctly, you should now be able to navigate to `https://ide.example.com` and be greeted by the UI of
Visual Studio Code.  
On our Chromebook, we can install the progressive web app using the small plus button in the URL bar. This will also
open the editor in its own, Chrome-less window, and is probably the next best thing to running VS Code natively on 
any other notebook!  
It looks like this on my box:
![A screenshot of the code-server window](/img/vscode-chromebook.png)

I'm soon going to work with a beefy, new MacBook Pro from the current generation, and definitely going to use
PhpStorm again. Nevertheless, I'm both astonished what's possible with current technology and proud of the
environment I've been able to bootstrap in an afternoon of work. No matter the situation, I will always have an
emergency IDE available from anywhere in the future. If I was working with Visual Studio Code primarily, I would 
really consider this setup as my go-to environment!
