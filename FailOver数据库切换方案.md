---
title: FailOver数据库切换方案
tags: 数据库,切换,FailOver
grammar_cjkRuby: true
---


### 设计目的

为防止系统因数据库出现问题导致整体服务不可用，能够快速进行数据库切换，做到秒级数据库切换，能够将数据库硬件软件等故障的风险降到最低。
	
### 设计思路

1. 将failOver数据源通过disconf分发给不同的业务系统
2. 通过failOver数据源配置将数据源配置进数据源中
3. 通过调整数据源的是否在failOver状态进行切库
4. 进行数据库恢复，成功后在通过api切回正常的库

### 方案优缺点

#### 优点：
1. 只需要disconf管理failOver数据库配置文件或者放在项目配置文件中即可
2. 通过更改数据源切换jar包，不需要对现有代码进行任何更改可以进行基本的数据库切换。
#### 缺点
1. 需要更改项目中的配置文件。

### 关联问题解决方案
1. 数据同步问题
>可以对failOver库进行实时数据同步，来尽量减少数据库的同步问题以及数据不一致问题

2. oracle 序列自增问题问题
>如果是生成的订单问题，可以通过改变订单的某一位或者加一个数据库编号，或者分布式序列生成来规避订单号冲突问题，如果是使用的是数据库序列就需要使用分布式ID生成方案了，需要对代码层面进行更改

#### 更改方案
1.	使用分布式id生成器（改动较大）
2.	使用序列+数据库编码（改动较小）
3. 分库事物问题
	暂时没有好的解决方案。
4. 数据最终一致性
	通过同步数据，数据库后，将数据导入到主库中。

### 详细设计步骤

1. disconf 管理配置文件
>将failOver等配置文件交给disconf配置项进行统一管理，进行分布式配置文件管理。

2. 服务加载数据源配置
>将正常的数据源和failOver数据源统一在服务启动中加载经服务的数据源连接池中，系统默认使用普通的数据源，如果系统出现问题，通过手工或者自动将数据源切换到failOver数据源中
		
3. 数据源切换
>通报系统的一个API可以将数据源从正常的数据源切换到failOver数据源，也可以将failOver数据源切换到正式的数据源，都可以通过每个服务的API来完成。

4. 数据源自动切换与恢复
>因系统都是由mybatis来实现的，使用mybatis拦截器实现sql异常监听器，
自动切换： 拦截sql异常，如果出现连接超时的异常过多，则通过接口切换到failOver数据库。
自动恢复：线程每隔几秒钟进行一次select * from dual 查询，如果几次查询都正常则判定异常恢复，通过接口将数据源切换到正常的数据库。

5. 消息数据总线
>有一个统一的管理平台来手动触发每个服务是否切换failOver数据源，以及切换数据源后的通知，通过mq每个服务都监听同一个主题，统一成一个数据总线，每一个服务的key是不一样的，每个服务接收到是切换数据源通知后，判断是否是自己的key如果不是则忽略，如果是则切换数据源，并向统一管理平台发送切换成功的消息，如果是系统自发的切换，也通过mq向系统发送切换数据源的通知。

6. 熔断器设计
>  系统监控数据库的使用情况，如果检测到数据库的可用率过低，（例如100次操作有20次超时）则判断数据库出现问题，符合自动切换规则，进行自动failOver数据源切换，切换后向数据总线发送消息。

7. 各业务系统的订单生成
>系统提供一个接口，判断当前的数据源连接是否是正常的数据源，如果是则使用一套生成规则，如果是failOver，则使用另一套规则（需要业务系统进行代码更改）

8. 使用微信客户端
>可以使用微信客户端作为控制中心，在微信端可以实时查看各业务系统的是否在failOver库中，以及可以进行实时手工进行数据库切换操作（有安全风险）。

### 系统架构图

![enter description here][1]



  ### 系统提供接口
1.  远程操作切换数据源接口

2. 实时获取当前数据源是否是failOver库

### 项目中使用

1.  引入jar包 base-common-database
``` xml
<dependency>
    <groupId>com.reapal</groupId>
    <artifactId>base-common-database</artifactId>
    <version>2.0.0-SNAPSHOT</version>
</dependency>
```

2. 项目中加入failOver数据源配置文件

	failoverresources.properties
	![enter description here][2]

3. 修改spring配置文件
	* 将failoverresources.properties 加入属性管理器
	``` xml
	 <bean id="propertyConfigurer" class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
        <property name="locations">
            <list>
                <value>classpath:resources.properties</value>
                <value>classpath:dubbo.properties</value>
                <value>classpath:failoverresources.properties</value>
            </list>
        </property>
    </bean>
	```
	* 增加failOver 数据源
		>增加failOver 的读写数据源

	```xml
	 <!-- spring配置多数据源开始 -->
    <bean id="parentDataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource"/>
    <!-- 以下是配置各个数据库的特性 -->
    <!-- 数据库 -->
    <bean id="writeDataSource" class="com.alibaba.druid.pool.DruidDataSource" init-method="init" destroy-method="close">
        <property name="url" value="${connection.url}"/>
        <property name="username" value="${connection.username}"/>
        <property name="password" value="${connection.password}"/>

        <!-- 配置初始化大小、最小、最大 -->
        <property name="initialSize" value="1"/>
        <property name="minIdle" value="1"/>
        <property name="maxActive" value="20"/>

        <!-- 配置获取连接等待超时的时间 -->
        <property name="maxWait" value="60000"/>

        <!-- 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒 -->
        <property name="timeBetweenEvictionRunsMillis" value="60000"/>

        <!-- 配置一个连接在池中最小生存的时间，单位是毫秒 -->
        <property name="minEvictableIdleTimeMillis" value="300000"/>

        <!-- <property name="validationQuery" value="SELECT 'x'" />-->
        <property name="testWhileIdle" value="true"/>
        <property name="testOnBorrow" value="false"/>
        <property name="testOnReturn" value="false"/>

        <!-- 打开PSCache，并且指定每个连接上PSCache的大小 -->
        <property name="poolPreparedStatements" value="true"/>
        <property name="maxPoolPreparedStatementPerConnectionSize" value="20"/>

        <!-- 配置监控统计拦截的filters，去掉后监控界面sql无法统计 -->
        <!-- <property name="filters" value="stat" />-->

        <property name="filters" value="config"/>
        <property name="connectionProperties" value="config.decrypt=true"/>
    </bean>
    <bean id="readDataSource" class="com.alibaba.druid.pool.DruidDataSource" init-method="init" destroy-method="close">
        <property name="url" value="${connection2.url}"/>
        <property name="username" value="${connection2.username}"/>
        <property name="password" value="${connection2.password}"/>

        <!-- 配置初始化大小、最小、最大 -->
        <property name="initialSize" value="1"/>
        <property name="minIdle" value="1"/>
        <property name="maxActive" value="20"/>

        <!-- 配置获取连接等待超时的时间 -->
        <property name="maxWait" value="60000"/>

        <!-- 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒 -->
        <property name="timeBetweenEvictionRunsMillis" value="60000"/>

        <!-- 配置一个连接在池中最小生存的时间，单位是毫秒 -->
        <property name="minEvictableIdleTimeMillis" value="300000"/>

        <!-- <property name="validationQuery" value="SELECT count(*)" />-->
        <property name="testWhileIdle" value="true"/>
        <property name="testOnBorrow" value="false"/>
        <property name="testOnReturn" value="false"/>

        <!-- 打开PSCache，并且指定每个连接上PSCache的大小 -->
        <property name="poolPreparedStatements" value="true"/>
        <property name="maxPoolPreparedStatementPerConnectionSize" value="20"/>

        <!-- 配置监控统计拦截的filters，去掉后监控界面sql无法统计 -->
        <!--<property name="filters" value="stat" />-->

        <property name="filters" value="config"/>
        <property name="connectionProperties" value="config.decrypt=true"/>
    </bean>


    <bean id="failOverWriteDataSource" class="com.alibaba.druid.pool.DruidDataSource" init-method="init"
          destroy-method="close">
        <property name="url" value="${fialoverconnection.url}"/>
        <property name="username" value="${fialoverconnection.username}"/>
        <property name="password" value="${fialoverconnection.password}"/>

        <!-- 配置初始化大小、最小、最大 -->
        <property name="initialSize" value="1"/>
        <property name="minIdle" value="1"/>
        <property name="maxActive" value="20"/>

        <!-- 配置获取连接等待超时的时间 -->
        <property name="maxWait" value="60000"/>

        <!-- 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒 -->
        <property name="timeBetweenEvictionRunsMillis" value="60000"/>

        <!-- 配置一个连接在池中最小生存的时间，单位是毫秒 -->
        <property name="minEvictableIdleTimeMillis" value="300000"/>

        <!-- <property name="validationQuery" value="SELECT 'x'" />-->
        <property name="testWhileIdle" value="true"/>
        <property name="testOnBorrow" value="false"/>
        <property name="testOnReturn" value="false"/>

        <!-- 打开PSCache，并且指定每个连接上PSCache的大小 -->
        <property name="poolPreparedStatements" value="true"/>
        <property name="maxPoolPreparedStatementPerConnectionSize" value="20"/>

        <!-- 配置监控统计拦截的filters，去掉后监控界面sql无法统计 -->
        <!-- <property name="filters" value="stat" />-->

        <property name="filters" value="config"/>
        <property name="connectionProperties" value="config.decrypt=true"/>
    </bean>
    <bean id="failOverReadDataSource" class="com.alibaba.druid.pool.DruidDataSource" init-method="init"
          destroy-method="close">
        <property name="url" value="${fialoverconnection2.url}"/>
        <property name="username" value="${fialoverconnection2.username}"/>
        <property name="password" value="${fialoverconnection2.password}"/>

        <!-- 配置初始化大小、最小、最大 -->
        <property name="initialSize" value="1"/>
        <property name="minIdle" value="1"/>
        <property name="maxActive" value="20"/>

        <!-- 配置获取连接等待超时的时间 -->
        <property name="maxWait" value="60000"/>

        <!-- 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒 -->
        <property name="timeBetweenEvictionRunsMillis" value="60000"/>

        <!-- 配置一个连接在池中最小生存的时间，单位是毫秒 -->
        <property name="minEvictableIdleTimeMillis" value="300000"/>

        <!-- <property name="validationQuery" value="SELECT count(*)" />-->
        <property name="testWhileIdle" value="true"/>
        <property name="testOnBorrow" value="false"/>
        <property name="testOnReturn" value="false"/>

        <!-- 打开PSCache，并且指定每个连接上PSCache的大小 -->
        <property name="poolPreparedStatements" value="true"/>
        <property name="maxPoolPreparedStatementPerConnectionSize" value="20"/>

        <!-- 配置监控统计拦截的filters，去掉后监控界面sql无法统计 -->
        <!--<property name="filters" value="stat" />-->

        <property name="filters" value="config"/>
        <property name="connectionProperties" value="config.decrypt=true"/>
    </bean>


    <bean id="dataSource" class="com.reapal.common.dynamicds.DynamicDataSource">
        <property name="targetDataSources">
            <map key-type="java.lang.String">
                <entry value-ref="readDataSource" key="READ"></entry>
                <entry value-ref="writeDataSource" key="WRITE"></entry>
                <entry value-ref="failOverReadDataSource" key="FIALOVERREAD"></entry>
                <entry value-ref="failOverWriteDataSource" key="FAILOVERWRITE"></entry>
            </map>
        </property>
        <property name="defaultTargetDataSource" ref="writeDataSource"></property>
    </bean>
	```
	* 增加failOver切换服务
``` xml
	<bean id="failOverService" class="com.reapal.failover.service.impl.FailOverServiceImpl">
        <property name="serviceName" value="base-sms-provider"/>
    </bean>
    <dubbo:service interface="com.reapal.failover.service.FailOverService" ref="failOverService"
                   version="${dubbo.version}"/>
```

### 管理界面
![enter description here][3]
 通过界面对服务进行数据库failOver切换

### 代码片段
``` java
 	private static final ThreadLocal<String> contextHolder = new ThreadLocal<String>(); // 线程本地环境


    private static boolean fialOver = false;

    // 设置数据源类型
    public static void setDataSourceType(String dataSourceType) {
        contextHolder.set(dataSourceType);
    }

    /**
     * 是否启用failOver数据源
     *
     * @param failOverStatus true: 启用failOver 数据源
     *                       false：使用普通数据源
     */
    public static synchronized void failOver(boolean failOverStatus) {
        fialOver = failOverStatus;
    }

    /**
     * 获取当前是否是failOver库
     * @return
     */
    public static boolean getIsFailOver() {
        return fialOver;
    }


    // 获取数据源类型
    public static String getDataSourceType() {
        return getDBType();
    }


    //获取数据源类型
    public static String getDBType() {
        String dbType = contextHolder.get();
        if (dbType == null) {
            dbType = DataSourceConst.WRITE;
        }
        if (fialOver) {
            if (dbType.equals(DataSourceConst.READ)) {
                dbType = FailOverConst.FIALOVERREAD;
            } else if (dbType.equals(DataSourceConst.WRITE)) {
                dbType = FailOverConst.FIALOVERWRITE;
            }
        }
        return dbType;
    }

    // 清除数据源类型
    public static void clearDataSourceType() {
        contextHolder.remove();
    }
```


  [1]: ./images/1504081449741.jpg
  [2]: ./images/1504081726553.jpg
  [3]: ./images/1504082538739.jpg