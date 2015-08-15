---
layout: post
title: "使用StatsD, Graphite, Grafana, Kamon搭建可用于JVM项目的可视化性能监控系统"
date: 2015-07-21 16:12:13 +0800
comments: true
categories: [StatsD, Graphite, Grafana, Kamon, Play Framework]
description: "StatsD, Graphite, Grafana, Kamon的介绍"
---

### 1. 什么是性能监控系统
这里说的性能监控系统，主要侧重点是监控应用系统的性能。
说直白点就是每个业务（例如注册，登录）的请求响应时间，请求次数等信息。
操作系统的监控不是这里的重点，因为业界已经有许多相当成熟的基于Linux的运维系统。
操作系统的运维和应用系统的运维是两码事，应用系统的运维相对来说没有这么多选择。
而对于任何线上系统来说，运维监控系统又是必不可少的。
如果你是在大公司，一般会选择开发自己的运维系统，而对于中小团队，因为人力有限，大多会采用开源的解决方案。  

我在这里要介绍的就是使用StatsD，Graphite, Grafana, Kamon搭建监控系统的方案。
这里的Kamon只适用于基于JVM的项目，另外，如果你的项目是基于Play或Akka或Spray，可以做到不写代码实现监控。
因为Kamon对这几个框架提供了AspectJ支持，在类加载的时候插入代码为你完成记录。
其他情况你需要调用Kamon API来进行数据记录，这也非常的简单。

### 2. 监控系统UI
介绍搭建步骤之前，先来看一看搭好后的界面。
{% img /images/custom/20150721/grafana1.png %}
{% img /images/custom/20150721/grafana2.png %}
转自(https://github.com/kamon-io/docker-grafana-graphite)  
这里包含4部分的图表。  
- Actor Metrics是Akka Actor的统计图表。因为我的项目中没有直接使用Akka，所以暂时忽略这一部分内容。  
- Trace Metrics是业务请求的统计图表。例如每个请求的响应时间，以及某个时间段内各请求数量的统计对比。  
- OS Metrics一看就知道是操作系统的统计图表了。  
- JVM Metrics是JVM的统计图表。  

<!-- more -->

### 3. StatsD, Graphite, Grafana, Kamon简介
简单介绍一下这四个开源项目，因为都在Github上就不贴链接了，  
已经对这几个项目很熟悉的可以略过。
#### 1. StatsD
StatsD是一个用于记录统计信息的守护进程。使用NodeJS开发，提供各种语言的客户端API。
#### 2. Graphite
使用Python开发，分为三个子项目  
- carbon 守护进程，接收StatsD发送过来的原始统计数据。  
- whisper 用来存储统计数据的库。  
- graphite webapp 用来图形化展示统计数据的web项目。
#### 3. Grafana
使用Go开发，可以直接在界面上设计统计图表。  
之前看到就是使用Grafana制作的界面。
#### 4. Kamon
一套类库用来记录统计数据，使用Scala开发，提供Java和Scala API。  
除了提供API，还结合AspectJ对一些框架提供自动记录的功能，当然性能上会有损耗。  

整体流程：业务系统调用Kamon的Api记录数据，Kamon将数据发送给StatsD，  
StatsD定期（默认10s）将数据汇总发送到Graphite，  
当用户访问Grafana界面的时候，Grafana调用Graphite接口读取数据绘制成图形展示给用户。

### 4.搭建环境
因为用到了多个项目，并且每个项目都基于不同语言，所以安装过程肯定不会很简单。  
使用docker的朋友直接参考这个项目(https://github.com/kamon-io/docker-grafana-graphite)  
用docker镜像会方便很多。  

如果不用docker，可以参考我的安装步骤。  
以下安装是基于Ubuntu14.04，如果是CentOS或其他系统可能某些步骤会不一样。  
另外因为测试的时候是在开发机上，所以下面安装的时候没有使用独立用户和权限。  

```
1.安装需要的软件

sudo apt-get -y install software-properties-common

sudo add-apt-repository -y ppa:chris-lea/node.js

sudo apt-get -y update

sudo apt-get -y install python-django-tagging python-simplejson python-memcache python-ldap python-cairo python-pysqlite2 python-support \

                           python-pip gunicorn nginx-light nodejs wget curl build-essential python-dev



sudo pip install Twisted==11.1.0

sudo pip install Django==1.5



# Install Elasticsearch

cd ~ && wget https://download.elasticsearch.org/elasticsearch/elasticsearch/elasticsearch-1.3.2.deb

cd ~ && sudo dpkg -i elasticsearch-1.3.2.deb && rm elasticsearch-1.3.2.deb



2.从源码安装StatsD, Graphite, Grafana

# Checkout the stable branches of Graphite, Carbon and Whisper and install from there

mkdir ~/src



git clone https://github.com/kamon-io/docker-grafana-graphite.git ~/src/docker-grafana-graphite



git clone https://github.com/graphite-project/whisper.git ~/src/whisper &&\

cd ~/src/whisper                                                                   &&\

git checkout 0.9.x                                                                &&\

sudo python setup.py install



git clone https://github.com/graphite-project/carbon.git ~/src/carbon              &&\

cd ~/src/carbon                                                                    &&\

git checkout 0.9.x                                                                &&\

sudo python setup.py install



git clone https://github.com/graphite-project/graphite-web.git ~/src/graphite-web  &&\

cd ~/src/graphite-web                                                              &&\

git checkout 0.9.x                                                                &&\

sudo python setup.py install



# Install StatsD

git clone https://github.com/etsy/statsd.git ~/src/statsd                                                                        &&\

cd ~/src/statsd                                                                                                                  &&\

git checkout v0.7.2



# Install Grafana

mkdir ~/src/grafana

wget http://grafanarel.s3.amazonaws.com/grafana-1.9.1.tar.gz -O ~/src/grafana.tar.gz                   &&\

        cd ~/src/ && tar -xzf ~/src/grafana.tar.gz && mv ~/src/grafana-1.9.1 ~/src/grafana && cd - &&\

        rm ~/src/grafana.tar.gz



3.修改配置

# Configure Elasticsearch

sudo mkdir -p /tmp/elasticsearch



# Confiure StatsD

cp ~/src/docker-grafana-graphite/statsd/config.js ~/src/statsd/config.js



# Configure Whisper, Carbon and Graphite-Web

sudo cp ~/src/docker-grafana-graphite/graphite/initial_data.json /opt/graphite/webapp/graphite/

sudo cp ~/src/docker-grafana-graphite/graphite/local_settings.py /opt/graphite/webapp/graphite/

# 此时要 sudo vi /opt/graphite/webapp/graphite/local_settings.py, 把TimeZone改为Asia/Shanghai

sudo cp ~/src/docker-grafana-graphite/graphite/carbon.conf /opt/graphite/conf

sudo cp ~/src/docker-grafana-graphite/graphite/storage-aggregation.conf /opt/graphite/conf

sudo cp ~/src/docker-grafana-graphite/graphite/storage-schemas.conf /opt/graphite/conf



sudo mkdir -p /opt/graphite/storage/whisper

sudo touch /opt/graphite/storage/graphite.db /opt/graphite/storage/index

sudo chmod 0775 /opt/graphite/storage /opt/graphite/storage/whisper

sudo chmod 0664 /opt/graphite/storage/graphite.db

cd /opt/graphite/webapp/graphite && sudo python manage.py syncdb --noinput



# Configure Grafana

cp ~/src/docker-grafana-graphite/grafana/config.js ~/src/grafana/config.js



# Add the default dashboards

mkdir ~/src/dashboards

cp ~/src/docker-grafana-graphite/grafana/dashboards/* ~/src/dashboards/



# Configure nginx

将下面的内容加入nginx.conf

  server {

    listen 80 default_server;

    server_name _;

    location / {

     # !!! change me 这里改成你的目录 !!!

      root /home/liubin/src/grafana;

      index index.html;

    }

    location /graphite/ {

        proxy_pass                 http://127.0.0.1:8000/;

        proxy_set_header           X-Real-IP   $remote_addr;

        proxy_set_header           X-Forwarded-For  $proxy_add_x_forwarded_for;

        proxy_set_header           X-Forwarded-Proto  $scheme;

        proxy_set_header           X-Forwarded-Server  $host;

        proxy_set_header           X-Forwarded-Host  $host;

        proxy_set_header           Host  $host;



        client_max_body_size       10m;

        client_body_buffer_size    128k;



        proxy_connect_timeout      90;

        proxy_send_timeout         90;

        proxy_read_timeout         90;



        proxy_buffer_size          4k;

        proxy_buffers              4 32k;

        proxy_busy_buffers_size    64k;

        proxy_temp_file_write_size 64k;



        add_header Access-Control-Allow-Origin "*";

        add_header Access-Control-Allow-Methods "GET, OPTIONS";

        add_header Access-Control-Allow-Headers "origin, authorization, accept";

    }



    location /elasticsearch/ {

        proxy_pass                 http://127.0.0.1:9200/;

        proxy_set_header           X-Real-IP   $remote_addr;

        proxy_set_header           X-Forwarded-For  $proxy_add_x_forwarded_for;

        proxy_set_header           X-Forwarded-Proto  $scheme;

        proxy_set_header           X-Forwarded-Server  $host;

        proxy_set_header           X-Forwarded-Host  $host;

        proxy_set_header           Host  $host;



        client_max_body_size       10m;

        client_body_buffer_size    128k;



        proxy_connect_timeout      90;

        proxy_send_timeout         90;

        proxy_read_timeout         90;



        proxy_buffer_size          4k;

        proxy_buffers              4 32k;

        proxy_busy_buffers_size    64k;

        proxy_temp_file_write_size 64k;

    }

  }



  server {

    listen 81 default_server;

    server_name _;



    open_log_file_cache max=1000 inactive=20s min_uses=2 valid=1m;



    location / {

        proxy_pass                 http://127.0.0.1:8000;

        proxy_set_header           X-Real-IP   $remote_addr;

        proxy_set_header           X-Forwarded-For  $proxy_add_x_forwarded_for;

        proxy_set_header           X-Forwarded-Proto  $scheme;

        proxy_set_header           X-Forwarded-Server  $host;

        proxy_set_header           X-Forwarded-Host  $host;

        proxy_set_header           Host  $host;



        client_max_body_size       10m;

        client_body_buffer_size    128k;



        proxy_connect_timeout      90;

        proxy_send_timeout         90;

        proxy_read_timeout         90;



        proxy_buffer_size          4k;

        proxy_buffers              4 32k;

        proxy_busy_buffers_size    64k;

        proxy_temp_file_write_size 64k;

    }



    add_header Access-Control-Allow-Origin "*";

    add_header Access-Control-Allow-Methods "GET, OPTIONS";

    add_header Access-Control-Allow-Headers "origin, authorization, accept";



    location /content {

      alias /opt/graphite/webapp/content;

    }



    location /media {

      alias /usr/share/pyshared/django/contrib/admin/media;

    }

  }



4.启动

export GRAPHITE_STORAGE_DIR='/opt/graphite/storage'

export GRAPHITE_CONF_DIR='/opt/graphite/conf'



# run nginx

sudo /etc/init.d/nginx restart



# run carbon

sudo /opt/graphite/bin/carbon-cache.py --debug start



# run graphite-web

export PYTHONPATH='/opt/graphite/webapp'

cd /opt/graphite/webapp

sudo /usr/bin/gunicorn_django -b127.0.0.1:8000 -w2 graphite/settings.py



# run StatsD

sudo /usr/bin/node ~/src/statsd/stats.js ~/src/statsd/config.js



# run elasticsearch

sudo /etc/init.d/elasticsearch start



# run grafana

cd ~/src/dashboards

sudo /usr/bin/node dashboard-loader.js system-metrics.json welcome.json
```

安装完成后，可以尝试访问(http://127.0.0.1)，如果出现之前的界面并且没有报错，就代表安装成功了。  

### 5. 配置项目
主要是引入Kamon依赖。  
因为我们的项目基于Play，所以直接使用了Kamon-Play依赖。

#### 1.修改build.sbt
```Scala
val kamonVersion = "0.4.0"
//...
val dependencies = Seq(
  "io.kamon" %% "kamon-core" % kamonVersion,
  "io.kamon" %% "kamon-statsd" % kamonVersion,
  "io.kamon" %% "kamon-play" % kamonVersion,
  "io.kamon" %% "kamon-system-metrics" % kamonVersion,
  "org.aspectj" % "aspectjweaver" % "1.8.1"
)
```

#### 2.修改application.conf
```
akka {
  extensions = ["kamon.statsd.StatsD", "kamon.system.SystemMetrics"]
}

kamon {

  metric {
    tick-interval = 1 second
  }

  statsd {
    # Hostname and port in which your StatsD is running. Remember that StatsD packets are sent using UDP and
    # setting unreachable hosts and/or not open ports wont be warned by the Kamon, your data wont go anywhere.
    hostname = "127.0.0.1"
    port = 8125

    # Interval between metrics data flushes to StatsD. It's value must be equal or greater than the
    # kamon.metrics.tick-interval setting.
    flush-interval = 1 second

    # Max packet size for UDP metrics data sent to StatsD.
    max-packet-size = 1024 bytes

    # Subscription patterns used to select which metrics will be pushed to StatsD. Note that first, metrics
    # collection for your desired entities must be activated under the kamon.metrics.filters settings.
    includes {
      actor      =  [ "*" ]
      trace      =  [ "*" ]
      dispatcher =  [ "*" ]
    }

    simple-metric-key-generator {
      # Application prefix for all metrics pushed to StatsD. The default namespacing scheme for metrics follows
      # this pattern:
      #    application.host.entity.entity-name.metric-name
      # !!! 这里改成项目名 !!!
      application = "sk-shop"
    }
  }

  play {
    include-trace-token-header = true
    trace-token-header-name = "X-Trace-Token"
  }
}
```

#### 3.启动项目
play运行时环境分为dev环境和prod环境。
因为Kamon-Play使用AspectJ在类加载加载的时候进行织入，AspectJ会使用自己的类加载器，
而运行在dev环境的Play项目因为要实现热加载，所以dev环境不能使用AspectJ。
prod一般会会使用dist命令将项目打包，可用以下方式启动（假设项目名是shop，并且已cd进入打包后的目录）：  
```
./bin/shop -J-javaagent:lib/org.aspectj.aspectjweaver-1.8.1.jar  
```
启动之后随便访问你项目几个页面，然后访问127.0.0.1，如果一切正常就可以看到数据了。  

### 6.总结
恭喜，如果做到这一步，你就已经初步的搭好你们运维系统的架子了。
如果你的应用比较简单，对性能要求也不高，到这一步就可以结束了。
不过，大多数的应用都对统计数据有更进一步的分析需求，例如绘制响应时间超过某一阀值（例如100ms）的饼状图，
或者绘制日注册人数的直方图。
这就需要你手动调用Kamon的API来记录数据了，不过这相当的简单。
如果需要设计其他的Grafana图表，需要对Graphite的函数比较熟悉。  

另外，如果应用对性能很敏感，不推荐使用AspectJ。因为LTW（Load Time Weaving）会有一些性能损耗。
我们的项目最开始是使用的Kamon-Play和Kamon-Akka，不过后来测试发现响应时间平均要增加10%-20%，
现在已经改成直接调用Kamon API完成记录。  

Enjoy!

### 7.附启动和停止脚本
#### 1.启动脚本 start_stats.sh
```
#!/bin/sh

export GRAPHITE_STORAGE_DIR='/opt/graphite/storage'
export GRAPHITE_CONF_DIR='/opt/graphite/conf'
nohup /opt/graphite/bin/carbon-cache.py --debug start > ~/logs/kamon/carbon.out 2>&1 &
export PYTHONPATH='/opt/graphite/webapp'
nohup /usr/bin/gunicorn_django -b127.0.0.1:8000 -w2 /opt/graphite/webapp/graphite/settings.py  > ~/logs/kamon/graphite.out 2>&1 &
nohup /usr/bin/node ~/src/statsd/stats.js ~/src/statsd/config.js  > ~/logs/kamon/graphite.out 2>&1 &
/etc/init.d/elasticsearch start
cd ~/src/dashboards
nohup /usr/bin/node dashboard-loader.js system-metrics.json  welcome.json   > ~/logs/kamon/grafana.out 2>&1 &
```
#### 2.停止脚本 stop_stats.sh
```
#!/bin/sh

pkill carbon
pkill gunicorn_django
pkill statsd
/etc/init.d/elasticsearch stop
kill $(ps aux | grep 'node dashboard-loader.js' | awk '{print $2}')
```
