---
title: "cacti源码分析－数据采集"
date: 2013-05-25T23:59:50+08:00
draft: false
tags: ["cacti", "源码分析", "数据采集", "php", "多进程"]
---

cacti用于监控系统的各项运行指标，提供了交互界面和图表，是一个整合工具集，它完成两个核心任务： 1)指标数据的采集，2) 将数据通过数图进行展示。其中图表的绘制、图表数据的存储是通过rrdtool工具实现的，[《RRDtool简体中文教程》](http://bbs.chinaunix.net/thread-864861-1-1.html)对rrdtool工具进行了介绍，是很好的资料。本文分析指标数据采集的实现。

## 如何获取目标数据
我们需要按照目标数据的暴露方式去采集相应的数据. 基于主机(host)的数据，如：系统负载，网卡流量，磁盘IO，TCP连接等已通过SNMP标准化，需要使用SNMP方式获取。应用级的数据要按照应用暴露数据的方式去获取,如: 如果要监控nginx（stub_status）数据,该项数据是通过http方式暴露，需使用http获取数据;如果要监控mysql-server（show status）数据，需要可以连接到mysql服务器，并有权限打印数据。

## cacti由cron驱动
cacti不是一个daemon进程，它由cron驱动。通常我们需要配置如下的cron: 下面的典型配置为每5分钟运行一次cacti的轮换数据进程。

```cron
*/5 * * * * /PATH/TO/php /PATH/TO/cacti/poller.php > /dev/null 2>&1
```
cron任务的运行频率，影响cacti采集数据周期，但cron的运行频率，并不是cacti最终的采集频率。举例来说，如果cron的运行频率为每5分钟触发一次，但我们希望每分钟采集一次数据，cacti是可以做到的，这要求我们将cron的运行频率正确的配置到cacti，这样cacti会在一次cron进程生命周期内，尽可能的按照预期的频率，进行多次数据采集。

### 实现
涉及 $cron_interval，$poller_interval 两项参数，比如cron的周期是5分钟，poller周期是1分钟，则cron触发的poller.php进程，要负责安排5次数据轮询，以满足poller粒度。

```php
$poller_runs = intval($cron_interval / $poller_interval);
define("MAX_POLLER_RUNTIME", $poller_runs * $poller_interval - 2); // poller.php进程的最大运行时间（比计划任务周期少2秒）
```

## poller进程(poller.php)
poller进程由cron启动，在其生命周期内，负责编排所有的数据采集任务.

### 任务初始化
poller进程启动后，在进行一系列cacti的初始化后，从系统中检索出数据采集任务集，然后将它们持久化到数据库(poller_output表)中,每一项数据指标一条记录。

### 并发控制
cacti提供了两种并发模型来提升数据采集的效率，cmd.php和spine。其中cmd.php是多进程模型，用php语言实现; spine是多线程模型，具体实现不详。这里我们只讨论cmd.php方式的并发。下面的引文来自cacti的配置界面说明，大意是说，当使用cmd.php抓取数据时，可以通过增加进程数来提高性能（多进程模型）；当使用spine时，应该通过增加“Maximum Threads per Process”的值，提升性能（多线程模型）。

>    "The number of concurrent processes to execute. Using a higher number when using cmd.php will improve performance. Performance improvements in spine are best resolved with the threads parameter"

并发的进程数的参数为 $concurrent_processes, 系统默认值1，即只会使用一个进程负责所有主机的数据轮询，cacti分配的规则如下。

- 分配是在host粒度上进行的。即只分配host编号，该host上的所有子数据项都在一个进程中完成。
- poller根据“任务集”和$concurrent_processes 的值，计算每个进程需要负责的任务的平均值 $items_per_process。然后poller进程开始遍历所有的host的任务集并累计计数, 并累计任务数 >= $items_per_process时，就生成一个cmd进程，并由这个进程负责完成累计范围内的host集的数据获取任务。

为了方便描述并发进程的任务分配，我们设计一个简化的场景。

    +-----------+-----------+--------+
    | 主机编号  | 主机标识  | 任务数 |
    +-----------+-----------+--------+
    | 1         | host1     |   2    |
    | 2         | host2     |   3    |
    | 3         | host3     |   3    |
    +-----------+-----------+--------+

设置发进程数为 $concurrent_processes = 2, 那么cacti的分配过程如下

    $items_per_process = (2+3+3) / 2 = 4;

迭代主机任务集过程

- host1时, 累计任务数 2, 小于4，继续迭代
- host2时, 累计任务数 5, 大于4，此时派生一个cmd.php后台进程，并将开始host（1）和结束host（2）的id传递给这个进程, 重置累计计数.
- host3时, 累计任务数 3, 小于4，但迭代结束，派生一个cmd.php进程，把host(3)传递给它. 

#### 派生cmd.php进程的代码实现

```php
// exec_background('/PATH/TO/php', "-q /PATH/TO/cacti/cmd.php 1 2");
exec_background($command_string, "$extra_args $first_host $last_host");

function exec_background($filename, $args) {
    ...
    exec($filename . ' ' . $args . ' > /dev/null &');
    ...
}
```

关于exec，手册有如下说明: "如果程序使用此函数启动，为了能保持在后台运行，此程序必须将输出重定向到文件或其它输出流。 否则会导致 PHP 挂起，直至程序执行结束。"，所以' > /dev/null &'很重要,通过查看php的实现，exec、system、passthru都是基于php_exec的。流程运行至此，获取数据的任务就交给cmd进程在后台运行了，cmd进程的实现，后面再分析。

### 具体数据项的采集(cmd.php)
cmd.php进程启动后，通过主机编号参数检索出任务集, 然后逐项采集，并把采集的数据集存储到对应任务的数据库记录中(poller_output表), 这里存储的数据是暂时性的. 当cmd.php完成所有的任务后，就会向poller进程报告。根据任务数据获取方式的不同，cacti使用3种方式采集. 这三种方式后文再详细描述.

- snmp
- shell 或其它可执行程序
- php脚本 script_server.php


### 数据汇总，趋势图表
poller进程把主机(host)分配给cmd.php进程后，就一直不停的(usleep(500))轮询数据库(poller_output表),当发现有完成的任务时，它就把数据写入rrd文件，并把poller_output表中的记录删除。界面查看到的趋势图表等的数据源都是从rrd文件获取的。同时poller进程会关注完成所有采集任务的cmd.php进程，当所有进程都完成后(成功完成的进程（end_time）数 >= 启动的后台进程数)，本轮采集就结束了。

rrd相关的代码如下:

```php
$rrdtool_pipe = rrd_init();
...
fwrite($rrdtool_pipe, escape_command(" $command_line") . "\r\n")
fflush($rrdtool_pipe);
...
rrd_close($rrdtool_pipe);

// rrd_init()主要使用下面的代码实现：
function rrd_init() {
    ...
    $command = "/usr/bin/rrdtool - > /dev/null 2>&1";
    popen($command, "w");
    ...
}
```

### 进程模型

    -- poller.php  ---- 初始化任务集并存储 ------------------------------- 轮询数据库usleep(500) --> 数据写入rrd文件
         |                                                                  \                    /
         |- (fork) cmd.php(spine) -> 采集(snmp) --------------------------> 结果写入数据库(poller_output表)
                                    |                   /               /
                                    |- script (popen) -/               /
                                    |- script_server.php (proc_open) -/
                                    |- ... ...(更多ss进程)
         |- ... ...(更多cmd进程)


## cmd进程(cmd.php)
cmd.php接收两个参数 $first_host $last_host, 脚本通过$_SERVER["argv"][1],$_SERVER["argv"][2] 获取这两项参数，取出给定host的poller_items(cacti管理界面中Data Souries)记录。如果没有提供参数（$_SERVER["argc"] == 1），脚本会从数据库中取最近需要轮询数据的host。下面是数据轮询流程的伪码。

```php
$ping = new Net_Ping;
foreach ($polling_items as $item) {
    if (($new_host) && (!empty($host_id))) {
        /* perform the appropriate ping check of the host */
        /* 如果是一个新host，cmd进程会先检测host是否可到达。 */
        if ($ping->ping(... ...) {
            /* up or down*/
        }
    }
    /* 根据目标数据暴露的方式，提供了3种获取数据的方式 */
    if (!$host_down) {
        switch ($item["action"]) {
            case POLLER_ACTION_SNMP: /* snmp */
            case POLLER_ACTION_SCRIPT: /* script (popen) */
            case POLLER_ACTION_SCRIPT_PHP: /* script (php script server) */
            default:
        }
    }
}
```

### POLLER_ACTION_SNMP
即通过snmp协议获取目标数据，需提供commuinty、OID等参数，如果启用了php_snmp扩展，可使用扩展提供了snmp函数获取数据；如果没有php_snmp扩展，可以通过php脚本调用net-snmp包提供的命令行工具来获取数据；或者用php的socket自己实现snmp协议也可以。使用此种方式采集数据在本进程中进行，不会派生子进程.

以下是一些OID, host信息
- snmptranslate -Td -Of 1.3.6.1.2.1.25
- .iso.org.dod.internet.mgmt.mib-2.host

网卡流量信息

- RFC1213-MIB::ifDescr
- RFC1213-MIB::ifInOctets
- RFC1213-MIB::ifOutOctets

### POLLER_ACTION_SCRIPT，
主要由下面的调用语句实现，是基于popen实现的

```php
$output = trim(exec_poll($command));
exec_poll()主要通过下面的调用实现：
$fp = popen($command, "r");
$output = fgets($fp, 8192);
```

### POLLER_ACTION_SCRIPT_PHP
主要由下面语句调用，是基于proc_open实现的。这种方式通过一个可双向通信的子进程（运行script_server.php上下文，后面均称ss进程），相互配合完成任务。这种方式可以复用子进程，因此会减少一些系统开销。ss进程通过下面的调用初始化。

```php
$cactiphp = proc_open("/PATH/TO/php -q /PATH/TO/cacti/script_server.php cmd", $cactides, $pipes);
```
关于proc_open的参数，作一些笔记，它表现如下的行为：
它派生(fork)一个子进程，子进程使用$cactides(参数2)指定的方式，初始化自己已打开的文件描述符表。如

```php
cactides = array(
   0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
   1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
   2 => array("file", "/tmp/error-output.txt", "a"), // stderr is a file to write to
   3 => array("file", "/PATH/TO/file", 'rw'),
);
```
该参数（参数2）是一个索引数组，其中0，1，2三个索引位置已经固定分配（0 is stdin, 1 is stdout, while 2 is stderr），其它索引位置可以是任何有效的文件描述符编号。
索引数组的元素也需要是一个数组，其元素有以下约定。

第一个元素是文件描述符的类型，有两种类型可供选择：pipe和file。第二个元素的语义依赖于第一个元素，如果是pipe，第二个元素是文件描述符对应到管道的哪一端，"r"为读端，"w"为写端；管道的另一端将通过$pipes(参数3)返回。管道常用作进程间通信，它是半双工的，数据只能向一个方向流动，即会有只能“写”数据的一端和只能“读”数据的一端。如果是file，第二个元素值是打开文件的路径，第三个元素是打开文件的模式；
在上例中，ss进程的文件描述符0，对应到一个管道(pipe)的读端，描述符1对应到一个管道的写端，描述符2、3都对应到一个文件。$pipes（参数3）是一个数组，$pipes[0]对应描述符0（stdin）的管道的写端，$pipes[1]对应描述符1（stdout）的管道的读端。
文件描述符处理完成后，ss进程运行（exe函数簇）$cmd(第一个参数)程序的上下文，这里是（script_server.php）。进程上下文的替换不影响描述符表。

cmd进程生成ss进程后，在$pipes[1]（对应ss进程的stdout）读取子进程的输出，以判定子进程是否已经成功启动。

```php
$output = fgets($pipes[1], 1024);
substr_count($output, "Started") != 0
```
fgets会阻塞，直到从$pipes[1]中读取足够数据、或者EOF、或者cmd进程收到一个signal。然后在输出字串中查找"Started"串。

ss进程启动后，会在stdout输出下面的内容（包含“Started”串），以告知cmd进程，自己已经准备好了。 然后ss进程阻塞在“读取stdin”上，等待cmd进程的指令

```php
fputs(STDOUT, "PHP Script Server has Started - Parent is " . $environ . "\n");
... ...

/* process waits for input and then calls functions as required */
while (1) {
    ... ...
    $input_string = fgets(STDIN, 1024);
    ... ...
}
```
至此，cmd进程和ss进程，已完成协调工作的准备工作。当cmd进程需要ss进程配合获取数据时，cmd进程就往ss进程的stdin写命令，ss进程完成后，将结果写入stdout以返回给cmd进程。下面是调用代码：

```php
$output = trim(str_replace("\n", "", exec_poll_php($item["arg1"], $using_proc_function, $pipes, $cactiphp)));
```

其中exec_poll_php的主要实现语句如下：

```php
function exec_poll_php($command, $using_proc_function, $pipes, $proc_fd) {
    ... ...
    fwrite($pipes[0], $command . "\r\n");
    $output = fgets($pipes[1], 8192);
    ... ...
    return $output;
}
```
cmd进程的分析就到这里，下面我们来看一下ss进程的实现。

## script_server.php
上文提到，ss进程初始完成后，便阻塞于 fgets(STDIN, 1024);以等待cmd进程的命令，下面我们就来看一下，ss进程从stdin获取命令后的流程实现。

```php
$input_string    = fgets(STDIN, 1024);
/* 如果命令的前4个字符是“quit”,ss进程执行退出 */
if (substr($input_string,0,4) == "quit") {
    exit(1);
}
/* pull off the parameters */
$i = 0;
while ( true ) {
    /* 查找命令中的空格，以切分命令参数 */
    $pos = strpos($input_string, " ");
    if ($pos > 0) {
        switch ($i) {
            case 0:
                /* cut off include file as first part of input string and keep rest for further parsing */
                $include_file = trim(substr($input_string,0,$pos));
                $input_string = trim(strchr($input_string, " ")) . " ";
                break;
            case 1:
                /* cut off function as second part of input string and keep rest for further parsing */
                $function = trim(substr($input_string,0,$pos));
                $input_string = trim(strchr($input_string, " ")) . " ";
                break;
            case 2:
                /* take the rest as parameter(s) to the function stripped off previously */
                $parameters = trim($input_string);
                break 2;
            }
        }else{
            break;
        }
    $i++;
}
```

上面的代码显示，传递给ss进程的命令串以下面的格式组织：

    include_file<空格>function名<空格>parameters串

- include_file    的路径为/PATH/TO/cacti/scripts/include_file
- function        名字为include_file文件中定义的函数名。
- parameters      为function的参数列表。

通过这个机制，我们可以扩展新的获取数据的方法。

## 最后
至此，获取数据流程实现就分析完了。平常用php写网页比较多一些，基本是与数据库打交道，不会涉及到进程层面的处理，cacti的实现恰好在把php用作shell方面给我一个学习的样例。学习过程中，涉及到了操作系统的一些知识，如系统调用、进程间通信、IO阻塞等；有时候会感叹：“哦，原来php是这样来实现这个功能的”。

作为参照比较，nagios提供了check_by_ssh nrpe NSCA几种标准化、易于扩展的机制。如果要整合nagios和cacti的话，在获取数据层面，应该可以实施。如果有net-snmp开发能力，也可以把nginx、mysql-server的数据通过SNMP的方式暴露。

大规则集群环境中，更多使用分布式的监控系统，如zabbix。但对于phper来说，能近距离的观察cacti的实现，仍然是一件大有裨益的事儿， 感谢cacti、感谢开源。
