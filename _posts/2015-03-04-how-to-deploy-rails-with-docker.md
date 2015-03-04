---
published: true
title: "如何用Docker部署Rails应用"
layout: post
summary: This post shows us how to deploy a Rails app with docker step by step. 
author: Junhao 
category: 部署
tags:
- Docker
- Rails
---


近期在运维界有一个新兴技术docker特别火，在看了相关的介绍之后果断决定尝试一下用docker部署一台服务器。过程中记录了一下整个操作的过程及相关配置文件，分享给各位也爱追求技术时尚的程序猿们。

### 适用环境

服务器：阿里云 （双核 + 2GB 内存） Ubuntu 14.04

应用的stack： nginx + unicorn + mongodb

### 在阿里云上安装dockerengine

基本按照[官网上的安装指南](https://docs.docker.com/installation/ubuntulinux/)来做的。我刚开始选择的是ubuntu管理的安装包，docker.io， 版本是 1.0.1，发现bug太多，后来重新安装了最新的版本 1.4.1。官网的安装包似乎被墙了，用了网页最下面的Yandex的镜像才把docker安装好。

### 启动docker的daemon程序

正常的情况下只需要执行下面的命令就可以启动docker

{% highlight Bash shell scripts %}
$ sudo service docker start
{% endhighlight %}

但是在阿里云的ECS上报出无闲置IP的错误，百度了一下才找到解决方案，操作步骤如下： 

打开`/etc/network/interfaces`，注释掉以下配置

{% highlight Bash shell scripts %}
# route del -net 172.16.0.0 netmask 255.240.0.0 dev eth0 
{% endhighlight %}

重新启动networking

{% highlight Bash shell scripts %}
$ sudo service networking restart
{% endhighlight %}

重新启动docker

{% highlight Bash shell scripts %}
$ sudo service docker restart
{% endhighlight %}

测试一下docker是否正常运行

{% highlight Bash shell scripts %}
$ docker info
Containers: 33
Images: 176
Storage Driver: aufs
 Root Dir: /var/lib/docker/aufs
 Dirs: 242
Execution Driver: native-0.2
Kernel Version: 3.13.0-32-generic
Operating System: Ubuntu 14.04.1 LTS
CPUs: 2
Total Memory: 3.859 GiB
Name: iZ256yal27dZ
ID: BQ3A:ZJIY:5EOM:JOTY:EROQ:7UI6:SB6P:QVBC:3FM5:DEMB:WBY2:ZDH6
WARNING: No swap limit support
{% endhighlight %}

### 启动nginx的container

在阿里云的机器上构建以下文件夹，并创建相应的文件

{% highlight Bash shell scripts %}
dockers
└── nginx
    ├── Dockerfile
    └── config
        └── nginx-app.conf
{% endhighlight %}

注意：我们暂时先将与rails app有关的配置文件注释了

{% highlight Bash shell scripts %}
# Dockerfile for installing and running Nginx

# Select ubuntu as the base image
From registry.mirrors.aliyuncs.com/library/ubuntu:14.04

# Install nginx
RUN apt-get update
RUN apt-get install -y nginx
RUN echo "daemon off;" >> /etc/nginx/nginx.conf
# ADD config/nginx-app.conf /etc/nginx/sites-enabled/default

# Publish port 80
EXPOSE 80

# Start nginx when container starts
ENTRYPOINT /usr/sbin/nginx
{% endhighlight %}

{% highlight Bash shell scripts %}
# nginx-app.conf

# this can be any application server, not just Unicorn/Rainbows!
upstream rails-app {
  server app:8080 fail_timeout=0;
}

server {
  listen 80 default deferred; # for Linux

  client_max_body_size 4G;
  server_name _;

  keepalive_timeout 5;

  # path for static files
  root /webapps/app/public;

  try_files $uri/index.html $uri.html $uri @unicorn;

  location @unicorn {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_redirect off;
    proxy_pass http://rails-app;
  }

  # Rails error pages
  error_page 500 502 503 504 /500.html;
  location = /500.html {
    root /webapps/app/public;
  }
}
{% endhighlight %}

然后在nginx文件夹下，生成新的docker image，并启动nginx的container

{% highlight Bash shell scripts %}
$ docker build -t junhao/nginx .
$ docker run --name web -d -p 80:80 junhao/nginx
{% endhighlight %}

运行`docker ps`来检查一下container的运行情况

{% highlight Bash shell scripts %}
$ docker ps
CONTAINER ID        IMAGE                  COMMAND                CREATED             STATUS              PORTS                    NAMES
87ae87c89a78        junhao/nginx:latest    "/bin/sh -c /usr/sbi   5 days ago          Up 5 days           0.0.0.0:80->80/tcp       web
{% endhighlight %}

打开浏览器，输入你的阿里云VM地址，应该就能看到“Welcome to Nginx”的页面。阶段性成功，yay！

### 启动unicorn的container

先把rails app上传到服务器上，在应用根目录下创建这样几个文件，`Dockerfile`, `.dockerignore`, `scripts/start-server.sh`

{% highlight Bash shell scripts %}
# Dockerfile for a Rails application using Nginx and Unicorn

# Select ubuntu as the base image
From registry.mirrors.aliyuncs.com/library/ubuntu:14.04

RUN apt-get update -q
RUN apt-get install -qy curl

# Install rvm, ruby, bundler
RUN curl -sSL https://get.rvm.io | bash -s stable
RUN /bin/bash -l -c "rvm requirements"
RUN /bin/bash -l -c "rvm install 2.1.5"
RUN /bin/bash -l -c "gem install bundler --no-ri --no-rdoc"

# Copy the Gemfile and Gemfile.lock into the image. 
# Temporarily set the working directory to where they are. 
WORKDIR /tmp 
ADD ./Gemfile Gemfile
ADD ./Gemfile.lock Gemfile.lock
RUN /bin/bash -l -c "bundle install"

# Add rails project to project directory
ADD ./ /webapps/app

# set WORKDIR
WORKDIR /webapps/app

# bundle install
# RUN /bin/bash -l -c "bundle install"

# Add configuration files in repository to filesystem
ADD scripts/start-server.sh /usr/bin/start-server
RUN chmod +x /usr/bin/start-server

# Publish port 80
EXPOSE 8080

# Startup commands
ENTRYPOINT /usr/bin/start-server
{% endhighlight %}

{% highlight Bash shell scripts %}
# .dockerignore

# Ignore bundler config.
/.bundle

# Ignore the default SQLite database.
/db

# Ignore all logfiles and tempfiles.
/log
/tmp

# Gemfile.lock

# Redis
dump.rdb
{% endhighlight %}

注意：我有一个unicorn的配置文件在config文件夹下，没有用配置文件的需要修改`start-server.sh`的最后一行命令

{% highlight Bash shell scripts %}
#!/bin/bash

cd /webapps/app
source /etc/profile.d/rvm.sh
mkdir -p /webapps/shared/pids
mkdir -p /webapps/shared/log
cat /webapps/shared/pids/unicorn.pid
kill -QUIT `cat /webapps/shared/pids/unicorn.pid`
bundle exec unicorn -c config/unicorn.rb -E production -p 8080
{% endhighlight %}

然后创建unicorn的docker image，并启动container

{% highlight Bash shell scripts %}
$ cd /webapps/app
$ docker build -t junhao/app .
$ docker run --name app -d -p 8080:8080 junhao/app
{% endhighlight %}

接着，我们要对nginx的container做一些改动：把和rails app相关的配置添加回来，并重新创建、启动nginx的container。

打开`dockers/nginx/conf/nginx-app.conf`，把下面这行设置添加回来

{% highlight Bash shell scripts %}
# ADD config/nginx-app.conf /etc/nginx/sites-enabled/default
{% endhighlight %}

然后停止现在的container，并重建container。

{% highlight Bash shell scripts %}
$ cd dockers/nginx
$ docker stop web
$ docker build -t junhao/web .
{% endhighlight %}

下一步就是重启，在重启的时候我们要用到一个叫container linking的技术手法。仔细看一下`nginx-app.conf`，里面有这样一段代码：

{% highlight Bash shell scripts %}
upstream rails-app {
  server app:8080 fail_timeout=0;
}
{% endhighlight %}

这里的`app:8080`中的`app`指的是我们创建的unicorn container。那么在nginx的container中，`app`代表的其实是unicorn container在本机的地址映射。这个是需要我们在启动nginx container的时候做特殊处理的，不然nginx container无法获得相关信息。`--link app:app`就是把app container的信息传递给了web container。

{% highlight Bash shell scripts %}
$ docker run --name web --link app:app -d -p 80:80 junhao/nginx
{% endhighlight %}

现在打开浏览器，试试打开一个不需要访问数据库的页面。

### 配置MongodDB

我用了MongoDB官方的部署服务[MMS](https://mms.mongodb.com)来管理MongoDB，所以没有用docker。大家也可以尝试不同的方法。在本机安装完MongoDB之后，在`config/mongoid.yml`中修改hosts的地址：`- dockerhost:27000`。这里的`dockerhost`指的是container运行的VM的地址。
这个地址我们可以在container启动时定义，由于之前运行时没有定义这个值，我们需要重启app container。

{% highlight Bash shell scripts %}
$ docker stop app
$ docker build -t junhao/app .
$ docker run --name app --add-host=dockerhost:<enter your host address here> -d -p 8080:8080 junhao/app
{% endhighlight %}

然后重启一下web container

{% highlight Bash shell scripts %}
$ docker stop web
$ docker run --name web --link app:app -d -p 80:80 junhao/nginx
{% endhighlight %}

这样就大功告成啦！