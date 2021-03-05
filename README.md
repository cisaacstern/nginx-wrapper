# nginx-wrapper

## Aims

So you want bring a web project to the cloud with IaaS? Perhaps it's a Flask app running on Gunicorn, or a Panel app running on BokehTornado. This repo demonstrates how to wrap that project for cloud deployment behind an SSL-secured proxy. The remainder of this document is a step-by-step outline of one approach.

> I once read that being an engineering manager is just saying "it depends" over and over again. Perhaps that insight is especially true for devops. When many technologies collide, the number of permutations and caveats increases exponentially. Thus I haven't attempted to create a true how-to here, but rather a trail of breadcrumbs for myself and others with a similar use case.  

## Prerequisites

This approach assumes you have a Git repo which already runs successfully on a local (production-grade) server. In addition, it assumes the top-level directory of that project contains a `Dockerfile` and a `requirements.txt` which collectively build a functional image of the project.

> Your local machine should have the `Docker`, `docker-machine`, and `docker-compose` CLI tools installed. You'll also need an account with www.digitalocean.com and a domain name [managed through that account](https://www.digitalocean.com/docs/networking/dns/).

## Submoduling

Before we go to the cloud, you'll need a new repo, let's call it `nginx-wrapper`, with the following project structure:

```
├── .env.dev
├── .git
├── .gitignore
├── docker-compose.dev.yml
└── services
    └── your-app-repo-name
```

Run:
```
git submodule init $YOUR_APP_GIT_URL services/your-app-repo-name/
``` 
to add your application as a submodule to this repo. Afterwhich, the project should look something like this (that is, if your app were to consist of only a single `app.py` file):

```
├── .env
├── .git
├── .gitignore
├── .gitmodules
├── docker-compose.dev.yml
└── services
    └── your-app-repo-name
        ├── Dockerfile
        ├── requirements.txt
        └── app.py
```

> Submoduling isolates the app's version control from the devops wrapper we're building, making ongoing maintenace and feature-building on the app considerably easier.

The contents of `docker-compose.dev.yml` will vary according to your project and are beyond the scope of this document. Generally speaking, our motivation here is to confirm that running:

```
docker-compose --file docker-compose.dev.yml up -d --build
``` 

builds our app correctly before adding the (considerable) additional complexity of Nginx and SSH.

> Again, probably best to save the `docker-compose` details for another venue, but I'll note that this development build might happen on a local virtual machine created via: 
>```
>docker-machine create --driver virtualbox dev
>```

With full recognition of the abstracted and outline-like nature of this document (so many rabbit holes left unexplored), let's move onto the cloud deployment.

## To the cloud!

### Spin up a droplet

We'll be deploying this project to a DigitalOcean droplet (i.e. virtual machine). To start a new droplet, run the following script, substituting your account's billing `$TOKEN` and other variables accordingly:

> Be aware that this is a purchase! Charges will vary according to droplet size. We've selected the smallest size `s-1vcpu-1gb`, which as of Feb 2021 costs $5/mo.

```docker-machine create \
--driver digitalocean \
--digitalocean-image ubuntu-18-04-x64 \
--digitalocean-access-token $TOKEN \
--digitalocean-region $REGION \
--digitalocean-size s-1vcpu-1gb \                                             
--engine-install-url https://releases.rancher.com/install-docker/19.03.12.sh \
$SERVER_NAME
```
### Initial setup

Once your droplet is created, ssh into it with `docker-machine ssh $SERVER_NAME`, which should log you in as `root`. Following recommendations from [this tutorial](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-18-04) (with a slight modification to allow `http` traffic through the firewall), setup your server as follows:

```adduser $USERNAME
usermod -aG sudo $USERNAME
ufw allow OpenSSH
ufw allow http
ufw enable
```

You can now switch to your non-root user with `su $USERNAME`.

> Your non-root user will likely start in the `/root` directory, in which case you can `cd ..` to the top level.

We specified a Docker install at spin-up time, but we still need to install `docker-compose`, which we can do as our non-root user, [as described here](https://www.digitalocean.com/community/tutorials/how-to-install-docker-compose-on-ubuntu-18-04) (but with a higher version), like so:

```
sudo curl -L https://github.com/docker/compose/releases/download/1.27.4/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose
```

Time to start building our app directory.

### Clone to the cloud

To start, let's `cd home` so we don't clone into the top level.

> At this point, we'll begin following [this DigitalOcean tutorial](https://www.digitalocean.com/community/tutorials/how-to-secure-a-containerized-node-js-application-with-nginx-let-s-encrypt-and-docker-compose) somewhat closely. Though we'll deviate from it stylistically (e.g., by separating our development and production builds with the `docker-compose --file` argument), that link is still a good place to refer to for more detail on what's to come.

We can now clone our project repo from GitHub (recursively, to include submodules) with:

```
sudo git clone --recurse-submodules https://github.com/your-github-username/your-app-repo-name.git
```
> If, like me, your local work happens in macOS Terminal, you can save some strife by remembering that many commonly used CLI tools, including many `git` and `vim` commands, will require the `sudo` preface when working in this Ubuntu VM.

Finally, it's probably a good idea to check our development build with:
```
sudo docker-compose --file docker-compose.dev.yml up -d
```
Assuming we've configured our domain (call it example.com) to point to this droplet via the DigitalOcean web interface, example.com should now be hosting your app (albeit over unsecured HTTP). If this checks out, bring the build down with:
```
sudo docker-compose --file docker-compose.dev.yml down
```
...and we can now move onto the proxy and SSL infrastructure.

### Setting up Nginx with SSL

Our structural approach is slight variation on the theme proposed in the above-linked tutorial, and begins with adding a `docker-compose.prod.yml` file,  an empty `volumes/web-root` directory, an empty `services/webserver/dhparam` directory, and a `services/webserver/nginx-conf` directory containing a proxy configuration file:

```
├── .env
├── .git
├── .gitignore
├── .gitmodules
├── docker-compose.dev.yml
├── docker-compose.prod.yml
├── services
│   ├── your-app-repo-name
│   │   ├── Dockerfile
│   │   ├── requirements.txt
│   │   └── app.py
│   └── webserver
|       ├── dhparam
│       └── nginx-conf
│           └── nginx.conf
└── volumes
    └── web-root
```

The `dockerfile-compose.prod.yml` and `nginx.conf` files follow closely the structure provided in the DigitalOcean tutorial, with a few tweaks as reflected in the files of the same name in this repo.

Once these files are in place, the newly-added `certbot` service can be run in `--staging` mode to obtain ssl certificates by building:

```
docker-compose --file docker-compose.prod.yml up -d
```

Various checks to ensure that the certificates were indeed obtained are nicely enumerated in the linked tutorial, which also does a good job of explaining how to reboot the `certbot` in `--force-renewal` mode, obtain a `dhparam` file through `openssl`, reconfigure the `nginx.conf` for HTTPS, and (finally!) setting up a `cron` job to renew the SSL certificates periodically.

## Editing and maintaining content

Once the site is up and secured, we can modify our app content by selectively bringing down our app service (i.e. container) and rebuilding it with new content.

> If you haven't done so already, this is probably a good time to alias the long docker-compose compound to something like this:
>```
>$ alias prod='sudo docker-compose --file docker-compose.prod.yml'
>```
>This will make our upcoming frequent invocations of that command considerably more concise.

Now to change web content, we can interrupt the current deployment with: 
```
$ prod stop $APP_SERVICE
$ prod rm $APP_SERVICE
```
> **Note** that if we do not `prod rm` the service after stopping it, it will not rebuild our upcoming changes, but rather reuse the existing image.

We can now make any desired changes with `sudo nano`, `sudo vim`, or by `git pull`ing a more recent version from GitHub.

Once we're happy with the changes, we can deploy them with:
```
$ prod up -d --build $APP_SERVICE
```
> For reasons that are not entirely clear to me if the changes we're making are to the `webserver` and not the webapp, it suffices to simply `prod stop webserver` and then `prod up -d --force-recreate --no-deps webserver` (no `prod rm` or explicit `--build` call required).

## Note on resources & websockets

In the case of serving a Bokeh or Panel app on the BokehTornado server, a few additional considerations are worth mentioning.

First, it is presumably possible to serve static resources (such as BokehJS) through the proxy, I found it easiest to use CDN via:

```
from bokeh.settings import settings
settings.resources = 'cdn'
```

And second, care must be taken to configure the Nginx proxy for websockets, as described in this section of the Bokeh documentation: https://docs.bokeh.org/en/latest/docs/user_guide/server.html#reverse-proxying-with-nginx-and-ssl

Please feel free to reach out with questions about any of the above.

