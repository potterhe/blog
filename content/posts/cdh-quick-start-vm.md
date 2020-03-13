---
title: cdh-5.13 quickstart vm 使用笔记
date: 2019-07-06T23:59:59+08:00
draft: false
tags: ["cdh", "5.13", "quickstart", "vm"]
---

## 下载和导入虚拟机.

- [下载 cdh-quickstart-vm](https://www.cloudera.com/downloads/quickstart_vms/5-13.html)
- [导入](https://www.cloudera.com/documentation/enterprise/5-13-x/topics/quickstart_vm_administrative_information.html)

## vm初始信息收集

```sh
[cloudera@quickstart ~]$ uname -a
Linux quickstart.cloudera 2.6.32-573.el6.x86_64 #1 SMP Thu Jul 23 15:44:03 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux

$ cat /etc/issue
CentOS release 6.7 (Final)
Kernel \r on an \m
```

vm里的cdh服务当前是使用操作系统的init系统管理的，而不是"Cloudera Manager"

```
By default, the Cloudera QuickStart VM run Cloudera's Distribution including
Apache Hadoop (CDH) under Linux's service and configuration management. If you
wish to migrate to Cloudera Manager, you must run one of the following
commands.

To use Cloudera Express (free), run:

    sudo /home/cloudera/cloudera-manager --express

This requires at least 8 GB of RAM and at least 2 virtual CPUs.

To begin a 60-day trial of Cloudera Enterprise with advanced management
features, run:

    sudo /home/cloudera/cloudera-manager --enterprise

This requires at least 10 GB or RAM and at least 2 virtual CPUs.
```

通过以下命令查看服务状态.

```sh
[cloudera@quickstart ~]$ service --status-all
```

## CDH集群webui端口整理

[CDH集群对外需要开放的端口整理](https://blog.csdn.net/u012551524/article/details/78887094)

- [hdfs](http://quickstart.cloudera:50070)
- [JournalNode](http://quickstart.cloudera:8480)
- [yarn](http://quickstart.cloudera:8088)
- [mapreduce.jobhistory.webapp.address](http://quickstart.cloudera:19888)
- [hbase-master](http://quickstart.cloudera:60010)
- [hue](http://quickstart.cloudera:8888)
- [spark.history.ui.port](http://quickstart.cloudera:18088)

## spark

### 升级spark到2.x

- https://www.cloudera.com/documentation/spark2/latest/topics/spark2_installing.html
- https://www.cloudera.com/documentation/spark2/latest/topics/spark2_known_issues.html#known_issues
- https://www.cloudera.com/documentation/spark2/latest/topics/spark2.html
- https://blog.csdn.net/u010936936/article/details/73650417
- https://www.youtube.com/watch?v=lQxlO3coMxM

### 安装openjdk-8

- https://www.cloudera.com/documentation/enterprise/upgrade/topics/ug_jdk8.html
```
[cloudera@quickstart ~]$ sudo yum -y install java-1.8.0-openjdk
```

修改JAVA_HOME. 修改/etc/profile

```
#export JAVA_HOME=/usr/java/jdk1.7.0_67-cloudera
export JAVA_HOME=/usr/lib/jvm/jre-1.8.0-openjdk.x86_64
```

### 安装apache spark

1. 下载[spark-2.3.3-bin-without-hadoop.tgz](https://mirrors.tuna.tsinghua.edu.cn/apache/spark/spark-2.3.3/spark-2.3.3-bin-without-hadoop.tgz)
2. 解压到/usr/lib/spark-2.3.3-bin-without-hadoop
3. http://dblab.xmu.edu.cn/blog/1307-2/
4. http://spark.apache.org/docs/2.3.3/configuration.html#inheriting-hadoop-cluster-configuration

### 共享目录

1. 宿主机目录/PATH/TO/vbox-shared-folders,不指定挂载点时，默认挂载到/media/sf_vbox-shared-folders, 且cloudera用户没有权限, 使用root帐号将cloudera加入vboxsf组以授权.

```sh
[cloudera@quickstart ~]$ ls -lah /media
总用量 12K
drwxr-xr-x.  4 root root   4.0K 7月   4 18:56 .
drwxrwxr-x. 22 root root   4.0K 7月   5 2019 ..
drwxr-xr-x   2 root root   4.0K 10月 23 2017 cdrom
drwxrwx---   1 root vboxsf   96 7月   4 19:00 sf_vbox-shared-folders

[cloudera@quickstart ~]$ sudo -i
[root@quickstart ~]# usermod -a -G vboxsf cloudera
```

3. 重登陆，查看权限组. 如果重新登陆不行，就重启下vm

```sh
[cloudera@quickstart ~]$ id
uid=501(cloudera) gid=501(cloudera) 组=501(cloudera),474(vboxsf),502(default)
```

## hbase

https://www.jianshu.com/p/cc9bfb116886
https://index.scala-lang.org/hbase4s/hbase4s/hbase4s-core/0.1.2?target=_2.11
