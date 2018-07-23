---
title: Docker + Wordpress + letsencrypt
---

Now that [letsencrypt](https://letsencrypt.org) is a thing, cost is no longer a barrier to having secure connections to your website. I've just set up this blog, via docker, on a VPS I had lying around. It's not secure, and it'll probably get hacked, eventually. I'd prefer that didn't happen.

I aim to build a docker image based on the official Wordpress image, which can have SSL enabled via letsencrypt.

First, the Dockerfile:
```Dockerfile
FROM wordpress:php7.1

RUN echo "export TERM=xterm LANG=en_US.UTF-8" &gt;&gt; ~/.bashrc \
    &amp;&amp; apt-get update &amp;&amp; apt-get -y install git \
    &amp;&amp; rm -rf "/opt/letsencrypt" \
    &amp;&amp; git clone https://github.com/letsencrypt/letsencrypt /opt/letsencrypt \
    &amp;&amp; cd /opt/letsencrypt \
    &amp;&amp; ./letsencrypt-auto --version
```
That will create a docker image, based on the latest php7.1+apache Wordpress image, and adds the letsencrypt software.

I've updated my docker-compose.yml file from the initial setup, so that it looks like this:

```yaml
version: '3.1'
services:
  mariadb:
    image: 'mariadb:latest'
    environment:
      - MYSQL_ROOT_PASSWORD=secretpasswordgoeshere
    restart: always
    volumes:
      - './mariadb_data:/var/lib/mysql'
  wordpress_tls:
    build: ./wordpress_tls
    ports:
      - '80:80'
      - '443:443'
    restart: always
    volumes:
      - ./letsencrypt_data:/etc/letsencrypt
    environment:
      - WORDPRESS_DB_HOST=mariadb
      - WORDPRESS_DB_USER=root
      - WORDPRESS_DB_PASSWORD=secretpasswordgoeshere
    depends_on:
      - mariadb
```
Now, to generate SSL keys:

```bash
$ docker-compose exec wordpress_tls /opt/letsencrypt/certbot-auto --apache -d your.domain.com --agree-tos -n -m you@your.domain.com
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Obtaining a new certificate
Performing the following challenges:
...snip...
```
And that's it! Pain-free. lets-encrypt SSL certificates are valid for 3 months, but the renewal process is similarly painless:

```bash
$ docker-compose exec wordpress_tls /opt/letsencrypt/certbot-auto renew
```
Running this straight away informs me that the cert it not due for renewal yet, but they recommend renewing about every 60 days to give you some breathing space.
