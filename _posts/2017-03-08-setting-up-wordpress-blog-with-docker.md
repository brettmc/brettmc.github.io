---
title: Setting up a Wordpress blog with docker
---

> When your only tool is a hammer, every problem looks like a nail.

My current tool of choice is docker, so since I'm setting up a blog, I might as well do it via docker. My favourite method of throwing up a container or two is the excellent docker-compose tool. In this case, I want a Wordpress container, plus a database as a data store.

```yaml
version: '3.1'
services:
  mariadb:
    image: 'mariadb:latest'
    environment:
      - MYSQL_ROOT_PASSWORD=secretpassword
    restart: always
    volumes:
      - './mariadb_data:/var/lib/mysql'
  wordpress:
    image: 'wordpress:php7.1'
    ports:
      - '80:80'
      - '443:443'
    restart: always
    environment:
      - WORDPRESS_DB_HOST=mariadb
      - WORDPRESS_DB_USER=root
      - WORDPRESS_DB_PASSWORD=secretpassword
    depends_on:
      - mariadb
```
NB the "restart: always" tells docker to restart the container if it ever dies.

Starting up the blog is as simple as:

```
$ docker-compose up -d
```
The blog is exposed on ports 80 and 443 (I still need to set up SSL/TLS, probably via lets-encrypt - if I'm lucky, there's a docker image for that already).

Once everything is up and running, open it up in a browser to go through the installation process.
