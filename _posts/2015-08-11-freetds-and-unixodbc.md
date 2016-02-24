---
layout: post
title: "unixodbc and freetds"
description: "connect to SqlServer under linux with freetds"
category: "tools"
tags: [ "unixodbc", "freetds" ]
---

公司使用的SqlServer数据库，如果想在linux下写个perl/php程序来访问SQLServer, 比较麻烦, SQLServer好像没有Linux/Mac下的驱动
还好， 有个freetds unixodbc的好东东。 

## ubuntu 14.04

一. 安装unixodbc, freetds

{% highlight bash %}
sudo apt-get install freetds-bin freetds-common freetds-dev libct4 libsybdb5 \  
    unixodbc unixodbc-dev unixodbc-bin libodbc1 odbcinst1debian2
{% endhighlight %}

二. 准备配置文件(/etc/freetds/freetds.conf)

{% highlight properties %}
[cactus]
host = cactus
port = 1433
tds version = 8.0
{% endhighlight %}


三. 准备配置文件(/etc/odbcinst.ini)
{% highlight properties %}
[FreeTDS]
Description = FreeTDS Driver v0.91
Driver = /usr/lib/x86_64-linux-gnu/odbc/libtdsodbc.so
Setup = /usr/lib/x86_64-linux-gnu/odbc/libtdsS.so
fileusage=1
dontdlclose=1
UsageCount=1
Trace = yes
TraceFile = /tmp/odbc.log
{% endhighlight %}


四. 准备配置文件(/etc/odbc.ini)
{% highlight properties %}
[cactus]
Description = sqlserver@cactus
Driver = FreeTDS
Server = cactus
Port = 1433

# 这个是个坑爹的配置啊！！！！ 一定加上
tds_version = 8.0
{% endhighlight %}

五. Perl测试程序(见mac部分)

## mac

一. 安装

{% highlight bash %}
brew install unixodbc  
brew install freetds --with-unixodbc
{% endhighlight %}

二. 准备配置文件(/usr/local/etc/freetds.conf)

{% highlight properties %}
[cactus]  
host = cactus  
port = 1433  
tds version = 8.0  
{% endhighlight %}

三. 准备配置文件(/usr/local/etc/odbcinst.ini)

{% highlight properties %}
[FreeTDS]
Description = FreeTDS Driver v0.91  
Driver = /usr/local/Cellar/freetds/0.91_2/lib/libtdsodbc.so  
Setup = /usr/local/Cellar/freetds/0.91_2/lib/libtdsodbc.so  
Trace = yes  
TraceFile = /tmp/odbc.log  
fileusage=1  
dontdlclose=1  
UsageCount=1  

{% endhighlight %}

四. 准备配置文件(/usr/local/etc/odbc.ini)

{% highlight properties %}
[cactus]  
Description = MSSQL Server  
Driver = FreeTDS  
Server = cactus  
Port  = 1433  
tds_version = 8.0  
{% endhighlight %}

五. Perl测试程序

{% highlight perl %}
#!/usr/bin/perl
use warnings;
use strict;
use DBI;
use Data::Dump;

my $dbh = DBI->connect(
    'dbi:ODBC:cactus',
    'hary',
    'jessie',
    {

      RaiseError => 1,
      AutoCommit => 0
    }
) || die "Database connection not made: $DBI::errstr";

warn "连接数据库成功";

$dbh->do("use batchtest");

warn "设置schema成功";

my $sth = $dbh->prepare(qq/select top 10 * from ApplicationOverDueObjects1103/);

$sth->execute();

while( my $href = $sth->fetchrow_hashref ) {
    Data::Dump->dump($href);
}

{% endhighlight %}

六. 打包成docker

{% highlight bash %}
FROM hary/ubuntu:14.04
MAINTAINER hary <94093146@qq.com>

# 修改apt源为aliyun
ADD assets/sources.list /etc/apt/sources.list
RUN apt-get update

# 安装freetds and unixodbc
RUN apt-get install freetds-bin freetds-common freetds-dev libct4 libsybdb5 unixodbc unixodbc-dev unixodbc-bin libodbc1 odbcinst1debian2 tdsodbc php5-odbc

# 安装perl相关内容
RUN cpan install DBI DBD::ODBC

# 准备配置文件
ADD assets/freetds.conf /etc/freetds/freetds.conf
ADD assets/odbcinst.ini /etc/odbcinst.ini
ADD assets/odbc.ini     /etc/odbc.ini
ADD setup.sh /setup.sh

# 配置
RUN /setup.sh

# 运行指令!
CMD /run.sh
{% endhighlight %}


{% include JB/setup %}
