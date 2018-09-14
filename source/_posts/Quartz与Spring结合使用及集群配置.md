---
title: Quartz与Spring结合使用及集群配置
date: 2018-09-15 00:44:35
categories:
- Spring
tags:
- Quartz
- 调度
- 集群
copyright:
top:
type:
---

# Quartz与Spring结合使用及集群配置

本文的原文在我个人的[CSDN](https://blog.csdn.net/benjaminlee1/article/details/72993879)上。是在2017年的时候由于工作上面需要用到在集群环境中使用调度，采用了Quartz实现，当时进行了记录。不过现在我写博客主要是在个人站点上，所以就把之前的博客部分搬移过来了。
本文代码[github地址](https://github.com/lightnine/spring-quartz)

## quartz介绍

quartz是进行任务调度执行的框架，相对于Java中线程池调度以及Spring自带注解的调度方法，有以下几个有点：

1. 能够支持上千上万个调度任务的执行 
2. 任务调度方式较为灵活 
3. 支持集群

## 运行环境

1. quartz版本：2.2.1，这是quartz最新的版本，其他版本可能跟此版本有一定差异
2. 操作系统centos7,我在vmware中搭建了三台独立的虚拟机
3. JDK8
4. 数据库：mysql 5.7.18
5. tomcat8

## 工程目录结构

工程目录结构如下，采用Idea IDE
{% asset_img 工程目录结构.png 工程目录结构%}

文件说明:
resources下是各种配置资源，create-schema.sql用于创建任务表，tables_mysql_innodb.sql用于创建quartz集群运行需要的表，共有十一张表。quartz_properties是quartz的配置

## 主要配置文件内容

配置文件中都有每个配置项详细的说明，我这里就只给出具体的配置内容
spring配置文件有两个，分别为applicationContext.xml和spring-mvc.xml
**applicationContext.xml文件如下**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
    http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-3.0.xsd
    http://www.springframework.org/schema/context
    http://www.springframework.org/schema/context/spring-context-3.0.xsd">

    <context:component-scan base-package="com.ll.*"/>

    <!-- 启用@Aspect支持 -->
    <aop:aspectj-autoproxy/>
    <!-- 加载数据库配置 -->
    <context:property-placeholder location="classpath:jdbc.properties"/>
    <!-- 数据源 -->
    <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource"
          destroy-method="close">
        <property name="driverClassName">
            <value>${jdbc.driver}</value>
        </property>
        <property name="url">
            <value>${jdbc.url}</value>
        </property>
        <property name="username">
            <value>${jdbc.username}</value>
        </property>
        <property name="password">
            <value>${jdbc.password}</value>
        </property>
        <property name="initialSize">
            <value>${jdbc.initialSize}</value>
        </property>
        <property name="maxActive">
            <value>${jdbc.maxActive}</value>
        </property>
        <property name="maxIdle">
            <value>${jdbc.minIdle}</value>
        </property>
        <property name="maxWait">
            <value>${jdbc.maxWait}</value>
        </property>
    </bean>

    <bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
        <property name="dataSource">
            <ref bean="dataSource"/>
        </property>
        <property name="fetchSize">
            <value>${jdbc.jdbcTemplate.fetchSize}</value>
        </property>
    </bean>
    <!-- 访问数据库方式 -->
    <bean id="jdbcDao" class="com.dexcoder.dal.spring.JdbcDaoImpl">
        <property name="jdbcTemplate" ref="jdbcTemplate"/>
    </bean>

    <!-- 配置quartz调度器 -->
    <bean id="scheduler" class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
        <property name="dataSource" ref="dataSource" />
        <!--可选，QuartzScheduler 启动时更新己存在的Job，这样就不用每次修改targetObject后删除qrtz_job_details表对应记录了 -->
        <property name="overwriteExistingJobs" value="true" />
        <!--必须的，QuartzScheduler 延时启动，应用启动完后 QuartzScheduler 再启动 -->
        <property name="startupDelay" value="3" />
        <!-- 设置自动启动 -->
        <property name="autoStartup" value="true" />
        <!-- 把spring上下 文以key/value的方式存放在了quartz的上下文中了 -->
        <property name="applicationContextSchedulerContextKey" value="applicationContext" />
        <!-- scheduler的名称 -->
        <property name="schedulerName" value="ClusterScheduler" />
        <property name="configLocation" value="classpath:quartz.properties" />
<!--         <property name="quartzProperties"> -->
<!--             <props> -->
<!--                 <prop key="org.quartz.scheduler.instanceName">ClusterScheduler</prop> -->
<!--                 <prop key="org.quartz.scheduler.instanceId">AUTO</prop> -->
<!--                 线程池配置 -->
<!--                 <prop key="org.quartz.threadPool.class">org.quartz.simpl.SimpleThreadPool</prop> -->
<!--                 <prop key="org.quartz.threadPool.threadCount">20</prop> -->
<!--                 <prop key="org.quartz.threadPool.threadPriority">5</prop> -->
<!--                 JobStore 配置 -->
<!--                 <prop key="org.quartz.jobStore.class">org.quartz.impl.jdbcjobstore.JobStoreTX</prop> -->

<!--                 集群配置 -->
<!--                 <prop key="org.quartz.jobStore.isClustered">true</prop> -->
<!--                 <prop key="org.quartz.jobStore.clusterCheckinInterval">15000</prop> -->
<!--                 <prop key="org.quartz.jobStore.maxMisfiresToHandleAtATime">1</prop> -->

<!--                 <prop key="org.quartz.jobStore.misfireThreshold">120000</prop> -->

<!--                 <prop key="org.quartz.jobStore.tablePrefix">QRTZ_</prop> -->
<!--             </props> -->
<!--         </property> -->
    </bean>
</beans>
```

spring-mvc.xml内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
         http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context-3.0.xsd">

    <context:component-scan base-package="com.ll.*" />

    <bean
        class="org.springframework.web.servlet.view.ContentNegotiatingViewResolver">
        <property name="order" value="1" />
        <property name="ignoreAcceptHeader" value="true" />
        <property name="mediaTypes">
            <map>
                <entry key="json" value="application/json;charset=UTF-8" />
                <entry key="xml" value="application/xml;charset=UTF-8" />
                <entry key="rss" value="application/rss+xml;charset=UTF-8" />
            </map>
        </property>
        <property name="defaultViews">
            <list>
            </list>
        </property>
    </bean>

    <bean id="velocityConfig"
        class="org.springframework.web.servlet.view.velocity.VelocityConfigurer">
        <property name="resourceLoaderPath" value="/WEB-INF" />
        <property name="configLocation" value="classpath:velocity.properties" />
    </bean>

    <bean id="viewResolver"
        class="org.springframework.web.servlet.view.velocity.VelocityViewResolver">
        <!-- <property name="order" value="2"/> -->
        <property name="suffix" value=".vm" />
        <property name="prefix" value="/"/>
        <!-- <property name="exposeSpringMacroHelpers" value="true"/> -->
        <!-- <property name="requestContextAttribute" value="rc"/> -->
        <property name="contentType" value="text/html;charset=UTF-8" />
    </bean>

</beans>
```

Quartz配置文件为quartz.properties,内容如下：

```
#配置参数详解，查看http://haiziwoainixx.iteye.com/blog/1838055
#============================================================================
# Configure Main Scheduler Properties  
#============================================================================
# 可为任何值,用在jdbc jobstrore中来唯一标识实例，但是在所有集群中必须相同
org.quartz.scheduler.instanceName: MyClusteredScheduler
#AUTO即可，基于主机名和时间戳来产生实例ID
#集群中的每一个实例都必须有一个唯一的"instance id",应该有相同的"scheduler instance name"
org.quartz.scheduler.instanceId: AUTO
#禁用quartz软件更新
org.quartz.scheduler.skipUpdateCheck: true

#============================================================================
# Configure ThreadPool  执行任务线程池配置
#============================================================================
#线程池类型，执行任务的线程
org.quartz.threadPool.class: org.quartz.simpl.SimpleThreadPool
#线程数量
org.quartz.threadPool.threadCount: 10
#线程优先级
org.quartz.threadPool.threadPriority: 5
org.quartz.threadPool.threadsInheritContextClassLoaderOfInitializingThread = true 

#============================================================================
# Configure JobStore  任务存储方式
#============================================================================

org.quartz.jobStore.misfireThreshold: 60000
#可以设置两种属性，存储在内存的RAMJobStore和存储在数据库的JobStoreSupport
#(包括JobStoreTX和JobStoreCMT两种实现JobStoreCMT是依赖于容器来进行事务的管理，而JobStoreTX是自己管理事务)
#这里的属性为 JobStoreTX，将任务持久化到数据中。
#因为集群中节点依赖于数据库来传播 Scheduler 实例的状态，你只能在使用 JDBC JobStore 时应用Quartz 集群。    
#这意味着你必须使用 JobStoreTX 或是 JobStoreCMT 作为 Job 存储；你不能在集群中使用 RAMJobStore。
org.quartz.jobStore.class=org.quartz.impl.jdbcjobstore.JobStoreTX

#JobStoreSupport 使用一个驱动代理来操作 trigger 和 job 的数据存储
org.quartz.jobStore.driverDelegateClass=org.quartz.impl.jdbcjobstore.StdJDBCDelegate
#若要设置为true，则将JobDataMaps中的值当作string
org.quartz.jobStore.useProperties=false
#对应下方的数据源配置，与spring结合不需要这个配置
#org.quartz.jobStore.dataSource=myDS
#org.quartz.jobStore.tablePrefix=QRTZ_

#你就告诉了Scheduler实例要它参与到一个集群当中。这一属性会贯穿于调度框架的始终，用于修改集群环境中操作的默认行为。
org.quartz.jobStore.isClustered=true
#属性定义了Scheduler实例检入到数据库中的频率(单位：毫秒)。默认值是 15000 (即15 秒)。
org.quartz.jobStore.clusterCheckinInterval = 20000

#这是 JobStore 能处理的错过触发的 Trigger 的最大数量。
#处理太多(超过两打) 很快会导致数据库表被锁定够长的时间，这样就妨碍了触发别的(还未错过触发) trigger 执行的性能。
org.quartz.jobStore.maxMisfiresToHandleAtATime = 1
org.quartz.jobStore.misfireThreshold = 120000
org.quartz.jobStore.txIsolationLevelSerializable = false

#============================================================================
# Other Example Delegates 其他的数据库驱动管理委托类
#============================================================================
#org.quartz.jobStore.driverDelegateClass=org.quartz.impl.jdbcjobstore.DB2v6Delegate
#org.quartz.jobStore.driverDelegateClass=org.quartz.impl.jdbcjobstore.DB2v7Delegate
#org.quartz.jobStore.driverDelegateClass=org.quartz.impl.jdbcjobstore.DriverDelegate
#org.quartz.jobStore.driverDelegateClass=org.quartz.impl.jdbcjobstore.HSQLDBDelegate
#org.quartz.jobStore.driverDelegateClass=org.quartz.impl.jdbcjobstore.MSSQLDelegate
#org.quartz.jobStore.driverDelegateClass=org.quartz.impl.jdbcjobstore.PointbaseDelegate
#org.quartz.jobStore.driverDelegateClass=org.quartz.impl.jdbcjobstore.PostgreSQLDelegate
#org.quartz.jobStore.driverDelegateClass=org.quartz.impl.jdbcjobstore.StdJDBCDelegate
#org.quartz.jobStore.driverDelegateClass=org.quartz.impl.jdbcjobstore.WebLogicDelegate
#org.quartz.jobStore.driverDelegateClass=org.quartz.impl.jdbcjobstore.oracle.OracleDelegate
#org.quartz.jobStore.driverDelegateClass=org.quartz.impl.jdbcjobstore.oracle.WebLogicOracleDelegate

#============================================================================
# Configure Datasources  数据源配置，与spring结合不需要这个配置
#============================================================================

#org.quartz.dataSource.myDS.driver: com.mysql.jdbc.Driver
#org.quartz.dataSource.myDS.URL: jdbc:mysql://localhost:3306/quartz
#org.quartz.dataSource.myDS.user: root
#org.quartz.dataSource.myDS.password: 123
#org.quartz.dataSource.myDS.maxConnections: 5
#org.quartz.dataSource.myDS.validationQuery: select 0

#============================================================================
# Configure Plugins
#============================================================================
#当事件的JVM终止后，在调度器上也将此事件终止
#the shutdown-hook plugin catches the event of the JVM terminating, and calls shutdown on the scheduler.
org.quartz.plugin.shutdownHook.class: org.quartz.plugins.management.ShutdownHookPlugin
org.quartz.plugin.shutdownHook.cleanShutdown: true

#记录trigger历史的日志插件
#org.quartz.plugin.triggHistory.class: org.quartz.plugins.history.LoggingJobHistoryPlugin
```

## 运行

当按照此项目配置完成之后，将项目打包成war包，然后在一台centos下启动项目，添加任务，之后再另一台centos下也运行此war包。为了简单起见，这里添加的任务仅仅是输出任务的详细信息，并没有其他动作。在两台机器的catalina.out中可以看到输出的结果，可以看到任务可以负载均衡的运行在不同的机器上。

> 如何动态的查看catalina.out中的内容，可以用tail -f来查看，能够实时看到文件的变化