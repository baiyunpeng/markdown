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

部署War（tomcat）
``` shell
vim /etc/tomcat7/server.xml 
```
修改server.xml文件，在Host结点下添加Context：
``` xml
<Context path="" docBase="/usr/local/disconf/build/war"></Context>
```

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
>注意：访问的html路径应该具有访问权限，可以在在nginx.conf文件第一行中更改用户，或者更改对应的HTML文件的权限。静态配置成功后，可以通过查看tomcat的日志和nginx日志文件来查看访问的记录，静态文件由nginx直接处理，而动态文件则是tomcat来处理的。

###### 能不安装nginx吗？
这是我刚开始在官方讨论群提的问题，得到的答案是不能，提到了什么动静分离，于是百度了解了下，对nginx在这里扮演的角色有了一个了解，知道他做了什么，才能知道他是否必须。了解了之后，就会知道，这里应该有多中方式实现不安装nginx，我实现了一种如下图所示，其他方式可以百度springMVC关于静态文件的处理方式，第一张截图就是我在eclipse中用tomcat运行的结果。这个能方便开发，正式环境建议还是按官方设计的方式使用，nginx对静态文件的处理要比tomcat快不少。
  
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

##### 在spring boot中使用 
 创建maven工程，添加对应的Maven库

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.egoo</groupId>
    <artifactId>demo</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <java.version>1.7</java.version>
    </properties>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.3.2.RELEASE</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>com.baidu.disconf</groupId>
            <artifactId>disconf-client</artifactId>
            <version>2.6.31</version>
        </dependency>

        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>2.1.0</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-logging</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-log4j</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

##### 添加对应的属性文件/resources

disconf.properties
``` profile
disconf.enable.remote.conf=true

disconf.conf_server_host=192.168.1.169:8080

disconf.version=V2

disconf.app=FreeLink

disconf.env=local

disconf.ignore=

disconf.conf_server_url_retry_sleep_seconds=1

disconf.user_define_download_dir=./disconf/download
```

disconf.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>

<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-3.1.xsd
        http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop-3.0.xsd">

    <aop:aspectj-autoproxy proxy-target-class="true"/>

<!-- 使用disconf必须添加以下配置 -->
<bean id="disconfMgrBean" class="com.baidu.disconf.client.DisconfMgrBean"
destroy-method="destroy">
        <property name="scanPackage" value="com.egoo.config"/>
    </bean>
    <bean id="disconfMgrBean2" class="com.baidu.disconf.client.DisconfMgrBeanSecond"
init-method="init" destroy-method="destroy">
    </bean>

</beans>
```

##### 基于XML配置文件实现
##### 配置管理

登陆disconf

默认用户名 admin admin

![enter description here][1]

新建一个APP

![enter description here][2]

新建一个配置文件

![enter description here][3]

选择相应的app一般本版号，上传配置文件

![enter description here][4]

添加完成后点击相应的环境查看

![enter description here][5]

##### 服务端配置

添加如下xml
``` xml
 <!--disconf 托管文件 配置更改会自动reload-->
    <bean id="configproperties_disconf" class="com.baidu.disconf.client.addons.properties.ReloadablePropertiesFactoryBean">
        <property name="locations">
            <list>
                <value>classpath:/sharding.xml</value>
            </list>
        </property>
    </bean>
	
	 <bean id="disconfPropertyConfigurer"
          class="com.baidu.disconf.client.addons.properties.ReloadingPropertyPlaceholderConfigurer">
        <property name="ignoreResourceNotFound" value="true" />
        <property name="ignoreUnresolvablePlaceholders" value="true" />
        <property name="propertiesArray">
            <list>
                <ref bean="configproperties_disconf"/>
            </list>
        </property>
    </bean>
```

重启服务配置文件已将添加到本地了

> 支持任意类型配置文件(.properties文件可支持自动注入,非.properties文件则只是简单托管)





  [1]: ./images/1500982025780.jpg
  [2]: ./images/1500982064075.jpg
  [3]: ./images/1500982133420.jpg
  [4]: ./images/1500982175389.jpg
  [5]: ./images/1500982221824.jpg