---
title: "Weekly news: docker.sock access and letsencrypt!"
date: 2015-12-04 13:00
comments: true
author: Kyle Kelley <kyle.kelley@rackspace.com>
published: true
excerpt: >
  In this week's roundup: We give you docker.sock, configure free letsencrypt
  TLS/SSL certs, and more!
categories:
 - Encryption
 - TLS
 - SSL
 - Docker
 - letsencrypt
 - Carina
 - News
 - Security
authorIsRacker: true
---

# Status page

We now have a status page up at https://carinabyrackspace.statuspage.io/

There have been issues with the public swarm discovery service and we'd like
to keep you all apprised of issues anywhere on our platform.

# docker.sock is now available

You can now mount `docker.sock` directly and use it:

```
$ docker run --rm -v /var/run/docker.sock:/var/run/docker.sock docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED                  STATUS                  PORTS                                      NAMES
a0581acf56c4        docker              "docker-entrypoint.sh"   Less than a second ago   Up Less than a second                                              kickass_perlman
b8afd1744cb3        nginxcellent        "nginx -g 'daemon off"   18 minutes ago           Up 18 minutes           0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp   nginx
423c16bf838d        swarm:1.0.0         "/swarm manage -H=tcp"   27 hours ago             Up 27 hours             2375/tcp, 0.0.0.0:2376->2376/tcp           swarm-manager
e7a9df39621d        swarm:1.0.0         "/swarm join --addr=1"   27 hours ago             Up 27 hours             2375/tcp                                   swarm-agent
```

# Let's Encrypt with Free Certificates

Yesterday [Let's Encrypt](https://letsencrypt.org/) went public, providing
TLS certificates to all, for *free*.

On Carina, you can set this up pretty quickly.

Prerequisites:

* [Carina cluster with only one segment](https://getcarina.com/docs/tutorials/create-connect-cluster/) and Docker environment set up
* DNS "A" record set to the IP of your node
* `docker` at your fingertips

## Setup the volume for lets-encrypt data

```
docker volume create --name letsencrypt
```

## Setup the volume for lets-encrypt backups (optional, recommended)

```
docker volume create --name letsencrypt-backups
```

## Let's Encrypt!

Now we'll use Let's Encrypt's Docker image to automate acceptance of the terms
of service (recommended reading) as well as generate certs for a domain. You'll
want to set both the domain you want to use as well as the email address to be
the point of contact.

:warning: You *must* have an A record set to this host (`$DOCKER_HOST`) in order
for letsencrypt to be able to verify you own the domain. :warning:

```
docker run -it --rm -p 443:443 -p 80:80 \
  -v letsencrypt:/etc/letsencrypt \
  -v letsencrypt-backups:/var/lib/letsencrypt \
  --name letsencrypt quay.io/letsencrypt/letsencrypt:latest \
  auth -d lets.ephem.it --email rgbkrk@gmail.com --agree-tos
```

After that, you should see output like the following:

```
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at
   /etc/letsencrypt/live/lets.ephem.it/fullchain.pem. Your cert will
   expire on 2016-03-02. To obtain a new version of the certificate in
   the future, simply run Let's Encrypt again.
 - If like Let's Encrypt, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

## Accessing the certificates

The certs are available on the `letsencrypt` volume at `{mountpoint}/live/{domain}/`.

```bash
$ docker run -it -v letsencrypt:/etc/letsencrypt \
    busybox ls /etc/letsencrypt/live/lets.ephem.it
cert.pem       chain.pem      fullchain.pem  privkey.pem
```

Let's launch an nginx container to verify this is all set up. First thing we'll
need is an `nginx.conf`. The one I used comes from the [Mozilla SSL Configuration
Generator](https://mozilla.github.io/server-side-tls/ssl-config-generator/).

```nginx
{% include_relative 2015-12-04-weekly-news-docker-sock-letsencrypt/lets.conf %}
```

The most important parts to modify are:

```
ssl_certificate /etc/letsencrypt/live/lets.ephem.it/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/lets.ephem.it/privkey.pem;
...
## verify chain of trust of OCSP response using Root CA and Intermediate certs
ssl_trusted_certificate /etc/letsencrypt/live/lets.ephem.it/chain.pem;
```

Now that you have your own default.conf, we'll need a Docker image to run. Here's
the Dockerfile:

```Dockerfile
{% include_relative 2015-12-04-weekly-news-docker-sock-letsencrypt/Dockerfile %}
```

index.html is just the text "We're Let's Encrypted!"

Go ahead and build it:

```bash
$ docker build -t nginxcellent .
Sending build context to Docker daemon 5.632 kB
Step 1 : FROM nginx
 ---> 198a73cfd686
Step 2 : COPY lets.conf /etc/nginx/conf.d/default.conf
 ---> 8934baea98eb
Removing intermediate container 35bc1da7d8a3
Step 3 : COPY index.html /data/www/index.html
 ---> 41e0e1ef05b8
Removing intermediate container 922f6c7890e8
Successfully built 41e0e1ef05b8
```

Now run it and `curl` against it:

```bash
$ docker run -d -p 80:80 -p 443:443 \
  -v letsencrypt:/etc/letsencrypt --name nginx nginxcellent
b8afd1744cb390279501bf4f1ee80a55bd1baafa9ae65f52e6b3b17db35a4c7e
$ curl https://lets.ephem.it
We're Let's Encrypted!
```

That's it! You now have encryption set up for a site! You'll need to renew the
certificates every 90 days, which is worth another post.

## Clean up

While not necessary, you can do ahead and delete the docker image for letsencrypt
now that you have the certs:

```
docker rmi quay.io/letsencrypt/letsencrypt:latest
```

# Wrap up

That's all folks! See you next time.