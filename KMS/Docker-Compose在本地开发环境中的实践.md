# Docker-Compose在本地开发环境中的实践
> 目前在同步开发的项目系统各自开发环境各不相同，此前通过VMware虚拟机创建各自对应的虚拟机进行本地开发，随之出现的CPU内存占用高，运行卡顿，文件挂载失效以及网络链接等问题出现，定位并修复的时间越来越长。对比Docker与VMware后，选择通过Docker-Compose方式快速搭建本地开发环境，满足快速迭代开发需求。

## Docker与VM对比
1. 结构差异

![image](https://www.backblaze.com/blog/wp-content/uploads/2018/06/whats-the-diff-container-vs-vm.jpg)

VM与Docker容器的差异主要在以下方面：
- 虚拟技术方面。VM是以包含CPU核心在内的硬件基础上，运行完整操作系统，而Docker只包含应用所需要的服务，容器之间共享同一套操作系统资源。
- 隔离方面。VM隔离级别为操作系统级别，会比Docker更加彻底。


2. 优势
- Docker容器启动速度方面，能够实现秒级启动，对比VMware虚拟机要快速得多。同时在开启多个应用时，系统开销更少，能够更高效地利用理系统资源。
- 在系统迁移和运行环境一致性方面，Docker保证了运行环境的一致性，可以非常轻松迁移到另一个平台。同时对比硬盘使用方面，Docker对比于VMware的GB级别的虚拟机文件，磁盘使用更加小巧。
3. 使用场景
Vmware虚拟机更适合于彻底隔离整个运行环境，Docker更加适合于隔离不同的应用。在本地开发环境需求中，Docker更加适合于同时开发多个系统应用，可以节约开发、测试、部署的时间，提高开发效率。

## Docker-compose构建配置记录
> 以校园招聘系统为例（Laravel + Nginx）

- 通过docker-compose可以非常方便描述开发环境所需要组件，同时通过SVN或者Git代码管理工具，三步就能实现开发环境的快速部署。
  - 安装docker与docker-compose
  - 同步项目代码文件与docker-compose.yaml相关文件
  - 执行构建启动命令

以下为相关配置文件示例与构建启动命令：

- docker-compose.yaml

```
version: "3"
services:
  # Laravel项目镜像
  laravel:
    build:
      context: .
      dockerfile: app.dockerfile
    container_name: consumemgr-dev-laravel
    # 项目代码 && PHP配置文件挂载
    volumes:
      - ./:/srv/sites/phpapi/consumemgr.leihuo.netease.com
      - ./php/dev.ini:/usr/local/etc/php/conf.d/dev.ini
      # - ./php/www.conf:/usr/local/etc/php-fpm.d/www.conf
      # - ./php/zz-docker.conf:/usr/local/etc/php-fpm.d/zz-docker.conf
      # - ./php/sock:/var/run
    working_dir: /srv/sites/phpapi/consumemgr.leihuo.netease.com
    environment:
      SERVICE_NAME: app
      SERVICE_TAGS: dev
    # 伪终端
    restart: unless-stopped
    tty: true
    ports:
      - "9000:9000"
    networks:
      - consumemgr-dev

  # Nginx镜像
  nginx:
    build:
      context: .
      dockerfile: nginx.dockerfile
    container_name: consumemgr-dev-nginx
    # Nginx配置文件 && 网站根目录挂载
    volumes:
      - ./:/srv/sites/phpapi/consumemgr.leihuo.netease.com
      - ./nginx/conf.d/:/etc/nginx/conf.d/
      # - ./php/sock:/var/run"
    restart: unless-stopped
    # 伪终端
    tty: true
    ports:
      # 端口映射
      - "80:80"
      - "443:443"
    networks:
      - consumemgr-dev
    links:
      - laravel
  
#Docker Networks
networks:
  consumemgr-dev:
    driver: bridge
```

- laravel-dockerfile

```
FROM php:5.6.37-fpm-alpine

# 更新源
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories

# 设置时区
ENV TIMEZONE Asia/Shanghai
RUN apk update && apk upgrade && \
  apk add tzdata && \
  cp /usr/share/zoneinfo/${TIMEZONE} /etc/localtime && \
  echo "${TIMEZONE}" > /etc/timezone

# 安装PHP依赖
# gd && zip
RUN apk add --no-cache freetype libpng libjpeg-turbo freetype-dev libpng-dev libjpeg-turbo-dev bash-doc bash-completion git openssh libxml2-dev
RUN docker-php-ext-configure gd \
        --with-gd \
        --with-freetype-dir=/usr/include/ \
        --with-png-dir=/usr/include/ \
        --with-jpeg-dir=/usr/include/ \
        --with-zlib-dir=/usr && \
    NPROC=$(grep -c ^processor /proc/cpuinfo 2>/dev/null || 1) && \
    docker-php-ext-install -j${NPROC} gd zip
# pdo_mysql
RUN docker-php-ext-install pdo pdo_mysql

# soap
RUN docker-php-ext-install soap

# curl
RUN apk add --virtual build-dependencies build-base curl
# mongodb
RUN apk add --no-cache zlib zlib-dev libressl libressl-dev autoconf pkgconf g++ make && \
    pecl install mongodb && \
    docker-php-ext-enable mongodb && \
    pecl clear-cache
# 清理
RUN cd / && \
    apk del tzdata && \
    apk del --no-cache freetype-dev libpng-dev libjpeg-turbo-dev autoconf pkgconf g++ make && \
    apk del build-dependencies && \
    rm -rf /var/cache/apk/*

 # composer
RUN curl -sS https://getcomposer.org/installer | php && \
    mv composer.phar /usr/local/bin/ && \
    ln -s /usr/local/bin/composer.phar /usr/local/bin/composer

# # 创建用户sites
RUN addgroup -g 3000 -S sites
RUN adduser -u 3000 -S -D -G sites sites

# 创建代码目录&&项目日志记录
RUN mkdir -p -m 777 /srv/sites/phpapi/consumemgr.leihuo.netease.com && \
    mkdir -p -m 777 /srv/logs/sites/consumemgr && \
    mkdir -p -m 777 /var/run/ && \
    mkdir -p -m 777 /srv/logs/nginx && \
    mkdir -p -m 777 /usr/local/log 

RUN chown -R sites:sites /var/www/ && \
    chown -R sites:sites /var/run/ && \
    chown -R sites:sites /var/log

# 拷贝项目代码
COPY --chown=sites . /srv/sites/phpapi/coinsumemgr.leihuo.netease.com
WORKDIR /srv/sites/phpapi/coinsumemgr.leihuo.netease.com

# # PHP-FPM配置
# # www.conf
# COPY ./php/dev.ini /usr/local/etc/php/conf.d
# # www.conf
# COPY ./php/dev.ini /usr/local/etc/php-fpm.d
# # zz-docker.conf
# COPY ./php/zz-docker.conf /usr/local/etc/php-fpm.d

# 安装Laravel项目依赖
RUN composer install --prefer-dist --no-interaction
ENV PATH="~/.composer/vendor/bin:./vendor/bin:${PATH}"

# Change current user to www
USER sites

# Expose port 9000 and start php-fpm server
EXPOSE 9000
CMD ["php-fpm"]
```

- nginx-dockerfile

```
FROM nginx:1.10-alpine
# # 创建用户
# RUN addgroup -g 3000 -S sites
# RUN adduser -u 3000 -S -D -G sites sites
# # 拷贝
# COPY ./nginx/conf.d/nginx.conf /etc/nginx/conf.d

RUN chgrp -R root /var/cache/nginx /var/run /var/log/nginx && \
    chmod -R 770 /var/cache/nginx /var/run /var/log/nginx
    
# # Change current user to www
# USER sites
```

- 构建启动

```
构建：
docker-compose build
启动：
docker-compose up -d
```


## 后续优化
- 通过自动化脚本完成Docker镜像构建、自动重启更新等功能。
- 整合开发环境Docker镜像，结合部门Docker Hub进行版本管理。
