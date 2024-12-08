---
title: 'James邮件服务器搭建'
date: 2024-12-08 11:28:09
tags: [James,邮件服务器]
published: true
hideInList: false
feature: 
isTop: false
---
## 下载

Apache James 3.x配置和2.x有较大区别，网上的文档大多基于2.x，所以我们也使用2.x版本。

```bash
wget http://mirrors.tuna.tsinghua.edu.cn/apache//james/server/james-binary-2.3.2.1.tar.gz
```

<!-- more -->

> 3.x启动的时候经常会报`'priority' init parameter is compulsory`异常，这里可能是因为配置文件里本应该是<priority>元素的地方配置成了<value>导致的。修改`conf/mailetcontainer.xml`文件中
>
> ```xml
> <mailet matcher="All" class="WithPriority">
> 	<value>8</value>
> </mailet>
> ```
>
> 为
>
> ```xml
> <mailet matcher="All" class="WithPriority">
> 	<value>8</value>
> 	<priority>8</priority>
> </mailet>
> ```

## 配置

然后解压之后运行`bin/run.sh`(需要给文件添加可执行权限 chmod +x *.sh)

首次运行之后会把`apps/james.sar`文件解压，然后修改解压后的文件里的配置`apps/james/SAR-INF/config.xml`即可。

### 1、修改postmaster

```xml
<config>
   <James>
      <postmaster>admin@steamfavor.com</postmaster>
       ...
    </James>
</config>
```

当有错误邮件时，会统一有该账户处理，类似其他邮件服务商提供的`catch-all`服务。

### 2、修改servername

```xml
<config>
   <James>
       <servernames autodetect="false" autodetectIP="false">
           <servername>steamfavor.com</servername>
       </servernames>
```

servername填申请的域名，同时需要关闭autodetect和autodetectIP

> 注册完域名之后，需要添加几个解析：
>
> A类：mail -> 邮件服务器IP
>
> A类：pop3 -> 邮件服务器IP
>
> A类：smtp -> 邮件服务器IP
>
> MX：@ -> mail.steamfavor.com

### 3、修改邮件存储为Mysql

打开下面的配置

```xml
<inboxRepository>
    <repository destinationURL="db://maildb/inbox/" type="MAIL"/>
</inboxRepository>
```

注释掉默认的文件保存配置

```xml
<inboxRepository>
    <repository destinationURL="file://var/mail/inboxes/" type="MAIL"/>
</inboxRepository>
```

打开数据库的配置并注释掉默认以文件保存的配置

```xml
<database-connections>
    <data-source name="maildb" class="org.apache.james.util.dbcp.JdbcDataSource">
        <driver>com.mysql.jdbc.Driver</driver>
        <dburl>jdbc:mysql://127.0.0.1:3306/caesar?autoReconnect=true&amp;characterEncoding=latin1&amp;useConfigs=maxPerformance</dburl>
        <user>root</user>
        <password>123456</password>
        <max>20</max>
    </data-source>
</database-connections>
```

### 4、修改dnsserver

配置邮件服务器的dnsserver

```xml
<dnsserver>
    <servers>
        <server>114.114.114.114</server>
        <server>8.8.8.8</server>
        <server>8.8.4.4</server>
    </servers>
    <autodiscover>false</autodiscover>
    <authoritative>false</authoritative>
    <maxcachesize>50000</maxcachesize>
</dnsserver>
```

### 5、开启远程管理

开启远程管理可以远程通过4555端口对用户数据进行修改，包括添加用户，删除用户，修改密码等。需要配置展示的名称和登陆的用户名和密码。

```xml
<remotemanager enabled="true">
    <port>4555</port>
    <handler>
        <helloName autodetect="false">steamfavor.com</helloName>
        <administrator_accounts>
            <account login="root" password="xh123456**"/>
        </administrator_accounts>
        <connectiontimeout> 60000 </connectiontimeout>
    </handler>
</remotemanager>
```

### 6、开启pop3服务

默认监听110端口，SSL端口995，配置服务器名称。

```xml
<pop3server enabled="true">
    <port>110</port>
    <handler>
        <helloName autodetect="false">steamfavor.com</helloName>
        <connectiontimeout>120000</connectiontimeout>
    </handler>
</pop3server>
```

### 7、开启smtp服务

默认监听25端口

```xml
<smtpserver enabled="true">
    <port>25</port>
    <handler>
        <helloName autodetect="true">myMailServer</helloName>
        <connectiontimeout>360000</connectiontimeout>
        <maxmessagesize>0</maxmessagesize>
    </handler>
</smtpserver>
```

配置完成后，重新启动james服务器。