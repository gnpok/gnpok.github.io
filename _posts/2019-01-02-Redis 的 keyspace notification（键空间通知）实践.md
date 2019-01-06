---
layout: post
title:  "Redis 的 keyspace notification（键空间通知）实践"
categories: Redis
tags:  Redis
author: cossete

---

[TOC]

[文章输入摘录，吸取别人长处](https://www.imooc.com/article/10431)

##  需求和可行性
最近有这样的需求：

- 设置了生存时间的Key，在过期时能不能有所提示？
- 如果能对过期Key有个监听，如何对过期Key进行一个回调处理？

在知道 Redis 从2.8.0版本后，推出 Keyspace Notifications 特性后，对Key过期事件的处理，有了可能。

## Key过期事件的Redis配置

这里需要配置 notify-keyspace-events 的参数为 “Ex”。x 代表了过期事件。
```
notify-keyspace-events "Ex"
```
保存配置后，重启Redis服务，使配置生效。
```
[root@chokingwin etc]# service redis-server restart /usr/local/redis/etc/redis.conf 
Stopping redis-server: [ OK ] 
Starting redis-server: [ OK ]
```

## 添加过期事件订阅
开启一个终端，redis-cli 进入 redis 。开始订阅所有操作，等待接收消息。
```
127.0.0.1:6379> psubscribe __keyevent@0__:expired
Reading messages... (press Ctrl-C to quit)
1) "psubscribe"
2) "__keyevent@0__:expired"
3) (integer) 1
```
再开启一个终端，redis-cli 进入 redis，新增一个 10秒过期的键 name
```
127.0.0.1:6379> setex name 10 chokingwin
OK
```
10秒过期时间到时，另外一边执行了阻塞订阅操作后的终端，有如下信息输出
```
1) "pmessage"
2) "__keyevent@0__:expired"
3) "__keyevent@0__:expired"
4) "name"
```
顺利获取到过期键 name，说明对过期Key信息的订阅是成功的。

## phpredis实现订阅Keyspace notification
phpredis 是 php 的一个扩展，使代码内可像客户端操作redis那样操作方便。
下面是对 phpredis 封装的一个类 redis.class.php
```
class MyRedis{

    private $redis;

    /**
     * 构造函数
     *
     * @param string $host 主机号
     * @param int    $port 端口号
     */
    public function __construct($host='127.0.0.1',$port=6379){
        $this->redis = new redis();
        $this->redis->connect($host,$port);
    }
    public function expire($key=null,$time=0){
       return $this->redis->expire($key,$time);
    }

    public function psubscribe($patterns=array(),$callback){
        $this->redis->psubscribe($patterns,$callback);
    }

}// End Class
```
下面对上面的类进行一些实例化，以期达到过期事件的订阅。脚本 testRedisNotify.php 如下
```
<?
require_once './redis.class.php';

$redis = new MyRedis();
$redis->psubscribe(array('__keyevent@0__:expired'),'psCallback');

// 回调函数,这里写处理逻辑
function psCallback($redis, $pattern, $chan, $msg){
    echo "Pattern: $pattern\n";
    echo "Channel: $chan\n";
    echo "Payload: $msg\n\n";
}
```

**说明：psCallback 函数为订阅事件后的回调函数。$redis, $pattern, $chan, $msg 四个参数为回调时返回的参数。
详细说明见下面 Redis 官方文档对 psubscribe 的说明。**

```
    /**
     * Subscribe to channels by pattern
     *
     * @param   array           $patterns   The number of elements removed from the set.
     * @param   stringarray    $callback   Either a string or an array with an object and method.
     *                          The callback will get four arguments ($redis, $pattern, $channel, $message)
     * @link    http://redis.io/commands/psubscribe
     * @example
     * <pre>
     * function psubscribe($redis, $pattern, $chan, $msg) {
     *  echo "Pattern: $pattern\n";
     *  echo "Channel: $chan\n";
     *  echo "Payload: $msg\n";
     * }
     * </pre>
     */
    public function psubscribe( $patterns, $callback ) {}
```
因为订阅事件启动后是阻塞执行的，所以我们尝试在终端执行 testRedisNotify.php 这个脚本。
```
[root@chokingwin etc]# php testRedisNotify.php
```
新开一个终端，在 redis 里 新增一个 10秒过期 的键 chokingwin
```
127.0.0.1:6379> setex name 10 chokingwin
OK
```
10秒过期时间到时，另外一边执行了脚本被阻塞的终端，有如下信息输出
```
Pattern: __keyevent@0__:expired
Channel: __keyevent@0__:expired
Payload: name
```
说明利用 phpredis 对过期事件的订阅，是成功的。

## 补充
redis的客户端读取超时原因导致。
那么对 redis客户端进行一些参数设置，使读取超时参数 为 -1，表示不超时。
在 redis.class.php 里新增一个函数
```
    public function setOption(){
        $this->redis->setOption(Redis::OPT_READ_TIMEOUT,-1);
    }
```
然后在脚本里调用这个 setOption() 函数。
```
<?
require_once './redis.class.php';

$redis = new MyRedis();
$redis->setOption();

$redis->psubscribe(array('__keyevent@0__:expired'),'psCallback');

// 回调函数,这里写处理逻辑
function psCallback($redis, $pattern, $chan, $msg){
    echo "Pattern: $pattern\n";
    echo "Channel: $chan\n";
    echo "Payload: $msg\n\n";
}
```
## 有个问题
做到这一步，利用 phpredis 扩展，成功在代码里实现对过期 Key 的监听，并在 psCallback()里进行回调处理。
开头提出的两个需求已经实现。
**似乎一切都很顺利。**
可是这里有个问题：redis 在执行完订阅操作后，终端进入阻塞状态，需要一直挂在那。且此订阅脚本需要人为在命令行执行，不符合实际需求。
实际上，我们对过期监听回调的需求，是**希望它像守护进程一样，在后台运行，当有过期事件的消息时，触发回调函数。**

## 使监听后台始终运行
希望像守护进程一样在后台一样，我是这样实现的。

Linux中有一个nohup命令。功能就是不挂断地运行命令。
同时nohup把脚本程序的所有输出，都放到当前目录的nohup.out文件中，如果文件不可写，则放到<用户主目录>/nohup.out 文件中。那么有了这个命令以后，不管我们终端窗口是否关闭，都能够让我们的php脚本一直运行。
来看操作。
我们新建一个 nohupRedisNotify.php。
```
vim nohupRedisNotify.php
```
编写脚本同 testRedisNotify.php 类似。
```
<?
#! /usr/local/php/bin/php

require_once './redis.class.php';

$redis = new MyRedis();
$redis->setOption();

$redis->psubscribe(array('__keyevent@0__:expired'),'psCallback');

// 回调函数,这里写处理逻辑
function psCallback($redis, $pattern, $chan, $msg){
    echo "Pattern: $pattern\n";
    echo "Channel: $chan\n";
    echo "Payload: $msg\n\n";
}

?>
```
不过我们在开头，需要申明 php 编译器的路径：#! /usr/local/php/bin/php 。 这是执行 php 脚本所必须的。

然后，nohup 不挂起执行 nohupRedisNotify.php，注意 末尾的 &
```
[root@chokingwin HiGirl]# nohup ./nohupRedisNotify.php &
[1] 4456
nohup: ignoring input and appending output to `nohup.out'
```
ps 确认一下脚本是否已在后台运行。查看进程结果如下：
```
[root@chokingwin HiGirl]# ps
  PID TTY          TIME CMD
 3943 pts/2    00:00:00 bash
 4456 pts/2    00:00:00 nohupRedisNotif
 4480 pts/2    00:00:00 ps
```
脚本确实已经在 4456 号进程上跑起来。

最后在查看下nohup.out
cat 一下 nohuo.out，看下是否有过期输出。
```
[root@chokingwin HiGirl]# cat nohup.out 
[root@chokingwin HiGirl]#
```
并没有。我们还是老样子，新增一个10秒过期的的键 name。10秒后，我们再 cat 一次。
```
[root@chokingwin HiGirl]# cat nohup.out 
Pattern: __keyevent@0__:expired
Channel: __keyevent@0__:expired
Payload: name
```
说明监听过期事件并回调成功。