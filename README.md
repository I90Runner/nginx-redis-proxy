
# Presentation

nginx-redis-proxy is a reverse proxy based on nginx and redis to cache objects (web pages and more).

# How it works

A http client requests a web page to the frontend nginx. Nginx looks for the corresponding cached object in redis.
If the object is in redis, nginx serves it.
If the object is not in redis, nginx requests a backend that generates the page and gives it back to nginx. Then, nginx put it in redis and serves it.
All cached objects in redis has a TTL, so they are uncached naturally without any other process.

The frontend is listening port 80. The backend is using nginx and listening port 8080 with PHP configured. You can use what you want as backend.

All the cache management is written using LUA script.

The 'nginx_conf/' directory contains the nginx configuration files :

* _www-php.conf_ : PHP backend configuration, listening port 8080
* _proxy-redis.conf_ : frontend configuration with cache engine, listening port 80
* _nginx-status.conf_ : just the common and simple monitoring status configuration, listening port 88

The 'doc_root/' directory contains some test pages in html and php. I used 'index.php' and 'david.php' to test the cache management.

# Requirements

nginx-redis-proxy is working with :

* LUA v5.1
* LuaJIT v2.0.0-beta8
* LuaRedisParser v0.09rc5 (redis.parser Lua library)
* redis v2.2.2
* nginx v1.0.9
* nginx modules :
 * HttpSetMiscModule v0.22rc2
 * HttpUpstreamKeepaliveModule v0.7
 * HttpHeadersMoreModule v0.16rc2
 * HttpRedis2Module v 0.07rc6
 * HttpLuaModule v0.3.1rc24
 * NDK (Nginx Development Kit) v0.2.17rc2
 * HttpEchoModule v0.37rc4

The system used is CentOS 5.7.

# Installation

## Redis :

see Installation section at http://redis.io/download.

## Lua :

The nginx HttpLuaModule needs Lua and LuaJIT.
see : http://www.lua.org/ - http://luajit.org/

First, you must have these packages :

```
$ yum install ncurses-devel readline-devel
```
Then, install Lua :

```
$ cd /opt/install
$ wget http://www.lua.org/ftp/lua-5.1.tar.gz
$ tar zxvf lua-5.1.tar.gz
$ cd lua-5.1
$ make
$ make linux
$ make install
```

## LuaJIT :

LuaJIT will be installed in /opt/luajit.

```
$ cd /opt/install
$ wget http://luajit.org/download/LuaJIT-2.0.0-beta8.tar.gz
$ tar zxvf LuaJIT-2.0.0-beta8.tar.gz
$ cd LuaJIT-2.0.0-beta8
$ mkdir /opt/luajit
$ make PREFIX=/opt/luajit
$ make install PREFIX=/opt/luajit
$ cd /opt/luajit/bin/
$ ln -s luajit-2.0.0-beta8 luajit
```

Add this in the .bash_profile :

```
export LD_LIBRARY_PATH=/opt/luajit/lib:$LD_LIBRARY_PATH
```

## LuaRedisParser :

```
$ wget --no-check-certificate https://github.com/agentzh/lua-redis-parser/tarball/v0.09rc5
$ mv v0.09rc5 lua-redis-parser_v0.09rc5.tar.gz
$ tar zxvf lua-redis-parser_v0.09rc5.tar.gz
$ cd agentzh-lua-redis-parser-2f302a6
$ make
$ make INSTALL_PATH=/usr/local/lib/lua/5.1/ install
```

## Nginx modules :

Download and uncompress all these nginx modules :

#### HttpSetMiscModule :

```
$ wget --no-check-certificate https://github.com/agentzh/set-misc-nginx-module/tarball/v0.22rc2
$ tar zxvf ...
```

#### HttpUpstreamKeepaliveModule :

```
$ wget http://mdounin.ru/hg/ngx_http_upstream_keepalive/archive/tip.tar.gz
$ tar zxvf ...
```

#### HttpHeadersMoreModule :

```
$ wget --no-check-certificate https://github.com/agentzh/headers-more-nginx-module/tarball/v0.16rc2
$ tar zxvf ...
```

#### HttpRedis2Module :

```
$ wget --no-check-certificate https://github.com/agentzh/redis2-nginx-module/tarball/v0.07rc6
$ tar zxvf ...
```

#### HttpLuaModule and NDK :

Note : NDK (Nginx Development Kit) is needed to compile nginx with HttpLuaModule - cf. http://wiki.nginx.org/HttpLuaModule

```
$ wget --no-check-certificate https://github.com/simpl/ngx_devel_kit/tarball/v0.2.17rc2
$ tar zxvf ...
$ wget --no-check-certificate https://github.com/chaoslawful/lua-nginx-module/tarball/v0.3.1rc24
$ tar zxvf ...
```

#### HttpEchoModule :

```
$ wget --no-check-certificate https://github.com/agentzh/echo-nginx-module/tarball/v0.37rc4
$ tar zxvf ...
```

## Nginx :

```
$ cd /opt/install
$ wget http://nginx.org/download/nginx-1.0.9.tar.gz
$ tar zxvf nginx-1.0.9.tar.gz
$ cd nginx-1.0.9
```

Now, choose between Lua and LuaJIT :

```
// nginx needs to know where is Lua :
//$ export LUA_LIB=/path/to/lua/lib
//$ export LUA_INC=/path/to/lua/include
```
OR :
```
// we say where is LuaJIT if we want to use it instead of Lua, and we want because LuaJIT is faster :
$ export LUAJIT_LIB=/opt/luajit/lib/
$ export LUAJIT_INC=/opt/luajit/include/luajit-2.0
```
I choose LuaJIT !

Configure :

```
$ ./configure --with-debug \
              --prefix=/opt/nginx \
              --error-log-path=/var/log/nginx/error.log \
              --http-log-path=/var/log/nginx/access.log \
              --pid-path=/var/run/nginx/nginx.pid \
              --lock-path=/var/lock/nginx.lock \
              --user=nginx \
              --group=nginx \
              --with-http_stub_status_module \
              --http-client-body-temp-path=/var/tmp/nginx/client/ \
              --http-fastcgi-temp-path=/var/tmp/nginx/fcgi/ \
              --http-proxy-temp-path=/var/tmp/nginx/proxy/ \
              --without-mail_pop3_module \
              --without-mail_imap_module \
              --without-mail_smtp_module \
              --add-module=/opt/install/simpl-ngx_devel_kit-bc97eea_v0.2.17rc2/ \
              --add-module=/opt/install/agentzh-set-misc-nginx-module-ecaa4e9-v0.22rc2/ \
              --add-module=/opt/install/agentzh-echo-nginx-module-03a2fd2/ \
              --add-module=/opt/install/chaoslawful-lua-nginx-module-4d92cb1/ \
              --add-module=/opt/install/agentzh-headers-more-nginx-module-b3c6230-v0.16rc2/ \
              --add-module=/opt/install/agentzh-redis2-nginx-module-05a0e22-v0.07rc6/ \
              --add-module=/opt/install/ngx_http_upstream_keepalive-d9ac9ad67f45/
```

Compile and install :

```
$ make
$ make install
$ useradd nginx
$ mkdir -p /var/tmp/nginx/{client,fcgi,proxy}/
```
