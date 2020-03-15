---
title: Mac OS X 10.9 自带 php-fpm 的配置使用和扩展安装
date: 2014-01-11T00:00:00+08:00
draft: false 
tags: ["php", "Mac OS X", "配置"]
slug: macosx-conf-preloaded-php-fpm
---

Mac OS X 10.9 自带有 php-fpm，本文把预装的 php-fpm 配置起来。

直接运行，有报错找不到配置文件。

```sh
$ php-fpm
[11-Jan-2014 16:03:03] ERROR: failed to open configuration file '/private/etc/php-fpm.conf': No such file or directory (2)
[11-Jan-2014 16:03:03] ERROR: failed to load configuration file '/private/etc/php-fpm.conf'
[11-Jan-2014 16:03:03] ERROR: FPM initialization failed
```

可以在 /private/etc/ 目录下生成配置文件，需要 root 权限(sudo)
或者在普通用户有权限的目录里放置配置文件，通过 `--fpm-config` 参数指定配置文件的位置，如下：

```sh
$ cp /private/etc/php-fpm.conf.default /usr/local/etc/php-fpm.conf
$ php-fpm --fpm-config /usr/local/etc/php-fpm.conf
[11-Jan-2014 16:10:49] ERROR: failed to open error_log (/usr/var/log/php-fpm.log): No such file or directory (2)
[11-Jan-2014 16:10:49] ERROR: failed to post process the configuration
[11-Jan-2014 16:10:49] ERROR: FPM initialization failed
```

错误信息显示：不能正确的打开”日志“文件，原因是默认在 /usr/var 目录下工作，可以修改配置文件指定正确的日志文件路径。修改 `/usr/local/etc/php-fpm.conf` 文件中的 error_log 项，默认前缀是/usr/var ，但并没有这个路径

```ini
error_log = /usr/local/var/log/php-fpm.log
pid = /usr/local/var/run/php-fpm.pid
```

或者不修改配置文件中配置项的路径，在 php-fpm 的运行参数中(-p)指定放置运行时文件的相对路径前缀

```sh
$ php-fpm --fpm-config /usr/local/etc/php-fpm.conf  --prefix /usr/local/var
```

到此，php-fpm守护进程已经基本可以正确的启动了。

下面我们看下php.ini配置文件及扩展的安装。
首先看下编译参数，有些值是编译进执行程序的，无法更改。

运行 `php -i|grep config` 找到配置文件(php.ini)、目录的位置，下面两项的值指定

    '--with-config-file-path=/etc'
    '--with-config-file-scan-dir=/Library/Server/Web/Config/php'

所以我们需要在/etc目录下创建 php.ini，Mac 在 /private/etc，/etc 下均提供了样例文件 php.ini.default，通过查验，两个文件完全相同，所以复制哪一个都无所谓，Mac 有提供 md5 而不是 Linux 下的 md5sum：

```sh
$ md5 /private/etc/php.ini.default /etc/php.ini.default
MD5 (/private/etc/php.ini.default) = 1c47241665ea5efdc55fd5809f675449
MD5 (/etc/php.ini.default) = 1c47241665ea5efdc55fd5809f675449
```

/etc目录权限root:wheel，需要root权限或使用sudo，关于如何设置Mac的sudo命令需要的密码，请查看

1. http://support.apple.com/kb/HT4103?viewlocale=zh_CN&locale=zh_CN
1. http://support.apple.com/kb/PH6515?viewlocale=zh_CN

```sh
# cp /etc/php.ini.default  /etc/php.ini
变更own，以后修改不用老是切换root，生产环境最好不要改
# chown <你的用户名> /etc/php.ini
# chmod u+w /etc/php.ini
```

安装PHP扩展
/Library/Server/Web/Config/php 这个目录并不存在，或者 Mac OS X Server 版本有吧，不知道，手动创建它，以root权限

```sh
# mkdir -p /Library/Server/Web/Config/php
```

编译扩展，brewhome 是另起炉灶，brew 方式安装扩展需要依赖 php，如 php54-redis 会依赖 php54，至于编译出来的扩展是否可以配置到自带的，没有实验过。下面以 [php_discuz](https://github.com/potterhe/php_discuz) 扩展为例。假如扩展源码在 /Users/apple/php_discuz 目录

```sh
$ ./configure
$ make
# 扩展编译后，默认会存储在 /Users/apple/php_discuz/modules/discuz.so
# 将扩展在配置文件中打开
$ echo "extension=/Users/apple/php_discuz/modules/discuz.so" > /Library/Server/Web/Config/php/discuz.ini

# 测试验证
$ php -i|grep discuz
discuz support => enabled

# 运行用例测试
$ php -f /Users/apple/php_discuz/discuz.php
```
