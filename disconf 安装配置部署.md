---
title: disconf 安装配置部署
tags: disconf,安装,配置
grammar_cjkRuby: true
---

### disconf 简述

在一个分布式环境中，同类型的服务往往会部署很多实例。这些实例使用了一些配置，为了更好地维护这些配置就产生了配置管理服务。通过这个服务可以轻松地管理成千上百个服务实例的配置问题。

王阿晶提出了基于zooKeeper的配置信息存储方案的设计与实现[1], 它将所有配置存储在zookeeper上，这会导致配置的管理不那么方便，而且他们没有相关的源码实现。淘宝的diamond[2]是淘宝内部使用的一个管理持久配置的系统，它具有完整的开源源码实现，它的特点是简单、可靠、易用，淘宝内部绝大多数系统的配置都采用diamond来进行统一管理。他将所有配置文件里的配置打散化进行存储，只支持KV结构，并且配置更新的推送是非实时的。百度内部的BJF配置中心服务[3]采用了类似淘宝diamond的实现，也是配置打散化、只支持KV和非实时推送。

同构系统是市场的主流，特别地，在业界大量使用部署虚拟化（如JPAAS系统，SAE，BAE）的情况下，同一个系统使用同一个部署包的情景会越来越多。但是，异构系统也有一定的存在意义，譬如，对于“拉模式”的多个下游实例，同一时间点只能只有一个下游实例在运行。在这种情景下，就存在多台实例机器有“主备机”模式的问题。目前国内并没有很明显的解决方案来统一解决此问题。

### disconf 安装部署

#### 下载安装disconf

##### 安装依赖软件

* 安装 Mysql
* 安装 Tomcat
* 安装 Nginx
* 安装 zookeeeper
* 安装 Redis

##### 下载安装包
``` shell
 # 下载安装包
 wget https://codeload.github.com/knightliao/disconf/zip/master
 
 # 重命名 disconf文件
 mv master disconf-master.zip
 
 #解压文件
 unzip disconf-master.zip
 
 #创建配置文件目录
 mkdir online-resources
 
 #创建war 目录
 mkdir war
 ```
 
 ##### 配置文件整理
 
 将disconf-web/profile/rd目录下的文件copy到online-resources目录下
 
 文件有
 * jdbc-mysql.properties (数据库配置)
 * redis-config.properties (Redis配置)
 * zoo.properties (Zookeeper配置)
 * application.properties (应用配置）

>注意，记得执行将application-demo.properties复制成application.properties：

```shell
cp application-demo.properties application.properties 
```

##### 构建disconf

``` shell
ONLINE_CONFIG_PATH=/home/dev1/Downloads/disconf/build/online-resources
WAR_ROOT_PATH=/home/dev1/Downloads/disconf/build/war
export ONLINE_CONFIG_PATH
export WAR_ROOT_PATH
cd disconf-web
sh deploy/deploy.sh
```

编译完成后,会在 /home/dev1/Downloads/disconf/build/war 生成以下结果：
* disconf-web.war  
* html  
* META-INF  
* WEB-INF

##### 部署
Disconf代码设计采用了静动分离设计，整个的代码的前后端完全分离。

###### 初始化数据库

对应的数据库脚本都在disconf-web/sql目录下，依次执行对应的sql语句就可以了

``` sql
mysql -u username -p password < 0-init_table.sql
mysql -u username -p password -Ddisconf  < 1-init_data.sql
mysql -u username -p password -Ddisconf < 201512/20151225.sql
```

用户数据：可以参考 sql/readme.md 来进行数据库的初始化。里面默认有6个用户

> 如果想自己设置初始化的用户名信息，可以参考代码来自己生成用户： 
src/main/Java/com/baidu/disconf/web/tools/UserCreateTools.java

##### 动静部署
Disconf代码采用了动静分离设计，

###### disconf 默认用户列表

name|pwd
--------|----
admin|admin
testUser1|MhxzKhl9209
testUser2|MhxzKhl167
testUser3|MhxzKhl783
testUser4|MhxzKhl8758
testUser5|MhxzKhl112

如果想自己设置初始化的用户名信息，可以参考代码来自己生成用户：
``` java
src/main/java/com/baidu/disconf/web/tools/UserCreateTools.java
```

部署War（tomcat）
``` shell
vim /etc/tomcat7/server.xml 
```
修改server.xml文件，在Host结点下添加Context：
``` xml
<Context path="" docBase="/usr/local/disconf/build/war"></Context>
```
###### 配置文件修改
修改 online-resources 目录下的各个配置文件
application.properties
使用默认的就可以不需要更改

jdbc-mysql.properties
修改数据库地址

redis-config.properties
修改redis链接地址
redis 因为使用双点的，就算单点也需要配置两个
``` profile
redis.group1.retry.times=2

redis.group1.client1.name=BeidouRedis1
redis.group1.client1.host=10.168.1.1
redis.group1.client1.port=6379
redis.group1.client1.timeout=5000
redis.group1.client1.password=foobared

redis.group1.client2.name=BeidouRedis2
redis.group1.client2.host=10.168.1.1
redis.group1.client2.port=6379
redis.group1.client2.timeout=5000
redis.group1.client2.password=foobared

redis.evictor.delayCheckSeconds=300
redis.evictor.checkPeriodSeconds=30
redis.evictor.failedTimesToBeTickOut=6
```
zoo.properties
配置zookeeper链接地址


并在对应的war目录下创建tmp目录，启动tomcat服务器。

##### 部署前端（nginx）

修改 nginx.conf，在HTTP标签中添加以下内容：
``` nginx
upstream disconf {
    server 127.0.0.1:8080;
}
server {
    listen   8081;
    server_name localhost;
    access_log /home/work/var/logs/disconf/access.log;
    error_log /home/work/var/logs/disconf/error.log;
    location / {
        root /home/work/dsp/disconf-rd/war/html;
        if ($query_string) {
            expires max;
        }
    }
    location ~ ^/(api|export) {
        proxy_pass_header Server;
        proxy_set_header Host $http_host;
        proxy_redirect off;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_pass http://disconf;
    }
}
```
这里只是部分配置 
完整的配置如下
``` nginx
user  root;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    gzip  on;
    gzip_http_version 1.0;
    gzip_disable "MSIE [1-6].";
    gzip_types text/plain application/x-javascript text/css text/javascript application/x-httpd-php image/jpeg image/gif image/png;

upstream disconf {
    server 10.10.10.10:8015;
}

server {
    listen   8081;
    server_name disconftest.com;
    access_log /home/work/dsp/access.log;
    error_log /home/work/dsp/error.log;

    location / {
        root /home/docker/garfield/war/html;
        if ($query_string) {
            expires max;
        }
    }

    location ~ ^/(api|export) {
        proxy_pass_header Server;
        proxy_set_header Host $http_host;
        proxy_redirect off;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_pass http://disconf;
    }
}
}
```

这里我把server name改成了disconftest.com,跟application.properties里面的配置保持一样即可,

注意:

1.须保证日志文件的写入有权限哦,日志文件夹提前建好,否则会报错

2.开头是运行权限,这边因为没有用root运行会导致页面持续被拦截,报403,有遇到的人注意下

服务管理端的部署到这里就结束了,启动zookeeper,redis,tomcat,mysql和nginx就可以访问页面了,输入
```http
http://10.10.10.10:8081/
```
![enter description here][1]

>注意：访问的html路径应该具有访问权限，可以在在nginx.conf文件第一行中更改用户，或者更改对应的HTML文件的权限。静态配置成功后，可以通过查看tomcat的日志和nginx日志文件来查看访问的记录，静态文件由nginx直接处理，而动态文件则是tomcat来处理的。

###### 能不安装nginx吗？
这是我刚开始在官方讨论群提的问题，得到的答案是不能，提到了什么动静分离，于是百度了解了下，对nginx在这里扮演的角色有了一个了解，知道他做了什么，才能知道他是否必须。了解了之后，就会知道，这里应该有多中方式实现不安装nginx，我实现了一种如下图所示，其他方式可以百度springMVC关于静态文件的处理方式，第一张截图就是我在eclipse中用tomcat运行的结果。这个能方便开发，正式环境建议还是按官方设计的方式使用，nginx对静态文件的处理要比tomcat快不少。
  
##### 测试环境disconf 地址
>开发环境配置端地址
``` http
http://10.168.16.113:8081
```

>开发环境接口端地址
``` http
http://10.168.16.113:8080
```
##### 业务功能说明

-支持用户登录/登出
-浏览配置
    按 APP/版本/环境 选择
修改配置
    修改配置
    修改配置项
    修改配置文件
新建配置
    新建配置项
    新建配置文件
    新建APP


#### Client使用

##### 引入POM
 创建maven工程，添加对应的Maven库

``` xml
  		<dependency>
            <groupId>com.baidu.disconf</groupId>
            <artifactId>disconf-client</artifactId>
            <version>2.6.36</version>
        </dependency>
```

##### spring disconf 配置
##### 在spring 中加入以下代码

``` xml
  		<!-- 使用disconf必须添加以下配置 -->
		<bean id="disconfMgrBean" class="com.baidu.disconf.client.DisconfMgrBean"
		destroy-method="destroy">
				<property name="scanPackage" value="com.egoo.config"/>
			</bean>
			<bean id="disconfMgrBean2" class="com.baidu.disconf.client.DisconfMgrBeanSecond"
		init-method="init" destroy-method="destroy">
			</bean>
```


##### 添加对应的属性文件/resources

disconf.properties
``` profile
#是否启用远程配置
disconf.enable.remote.conf=true
#disconf服务器地址
disconf.conf_server_host=10.168.16.113:8080
#disconf 版本号 disconf 推荐使用 x_x_x_x 版本控制
disconf.version=1_0_0_0
 # APP 请采用 产品线_服务名 格式
disconf.app=basesms
#disconf 环境
disconf.env=qa
# 忽略哪些分布式配置，用逗号分隔
disconf.ignore=
# 获取远程配置 重试次数，默认是3次
disconf.conf_server_url_retry_times=1
#获取远程配置 重试时休眠时间，默认是5秒
disconf.conf_server_url_retry_sleep_seconds=1
#disconf 现在文件目录 默认 ./disconf/download
disconf.user_define_download_dir=./disconf/download
# 下载的文件会被迁移到classpath根路径下，强烈建议将此选项置为 true(默认是true)
disconf.enable_local_download_dir_in_class_path=true
```


##### 基于XML配置文件实现
##### 配置管理

登陆disconf




![enter description here][2]

新建一个APP

![enter description here][3]

新建一个配置文件

![enter description here][4]

选择相应的app一般本版号，上传配置文件

![enter description here][5]

添加完成后点击相应的环境查看

![enter description here][6]

##### spring配置需要加载的配置文件

添加如下xml
``` xml
 <!--disconf 托管文件 配置更改会自动reload-->
    <bean id="configproperties_disconf" class="com.baidu.disconf.client.addons.properties.ReloadablePropertiesFactoryBean">
        <property name="locations">
            <list>
				<!-- 需要下载的文件路径及配置文件名，可以多个 -->
                <value>classpath:/sharding.xml</value>
            </list>
        </property>
    </bean>
	
	 <!--disconf 属性文件配置管理-->
	 <bean id="disconfPropertyConfigurer"
 class="com.baidu.disconf.client.addons.properties.ReloadingPropertyPlaceholderConfigurer">
        <property name="ignoreResourceNotFound" value="true" />
        <property name="ignoreUnresolvablePlaceholders" value="true" />
        <property name="propertiesArray">
            <list>
			<!-- 需要加载的配置 -->
                <ref bean="configproperties_disconf"/>
            </list>
        </property>
    </bean>
```

重启服务配置文件已将添加到本地了

> 支持任意类型配置文件(.properties文件可支持自动注入,非.properties文件则只是简单托管)

##### 启用服务
> 启动服务 配置文件已加载到class下

![enter description here][7]

#### 注解使用方式
>不推荐使用 具有代码入侵

``` java
@Service
@Scope("singleton")
@DisconfFile(filename = "redis.properties")
public class SimpleConfig {

    // 代表连接地址
    private String host;

    // 代表连接port
    private String port;

    /**
     * 地址
     *
     * @return
     */
    @DisconfFileItem(name = "host", associateField = "host")
    public String getHost() {
        return host;
    }

    public void setHost(String host) {
        this.host = host;
    }

    /**
     * 端口
     *
     * @return
     */
    @DisconfFileItem(name = "port", associateField = "port")
    public String getPort() {
        return port;
    }

    public void setPort(String port) {
        this.port = port;
    }
}
```


  [1]: ./images/1500984558848.jpg
  [2]: ./images/1500982025780.jpg
  [3]: ./images/1500982064075.jpg
  [4]: ./images/1500982133420.jpg
  [5]: ./images/1500982175389.jpg
  [6]: ./images/1500982221824.jpg
  [7]: ./images/1501230249925.jpg