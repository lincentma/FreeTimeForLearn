## 部署背景
由于项目依赖的数据源迁移到内网，导致对应数据分析服务的Laravel项目也需要同步迁移到内网，由于内网机器变化，区别于目前的发布系统，于是便有了这一次的docker容器化部署实践的机会。

## Dockerfile
> docker的好处之一就是在不同的运行环境（以及嗦扩容）下，只需要一个相同的镜像就可以提供一致的服务，无需重复处理环境的初始化和环境依赖部署工作。

> Dockerfile 是一个文本文件，包含了一系列指令和参数，每一条指令构建一层，最终所有指令在基础镜像上最终创建一个新的镜像。

### 基础镜像

> 当前项目线上运行环境采用PHP-FPM + Nginx + Laravel，为保证环境一致性，采用相同版本的PHP和Nginx，保证服务运行稳定性
- 查看线上环境配置信息

PHP环境版本：
```
PHP 5.6.33-0+deb8u1 (cli) (built: Jan  5 2018 15:46:26) 
Copyright (c) 1997-2016 The PHP Group
Zend Engine v2.6.0, Copyright (c) 1998-2016 Zend Technologies
    with Zend OPcache v7.0.6-dev, Copyright (c) 1999-2016, by Zend Technologies
```

PHP扩展安装：
```
[PHP Modules]
bcmath
bz2
calendar
Core
ctype
curl
date
dba
dom
ereg
exif
fileinfo
filter
ftp
gd
gettext
hash
iconv
json
libxml
mbstring
mcrypt
mhash
mongo
mongodb
mysql
mysqli
openssl
pcntl
pcre
PDO
pdo_mysql
Phar
posix
redis
Reflection
session
shmop
SimpleXML
soap
sockets
SPL
standard
sysvmsg
sysvsem
sysvshm
tokenizer
wddx
xml
xmlreader
xmlwriter
Zend OPcache
zip
zlib

[Zend Modules]
Zend OPcache

```
Nginx版本以及扩展：
```
nginx version: nginx/1.10.1
built with OpenSSL 1.0.1k 8 Jan 2015 (running with OpenSSL 1.0.1t  3 May 2016)
TLS SNI support enabled
configure arguments: --with-cc-opt='-g -O2 -fstack-protector-strong -Wformat -Werror=format-security -D_FORTIFY_SOURCE=2' --with-ld-opt=-Wl,-z,relro --prefix=/usr/share/nginx --conf-path=/etc/nginx/nginx.conf --http-log-path=/var/log/nginx/access.log --error-log-path=/var/log/nginx/error.log --lock-path=/var/lock/nginx.lock --pid-path=/run/nginx.pid --http-client-body-temp-path=/var/lib/nginx/body --http-fastcgi-temp-path=/var/lib/nginx/fastcgi --http-proxy-temp-path=/var/lib/nginx/proxy --http-scgi-temp-path=/var/lib/nginx/scgi --http-uwsgi-temp-path=/var/lib/nginx/uwsgi --with-debug --with-pcre-jit --with-ipv6 --with-http_ssl_module --with-http_stub_status_module --with-http_realip_module --with-http_auth_request_module --with-http_addition_module --with-http_dav_module --with-http_flv_module --with-http_geoip_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_image_filter_module --with-http_mp4_module --with-http_perl_module --with-http_random_index_module --with-http_secure_link_module --with-http_v2_module --with-http_sub_module --with-http_xslt_module --with-mail --with-mail_ssl_module --with-stream --with-stream_ssl_module --with-threads --add-module=/tmp/buildd/nginx-1.10.1/debian/modules/headers-more-nginx-module --add-module=/tmp/buildd/nginx-1.10.1/debian/modules/nginx-auth-pam --add-module=/tmp/buildd/nginx-1.10.1/debian/modules/nginx-cache-purge --add-module=/tmp/buildd/nginx-1.10.1/debian/modules/nginx-dav-ext-module --add-module=/tmp/buildd/nginx-1.10.1/debian/modules/nginx-development-kit --add-module=/tmp/buildd/nginx-1.10.1/debian/modules/nginx-echo --add-module=/tmp/buildd/nginx-1.10.1/debian/modules/ngx-fancyindex --add-module=/tmp/buildd/nginx-1.10.1/debian/modules/nginx-http-push --add-module=/tmp/buildd/nginx-1.10.1/debian/modules/nginx-lua --add-module=/tmp/buildd/nginx-1.10.1/debian/modules/nginx-upload-progress --add-module=/tmp/buildd/nginx-1.10.1/debian/modules/nginx-upstream-fair --add-module=/tmp/buildd/nginx-1.10.1/debian/modules/ngx_http_substitutions_filter_module --add-module=/tmp/buildd/nginx-1.10.1/debian/modules/passenger/src/nginx_module
```

- 选择基础镜像

[Docker Hub](https://hub.docker.com/)为Docker官方维护的公共仓库，大部分的基础基础镜像可以在这里找到。首先查询是否有对应的php-fpm+nginx的基础镜像可以直接使用，发现有一个webdevops/php-nginx镜像中有php5.6版本，但是在后续安装过程中，存在"Fatal Error Zend OPcache cannot allocate buffer for interned strings"，需要覆盖基础镜像的php.ini中相关配置，与其仍要修改配置文件，不如直接采用php-fpm和nginx官方镜像作为基础镜像，进行配置修改。

> 自己当时的考虑是，将PHP-FPM和Nginx打包在一起，会方便后续的部署（虽然也有文章建议容器中只运行单个应用）。同时在一个Dockerfile中是不能够存在多个基础镜像，所以当时考虑的是nginx比php编译安装容易，就采用基础镜像为PHP-5.6 + Nginx 1.10.1编译安装方式。 


- .dockerignore文件

.dockerignore文件可以忽略不需要打包进镜像中的文件，类似于.gitignore，减少Docker镜像的大小。其中'**'表示匹配任意数量的目录（包括零），可以用来忽略多个文件夹目录中的相同文件，例如.gitignore。
```
# 文件
.env
.git
**.gitignore
.idea
.svn

bootstrap/cache/services.php

# 目录
storage/app/*
storage/framework/cache/*
storage/framework/sessions/*
storage/framework/views/*
storage/logs/*

node_modules

vendor
```
    
- 合并Run命令

Dockerfile文件中的每一个指令都会创建一个新的镜像层，将多个Run指令（更新变化频率一致）合并为一个指令就可以降低镜像文件的大小。

- 更新alphine镜像源

由于基础镜像采用的是php:5.6.33-fpm-alpine，通过替换为阿里云镜像可以更加快速的构建完成镜像。


- 扩展安装

由于基础镜像采用alphine镜像和nginx源码包编译安装方式，**对于线上部署环境需要对比查看拿下PHP扩展和Nginx扩展没有安装，并在Dockerfile中补充完善**。例如PHP扩展的gd、zippdo_mysql,以及composer安装需要的git等。

- 完善环境运行配置

在完成运行环境的安装配置后，需要将项目代码拷贝至镜像中，由于没有解压文件添加以及Url拷贝文件的需求，采用COPY命令即可。同时注意由于之前的.dockerignore文件中的设置以及基础镜像与线上环境的区别，也需要**手动创建对应的文件夹保证能够正常运行**。

最终Dockerfile文件内容如下：
```
# 基础镜像
FROM php:5.6.33-fpm-alpine

MAINTAINER maling<maling1@corp.netease.com>

# 更新源
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories

# 创建用户sites
RUN addgroup -S sites
RUN adduser -S -D -h /home sites sites
RUN chown -R sites:sites /srv

ENV TIMEZONE Asia/Shanghai
# 设置时区
RUN apk update && apk upgrade && \
  apk add tzdata && \
  cp /usr/share/zoneinfo/${TIMEZONE} /etc/localtime && \
  echo "${TIMEZONE}" > /etc/timezone

# nginx版本
ENV NGINX_VERSION 1.10.3
# 安装所需要软件
# gd && zip
RUN apk add --no-cache freetype libpng libjpeg-turbo freetype-dev libpng-dev libjpeg-turbo-dev && \
    docker-php-ext-configure gd \
        --with-gd \
        --with-freetype-dir=/usr/include/ \
        --with-png-dir=/usr/include/ \
        --with-jpeg-dir=/usr/include/ \
        --with-zlib-dir=/usr && \
    NPROC=$(grep -c ^processor /proc/cpuinfo 2>/dev/null || 1) && \
    docker-php-ext-install -j${NPROC} gd zip && \
    # pdo_mysql
    docker-php-ext-install mysqli pdo pdo_mysql && \
    # git
    apk add --no-cache bash git openssh && \
    # composer
    curl -sS https://getcomposer.org/installer | php && \
    mv composer.phar /usr/local/bin/ && \
    ln -s /usr/local/bin/composer.phar /usr/local/bin/composer && \
    # nginx
    apk --update add pcre-dev openssl-dev && \
    apk add --virtual build-dependencies build-base curl && \
    curl -SLO http://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz && \
    tar xzvf nginx-${NGINX_VERSION}.tar.gz && \
    cd nginx-${NGINX_VERSION} && \
    ./configure \
        --with-debug \
        --with-ipv6 \
        --with-mail \
        --with-mail_ssl_module \
        --with-http_ssl_module \
        --with-http_realip_module \
        --with-http_stub_status_module \
        --with-http_gzip_static_module \
        --with-http_gunzip_module \
        --with-http_dav_module \
        --with-http_flv_module \
        --with-http_mp4_module \
        --with-http_addition_module \
        --with-http_auth_request_module \
        --with-http_secure_link_module \
        --http-client-body-temp-path=/var/tmp/nginx/client \
        --http-proxy-temp-path=/var/tmp/nginx/proxy \
        --http-fastcgi-temp-path=/var/tmp/nginx/fastcgi \
        --http-uwsgi-temp-path=/var/tmp/nginx/uwsgi \
        --prefix=/usr/share/nginx \
        --conf-path=/etc/nginx/nginx.conf \
        --sbin-path=/usr/local/sbin/nginx \
        --pid-path=/var/run/nginx.pid \
        --lock-path=/var/lock/nginx.lock \
        --http-log-path=/var/log/nginx/access.log \
        --error-log-path=/var/log/nginx/error.log && \
    make && \
    make install && \
    ln -sf /dev/stdout /var/log/nginx/access.log && \
    ln -sf /dev/stderr /var/log/nginx/error.log && \
    # supervisor
    apk add supervisor && \
    # del
    cd / && \
    apk del tzdata && \
    apk del --no-cache freetype-dev libpng-dev libjpeg-turbo-dev && \
    apk del build-dependencies && \
    rm -rf \
        nginx-${NGINX_VERSION} \
        nginx-${NGINX_VERSION}.tar.gz \
        /var/cache/apk/*

# 创建代码目录&&项目日志记录
RUN mkdir -p -m 777 /srv/sites/phpapi/confluence-plugin.leihuo.netease.com && \
    mkdir -p -m 777 /srv/logs/sites/confluence && \
    mkdir -p -m 777 /var/tmp/nginx/{client,proxy,fastcgi,uwsgi,scgi} && \
    mkdir -p -m 777 /srv/logs/nginx && \
    mkdir -p -m 777 /etc/nginx/sites-enabled && \
    mkdir -p -m 777 /etc/supervisor.d && \
    mkdir -p -m 777 /var/log/supervisor

# 拷贝项目代码
COPY . /srv/sites/phpapi/confluence-plugin.leihuo.netease.com
WORKDIR /srv/sites/phpapi/confluence-plugin.leihuo.netease.com

# 安装laravel项目
RUN composer install --prefer-dist --no-interaction
ENV PATH="~/.composer/vendor/bin:./vendor/bin:${PATH}"

# 拷贝php-fpm配置文件
# php-fpm.conf
COPY ./php-fpm.conf /usr/local/etc
# www.conf
COPY ./www.conf /usr/local/etc/php-fpm.d
# zz-docker.conf
COPY ./zz-docker.conf /usr/local/etc/php-fpm.d
# 拷贝nginx配置文件
# nginx.conf
COPY ./nginx.conf /etc/nginx
# confluence-plugin.leihuo.netease.com
COPY ./confluence-plugin.leihuo.netease.com /etc/nginx/sites-enabled/
COPY ./proxy_params /etc/nginx/proxy_params
# 权限设置
RUN chmod 777 -R /srv

# 拷贝supervisor配置文件
COPY ./supervisord.conf /etc/supervisord.conf

# 拷贝crontab文件
COPY ./crontab /var/spool/cron/crontabs/
RUN cat /var/spool/cron/crontabs/crontab >> /var/spool/cron/crontabs/root
RUN mkdir -p /var/log/cron && \
    touch /var/log/cron/cron.log

# 暴露端口
EXPOSE 8080 443 80

# 执行命nginx服务和php-fpm服务启动
ENV SUPERVISOR_CONF_FILE=/etc/supervisord.conf
ENTRYPOINT supervisord -n -c $SUPERVISOR_CONF_FILE
```

## 相关配置文件实践记录

### PHP-FPM
- 在修改为使用sock套接字连接时，在修改了php-fpm.conf和www.conf文件后，启动后依然没有找到生成对应的sock文件。
- 关键是在于zz-docker.conf文件，由于php-fpm.conf采用include加载所有的配置文件，而zz-docker.conf是按照字母排序规则是最后加载的，会将前面的相同的配置文件都覆盖，所以修改zz-docker.conf才会最终生效。

```
[global]
daemonize = no

[www]
;listen = 9000
listen = /var/run/php-fpm.sock
listen.mode = 0666
```


### Nginx
- 需要注意的是nginx运行用户和php-fpm运行用户保持一致，默认配置会有不同，需要进行覆盖。

### Laravel
- composer安装同样可以采用阿里云镜像地址，提高composer安装速度和镜像build速度。

### Supervisor
- PHP-FPM+Nginx前台启动，由于自己在本地测试的时候是通过php-fpm -D的方式进行测试，也就是默认是daemon方式启动，但是这种后台运行的守护进程方式，是无法使用supervisor进行监控进程的。所以需要php-fpm -F强制前台启动。
- 前台启动的方式，同时满足了Docker容器中至少需要一个进程挂起到前台的要求，否则认为容器会停止运行，因为Docker容器仅在 1 号进程（PID为1）运行时，会保持运行。

supervisor.conf文件内容如下：
```
; Sample supervisor config file.

[unix_http_server]
file=/run/supervisord.sock   ; (the path to the socket file)
;chmod=0700                  ; socked file mode (default 0700)
;chown=nobody:nogroup        ; socket file uid:gid owner
;username=user               ; (default is no username (open server))
;password=123                ; (default is no password (open server))

[inet_http_server]          ; inet (TCP) server disabled by default
port=0.0.0.0:9001         ; (ip_address:port specifier, *:port for all iface)
;username=user               ; (default is no username (open server))
;password=123                ; (default is no password (open server))

[supervisord]
logfile=/var/log/supervisord.log ; (main log file;default $CWD/supervisord.log)
;logfile_maxbytes=50MB       ; (max main logfile bytes b4 rotation;default 50MB)
;logfile_backups=10          ; (num of main logfile rotation backups;default 10)
loglevel=info                ; (log level;default info; others: debug,warn,trace)
;pidfile=/run/supervisord.pid ; (supervisord pidfile;default supervisord.pid)
;nodaemon=false              ; (start in foreground if true;default false)
;minfds=1024                 ; (min. avail startup file descriptors;default 1024)
;minprocs=200                ; (min. avail process descriptors;default 200)
;umask=022                   ; (process file creation umask;default 022)
;user=chrism                 ; (default is current user, required if root)
;identifier=supervisor       ; (supervisord identifier, default is 'supervisor')
;directory=/tmp              ; (default is not to cd during start)
;nocleanup=true              ; (don't clean up tempfiles at start;default false)
;childlogdir=/var/log/supervisor ; ('AUTO' child log dir, default $TEMP)
;environment=KEY=value       ; (key value pairs to add to environment)
;strip_ansi=false            ; (strip ansi escape codes in logs; def. false)

; the below section must remain in the config file for RPC
; (supervisorctl/web interface) to work, additional interfaces may be
; added by defining them in separate rpcinterface: sections
[rpcinterface:supervisor]
; Sample supervisor config file.
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

[supervisorctl]
serverurl=unix:///run/supervisord.sock ; use a unix:// URL  for a unix socket
;serverurl=http://127.0.0.1:9001 ; use an http:// url to specify an inet socket
;username=chris              ; should be same as http_username if set
;password=123                ; should be same as http_password if set
;prompt=mysupervisor         ; cmd line prompt (default "supervisor")
;history_file=~/.sc_history  ; use readline history if available

[supervisord]
logfile=/var/log/supervisord.log ; (main log file;default $CWD/supervisord.log)
logfile_maxbytes=50MB        ; (max main logfile bytes b4 rotation;default 50MB)
logfile_backups=10           ; (num of main logfile rotation backups;default 10)
loglevel=info                ; (log level;default info; others: debug,warn,trace)
pidfile=/var/run/supervisord.pid ; (supervisord pidfile;default supervisord.pid)
nodaemon=false               ; (start in foreground if true;default false)
minfds=1024                  ; (min. avail startup file descriptors;default 1024)
minprocs=200                 ; (min. avail process descriptors;default 200)
user=root

; the below section must remain in the config file for RPC
; (supervisorctl/web interface) to work, additional interfaces may be
; added by defining them in separate rpcinterface: sections
[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

[supervisorctl]
serverurl=unix:///var/run/supervisor.sock ; use a unix:// URL  for a unix socket

[program:php-fpm]
command=/usr/local/sbin/php-fpm -F
autostart=true
autorestart=true
priority=5
stdout_events_enabled=true
stderr_events_enabled=true

[program:nginx]
command=/usr/local/sbin/nginx -g 'daemon off;'
autostart=true
autorestart=true
priority=10
stdout_events_enabled=true
stderr_events_enabled=true

[program:cron]
command=/usr/sbin/crond -f
autostart=true
autorestart=true

[include]
files = /etc/supervisor/conf.d/*.conf
```


### Docker-compose
> Docker Compose可以快速部署分布式应用，例如应用和mysql和redis联合部署
- mysql和redis端口
redis默认端口为6379，需要通过"command: --port 36379"命令来更改启动端口，同时通过ports属性将容器内的端口映射到容器外。

docker-compose.yml内容如下：
```
version: "3"
services:
  laravel:
    image: hub.leihuo.netease.com/web/confluence-plugin:0.1.3
    container_name: laravel
    ports:
      - "8080:80"
    env_file:
      - .env
    depends_on:
      - laravel-db
      - laravel-redis
  laravel-db:
    image: mysql:5.6.46
    container_name: laravel-db
    ports:
      - "33306:3306"
    expose:
      - "33306"
    environment:
      MYSQL_USER: "*"
      MYSQL_PASSWORD: "*"
      MYSQL_ROOT_PASSWORD: "*"
      MYSQL_DATABASE: "*"
  laravel-redis:
    image: redis:5.0.7
    container_name: laravel-redis
    command: --port 36379
    ports:
      - "36379:36379"
    expose:
      - "36379"
    restart: always

```

## 写在最后
- 通过查阅资料和实践，从0到1的将线上开发环境同项目代码打包为docker镜像，并通过docker-compose完成部署，学习和认识到docker容器化对于项目迁移的高效率和便利性。
- 将项目部署环境容器化，基础镜像通用标准化，使得其他项目可以在通用的基础镜像上，直接添加项目代码即可完成部署运行，这是自己认为后续的优化努力的方向。此外，关于HTTPS证书配置，项目数据库相关配置文件、日志文件以及持久化数据抽离到宿主机方式，也是后面需要不断学习完善的地方。
- 由于当前项目只需要单机器部署，可以通过docker-compose进行快速管理。但随着业务服务增加，以及多机器跨机房的需求，则需要通过K8s来实现更为复杂，高可用的容器管理维护服务。在学习[【雷火学堂】「丹炉云平台 」容器专场](http://salon.netease.com/index.html#/offline/courseDetail/s002911)中，也了解到了基于丹炉容器云平台所做的优秀实践案例，希望可以在后续业务开发过程中有更好的实践。
