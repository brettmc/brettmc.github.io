---
title: docker + alpine linux + php + sql server
---

I've been trying to create an Alpine-based docker image which can connect to a SQL Server database from PHP 7.1.

The official microsoft ODBC driver can be used for [Ubuntu](https://sqlchoice.azurewebsites.net/en-us/sql-server/developer-get-started/php/ubuntu/) and [RHEL/centOS](https://sqlchoice.azurewebsites.net/en-us/sql-server/developer-get-started/php/rhel/), and Microsoft provide packages for this, but according to some github comments I found the Microsoft developers, they [couldn't compile the driver under Alpine](https://github.com/Microsoft/msphpsql/issues/300#issuecomment-282458676). I did manage to get the source and almost compile it, but it really wanted glibc.

However, what I did get working was using TDS and pdo_odbc (based on ideas from [Remi's blog post](https://blog.remirepo.net/post/2016/09/20/Microsoft-SQL-Server-from-PHP)):

```Dockerfile
FROM php:7-alpine
RUN apk update \
    && apk add --no-cache --virtual .php-build-dependencies \
        autoconf \
        g++ \
        make \
        unixodbc-dev \
    && apk add --virtual .php-runtime-dependencies \
        freetds \
        unixodbc \
    && docker-php-source extract \
    && docker-php-ext-configure pdo_odbc --with-pdo-odbc=unixODBC,/usr \
    && docker-php-ext-install \
        pdo_odbc \
    && docker-php-source delete \
    && apk del .php-build-dependencies \
    && rm -rf /var/cache/apk/* /var/tmp/* /tmp/*
COPY conf/odbc/*.ini /etc/
```

odbc.ini:

```INI
[sqlsrv_freetds]
Driver=FreeTDS
Description=SQL via FreeTds
Port=1433
```
odbcinst.ini:

```INI
[FreeTDS]
Description=FreeTDS
Driver=/usr/lib/libtdsodbc.so.0.0.0
```
I've deliberately kept the .ini files minimalistic, as I prefer to have the connection string config-based, or use a database abstraction layer such as Zend-Db or doctrine.

```php
php> $pdo = new PDO('odbc:Driver={FreeTDS};Server=sql-server-host.domain.com;Database=MyDatabase;Port=1433', 'username', 'password');
```
