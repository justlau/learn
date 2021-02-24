# 1. redis 安装
### redis是什么？
redis是数据库的一种，我们常见的数据库可以分为关系型数据库和非关系型数据库，redis就是非关系型数据库的一种。并且redis是key-value型数据库。
### 从上面的解释引出新的问题：关系型数据库和非关系型数据库：
* 关系型数据库：使用关系模型来组织数据的数据库。使用表结构来维护数据结构，可以使用通用sql语句操作数据库。数据库事务必须遵循**ACID**
* 非关系型数据库：非关系型，数据结构不固定，不需要遵循acid，分布式的。
### 为什么要使用redis？
当我们使用一门技术时，首先要想的是我为什么要使用它，使用它能给我带来什么好处,能解决什么问题，又会带来什么问题？
redis要记住的特点就是：快，单线程
因为redis是单线程的，因此不存在线程安全的问题，而且reids是基于内存的。所以速度快。使用redis可以显著的缓解数据库的压力。
使用redis做缓存时，可以解决缓存多个服务之间缓存共享的问题。
### redis为什么单线程还会这么快？
redis采用了非阻塞的I/O多路复用技术，保证redis在多连接时系统的吞吐量。

![Image text](https://raw.githubusercontent.com/BinLiuA/resources/main/redis%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8.png)
假设有三个redis实例连接了redis server
````
任何进程在linux系统中都有IO对应的一个文件描述符fd(file discriptor),三个redis client 在连接上redis server时都会有对应的文件描述符

内核中Socket的创建可以指定是阻塞或非阻塞

int socket(int domain,int type,int protocol),domain 就是ip，type就可以指定是阻塞或者非阻塞的，protocol就是协议

这些都可以通过linux的man命令来查看，安装man命令，yum install man，man可以查看8类文档，其中2类就是系统调用，man 2 
````

图一：早期在操作系统内核暴露的read方法，server需要阻塞的调用read方法来获取文件描述符到达的指令，这就是早期的BIO模型。

图二：在内核的跃迁中，出现了非阻塞IO，在用户空间中使用同步非阻塞方式轮询这三个文件描述符，查询是否有指令到达，这样的弊端在于，如果需要监听的文件描述符过多，那么就会造成大量的系统调用，造成频繁的内核态到用户态的切换。这就是NIO

图三：对于图二出现的问题，内核出现了一个系统调用select，将所监控的文件描述符通过select全部传给内核，由内核来轮询这些文件描述符是否有指令到达，内核遍历结束后将有指令到达的文件描述符返回给server，server再通过read方法去内核读取已经到达的指令。这就是多路复用的NIO

图四：图三的调用也略显复杂，因为由于内核空间和用户空间是相互隔离的，每次的select和read调用都会触发数据的拷贝(也就是select或read的方法参数)，随着内核的再次跃迁，epoll和mmap出现了，epoll是一个IO模型，内核通过mmap开辟一块共享空间，由内核和server共同使用，在这块共享空间内维护了一个红黑树和一个链表，红黑树存放的是server需要监听的文件描述符，链表存放的是到达的指令。server通过调用epoll的create方法获得一个epoll的文件描述符并且会开辟一块共享空间，如果server有新的文件描述符到达时，通过epoll的ctl的调用将文件描述符写入到红黑树中，若有新的指令到达，内核会将指令放在链表中，server只需要阻塞获取链表的数据即可，这就是事件驱动，减少了内核态到用户态的切换，同时也减少了数据的拷贝。redis就是基于epoll的事件驱动来实现单线程的高性能的。
## 1.1 简单安装redis
从网上下载redis-5.0.0.tar.gz压缩包，解压压缩包：
```
tar zxvf redis-5.0.0.tar.gz
```
然后移动文件至目录 `/usr/local/`下：
```
mv redis-5.0.0 /usr/local/
```
进入redis解压后文件目录 `cd /usr/local/redis-5.0.5`，可以看到如下内容：
```
[root@localhost redis-5.0.5]# ll
total 268
-rw-rw-r--.  1 root root 106874 May 15  2019 00-RELEASENOTES
-rw-rw-r--.  1 root root     53 May 15  2019 BUGS
-rw-rw-r--.  1 root root   2381 May 15  2019 CONTRIBUTING
-rw-rw-r--.  1 root root   1487 May 15  2019 COPYING
drwxrwxr-x.  6 root root   4096 May 15  2019 deps
-rw-rw-r--.  1 root root     11 May 15  2019 INSTALL
-rw-rw-r--.  1 root root    151 May 15  2019 Makefile
-rw-rw-r--.  1 root root   6888 May 15  2019 MANIFESTO
-rw-rw-r--.  1 root root  20555 May 15  2019 README.md
-rw-rw-r--.  1 root root  61797 May 15  2019 redis.conf
-rwxrwxr-x.  1 root root    275 May 15  2019 runtest
-rwxrwxr-x.  1 root root    280 May 15  2019 runtest-cluster
-rwxrwxr-x.  1 root root    341 May 15  2019 runtest-moduleapi
-rwxrwxr-x.  1 root root    281 May 15  2019 runtest-sentinel
-rw-rw-r--.  1 root root   9710 May 15  2019 sentinel.conf
drwxrwxr-x.  3 root root   4096 May 15  2019 src
drwxrwxr-x. 11 root root   4096 May 15  2019 tests
drwxrwxr-x.  8 root root   4096 May 15  2019 utils
```
查看README.md,并按照此文件来安装redis。  
执行`make`命令，开始redis的编译，默认redis安装位置为`/usr/local/bin`。
如果想要更改此安装位置，使用编译命令 `make install PREFIX=/home/zhaoshuai/redis`。
>> 默认安装目录/usr/local/bin, 因为linux默认环境变量配置了/usr/local/bin目录，因此如果使用PREFIX指定目录的话，需要配置环境变量指定此目录。
>> 配置方式：
>> vi /etc/profile
>> 添加如下配置
>> export REDIS_HOME=/home/zhaoshuai/redis
>> export PATH=$PATH:$REDIS_HOME/bin
>> 保存并退出，刷新变量source /etc/profile

看到如下输出则表示安装成功：
```
Hint: It's a good idea to run 'make test' ;)

    INSTALL install
    INSTALL install
    INSTALL install
    INSTALL install
    INSTALL install
```
如果安装失败，则执行命令 `make distclean`，撤销上次执行命令，重新编译。
编译成功后并配置环境变量后，可以在任意目录执行 `redis-server`来启动redis实例,启动后输出如下表示启动成功：
```
[root@localhost src]# redis-server 
59669:C 20 Oct 2020 14:49:43.063 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
59669:C 20 Oct 2020 14:49:43.063 # Redis version=5.0.5, bits=64, commit=00000000, modified=0, pid=59669, just started
59669:C 20 Oct 2020 14:49:43.063 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
59669:M 20 Oct 2020 14:49:43.064 * Increased maximum number of open files to 10032 (it was originally set to 1024).
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 5.0.5 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 59669
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           http://redis.io        
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

59669:M 20 Oct 2020 14:49:43.075 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
59669:M 20 Oct 2020 14:49:43.075 # Server initialized
59669:M 20 Oct 2020 14:49:43.075 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
59669:M 20 Oct 2020 14:49:43.077 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
59669:M 20 Oct 2020 14:49:43.077 * Ready to accept connections
```
此时启动是非后台启动的，也就是当前终端窗口关闭，或按 `ctrl+c`后，那么此实例将被关闭。
## 1.2 将redis作为linux的服务启动
进入redis主目录下的utils目录下  
```
cd /usr/local/redis-5.0.5/utils
```
执行 `./install_server.sh`命令，可以看到如下内容： 
```
[root@localhost utils]# ./install_server.sh 
Welcome to the redis service installer
This script will help you easily set up a running redis server

Please select the redis port for this instance: [6379]    
```
可以看到，请为此redis实例选择一个端口号，如果不输的话，默认是6379
点击回车后，会输出如下：
```
Please select the redis config file name [/etc/redis/6379.conf] 
```
请选择此redis的配置文件，默认是/etc/redis/6379.conf,也可以手动指定目录。
后面还有其他配置，我们一直点回车使用默认配置，可以看到输出如下：
```
Please select the redis log file name [/var/log/redis_6379.log] 
Selected default - /var/log/redis_6379.log
Please select the data directory for this instance [/var/lib/redis/6379] 
Selected default - /var/lib/redis/6379
Please select the redis executable path [/usr/local/bin/redis-server] 
Selected config:
Port           : 6379
Config file    : /etc/redis/6379.conf
Log file       : /var/log/redis_6379.log
Data dir       : /var/lib/redis/6379
Executable     : /usr/local/bin/redis-server
Cli Executable : /usr/local/bin/redis-cli
Is this ok? Then press ENTER to go on or Ctrl-C to abort.
Copied /tmp/6379.conf => /etc/init.d/redis_6379
Installing service...
Successfully added to chkconfig!
Successfully added to runlevels 345!
Starting Redis server...
Installation successful!
```
至此我们已经启动了一个redis实例。可以通过命令 `ps -ef|grep redis`来查看redis的进程信息。
上面的命令 `./install_server`可以执行多次，也就是说一台物理机可以启动多个redis实例，不同的redis实例通过端口号进行区分。每个实例的配置文件等都会随着端口号而变化。
为了验证我们再重新启动一台，端口号为6380。启动日志如下：
```
[root@localhost utils]# ./install_server.sh 
Welcome to the redis service installer
This script will help you easily set up a running redis server

Please select the redis port for this instance: [6379] 6380
Please select the redis config file name [/etc/redis/6380.conf] 
Selected default - /etc/redis/6380.conf
Please select the redis log file name [/var/log/redis_6380.log] 
Selected default - /var/log/redis_6380.log
Please select the data directory for this instance [/var/lib/redis/6380] 
Selected default - /var/lib/redis/6380
Please select the redis executable path [/usr/local/bin/redis-server] 
Selected config:
Port           : 6380
Config file    : /etc/redis/6380.conf
Log file       : /var/log/redis_6380.log
Data dir       : /var/lib/redis/6380
Executable     : /usr/local/bin/redis-server
Cli Executable : /usr/local/bin/redis-cli
Is this ok? Then press ENTER to go on or Ctrl-C to abort.
Copied /tmp/6380.conf => /etc/init.d/redis_6380
Installing service...
Successfully added to chkconfig!
Successfully added to runlevels 345!
Starting Redis server...
Installation successful!
```
>> 注意上面我们在输入端口号时输入了6380，后面所有的配置文件名都随着端口号而变化。

我们打开一个实例的配置文件查看内容，`vi /etc/redis/6379.conf`。会看到很多配置内容，这些内容再后面会进行详细解释。
现在了解就好，可以再配置文件中查看到port，logfile等。
使用 `install_server`脚本后，还会将启动的redis实例注册为linux的服务。我们试着使用如下命令关闭6380的实例。
```
[root@localhost ~]# service redis_6380 stop
Stopping ...
Redis stopped
```
可以看到redis实例已经关闭，然后再使用 `ps -ef|grep redis`查看，确认redis实例已关闭。
>>> 如何将服务注册为linux服务？
>>> 想要使用service 服务名 start/stop/restart来启动服务，需要在/etc/init.d/文件夹下，新建脚本，脚本名就是服务名。
>>> 执行install_server脚本后，除了创建配置文件等外，还在/etc/init.d/文件夹下创建了启动脚本，脚本名就是redis_端口号。
>>> 打开此文件，查看内容如下,以后想要将服务注册为linux服务也可以照着写。

# redis 的数据类型
在学习redis的数据类型前，首先学习一个命令 `help`。使用命令`redis-cli -h`查看客户端的帮助信息,输入参数信息等。  
执行 `redis-cli`命令,默认连接6379端口服务。
首先我们需要了解一个知识，redis有16个数据库，这些库的名字为0-15。可以通过 `select 0`来选择库，默认为0库，不同的库之间数据是隔离的。
然后我们在学习一个命令 `help @group`，使用这个命令我们可以自己学习redis的各种命令。

````
redis存储的字节是有正反序索引的。
假设 value类型为string，值为hello，那么hello的正序索引为0,1,2,3,4，反序索引为-5,-4,-3,-2,-1
````
## string类型
使用命令 `help @string`来查看string类型的命令。根据help来学习string命令
````
string类型的encoding有三种，分别是embstr、int、bitmap
redis的key维护了当前key的encoding，在set时就会将其做出明确标记，所以在做incr等操作时，就跟根据key的encoding来判断是否支持数值操作
````
### set 添加一条数据
```shell script
127.0.0.1:6379> set k1 hello
OK
```
### append value追加
```shell script
127.0.0.1:6379> append k1 redis
(integer) 10
```
### get 根据key查询value
```shell script
127.0.0.1:6379> get k1
"helloredis"
```
### del 删除key
```shell script
127.0.0.1:6379> del k1
(integer) 1
```
del命令不只针对string类型，所有类型的数据都可以通过del key的方式删除，del后面可以跟多个key，可以一次删除多条数据。
### incr key 自增命令,每次自增1
```shell script
127.0.0.1:6379> set k1 1
OK
127.0.0.1:6379> incr k1
(integer) 2
127.0.0.1:6379> incr k1
(integer) 3
```
### incrby key increment 增加指定数字
```shell script
127.0.0.1:6379> INCRBY k1 5
(integer) 8
```
### decr key 自减，每次自减1
有增就有减
```shell script
127.0.0.1:6379> DECR k1
(integer) 7
```
### decrby key decrement 减少指定数据
```shell script
127.0.0.1:6379> DECRBY k1 4
(integer) 3
```
### incrbyfloat key increment 增加浮点数
```shell script
127.0.0.1:6379> INCRBYFLOAT k1 3.5
"6.5"
```
注意自增浮点数后，数据类型变成了 ""string类型浮点数，之前都是(integer)类型的。浮点数进行整数加减法运算结果仍为浮点数，
而incr，incrby，decr，decrby等运算结果为integer，所以进行incrbyfloat后，如果数字为浮点数，不能使用上面命令，
不过可以使用incrbyfloat key -num来减调浮点数。

### strlen key 计算值的长度
```shell script
127.0.0.1:6379> set a1 "hello"
OK
127.0.0.1:6379> STRLEN a1
(integer) 5
127.0.0.1:6379> set a2 100
OK
127.0.0.1:6379> STRLEN a2
(integer) 3
```
### setrange 设置指定偏移量的值
```shell script
127.0.0.1:6379> SETRANGE a1 5 redis
(integer) 10
127.0.0.1:6379> get a1
"helloredis"
127.0.0.1:6379> SETRANGE a1 4 " test"
(integer) 10
127.0.0.1:6379> get a1
"hell tests"
127.0.0.1:6379> 
```
需要注意当值的长度不够偏移量时，往后追加，如果设置的值长度+偏移量小于原值长度时，替换指定长度的值。
### getrange 获取指定起始位置的值，终止位置可以为负数（-1），代表从倒数，但是起始位置不能为负数。
```shell script
127.0.0.1:6379> GETRANGE a1 0 -1
"hell tests"
127.0.0.1:6379> GETRANGE a1 0 5
"hell t"
```
### setnx 设置值当key不存在时,如果key已经存在，那么设置失败。
```shell script
127.0.0.1:6379> SETNX k1 "redis"
(integer) 0
127.0.0.1:6379> SETNX k5 "redis"
(integer) 1
```
### setex 设置值的过期时间
```shell script
127.0.0.1:6379> SETEX k3 1000 "expire data"
OK
127.0.0.1:6379> TTL k3
(integer) 994
```
上面设置了一个key为k3，值为expire data的数据，设置的过期时间是1000 秒， `在redis中，过期时间单位是秒`，ttl可以查看指定key还有多久过期。
>>> 补充一个命令 expire key seconds 指定key的过期时间
127.0.0.1:6379> expire k1 10
 (integer) 1
过十秒后再查看key
127.0.0.1:6379> get k1
(nil)

### mset 设置多个值
```shell script
127.0.0.1:6379> mset b1 "hello" b2 "redis" b3 "java"
OK
```
### mget 查看多个值
```shell script
127.0.0.1:6379> mget b1 b2 b3
1) "hello"
2) "redis"
3) "java"
```
### getset 获取旧值并设置新值
```shell script
127.0.0.1:6379> GETSET k2 10
"5"
127.0.0.1:6379> get k2
"10"
```
### msetnx 设置多个值，只有当key不存在时才能设置成功，如果有一个key设置失败，那么则全都失败。
```shell script
127.0.0.1:6379> MSETNX c1 "hello" c2 "redis" c3 "python"
(integer) 1
127.0.0.1:6379> MSETNX c1 "hello" c2 "redis" b2 "python"
(integer) 0
```
### psetex 设置值，而且key将在指定毫秒数内过期。
```shell script
127.0.0.1:6379> PSETEX k1 1000 hello
OK
127.0.0.1:6379> get k1
(nil)
```
上面设置k1在1秒内过期。
### ** bitmap位图，下面的内容仍然数据string类型的命令,属于位运算。
#### redis是二进制安全的
什么是二进制安全？  
redis与外界交互的时候永远都是字节数组。面向流一般有字节流和字符流，那么当别人访问redis的时候，拿到的永远是字节流。  
为什么？  
因为如果redis只存字节，没有从字节中取出东西，按照某一编码集转换的话，数据就不会被破坏，所以叫二进制安全。
```shell script
127.0.0.1:6379> set k2 中
OK
127.0.0.1:6379> get k2
"\xe4\xb8\xad"
```
--raw触发格式化
```shell script
[root@zhaoshuai ~]# redis-cli --raw
127.0.0.1:6379> get k2
中
```
关于位图的操作：
#### setbit 设置二进制位
默认一个字节是8个二进制位，而位操作就是在这八个二进制位上进行操作。二进制只有0和1
```shell script
127.0.0.1:6379> SETBIT k1 1 1
(integer) 0
127.0.0.1:6379> SETBIT k1 7 1
(integer) 0
127.0.0.1:6379> GET k1
"A"
```
上面操作，表示将k1的值的第二位设置为1，第7位设置为1。那么此二进制就表示为：01000001，对应的ascii码就是A。那么如何将它变成B呢？
B对应的二进制码为：01000010
```shell script
127.0.0.1:6379> SETBIT k1 6 1
(integer) 0
127.0.0.1:6379> SETBIT k1 7 0
(integer) 1
127.0.0.1:6379> get k1
"B"
```
#### bitcount 统计值二进制位中1的个数，可以指定起止字符
```shell script
127.0.0.1:6379> BITCOUNT k1
(integer) 2
```
将k1的值设置为4个字符
```shell script
127.0.0.1:6379> SETBIT k1 9 1
(integer) 0
127.0.0.1:6379> SETBIT k1 15 1
(integer) 0
127.0.0.1:6379> get k1
"BA"
127.0.0.1:6379> APPEND k1 B
(integer) 4
127.0.0.1:6379> get k1
"BACB"
```
统计最后两个字符CB中的1的个数
```shell script
127.0.0.1:6379> BITCOUNT k1 2 3
(integer) 5
```
#### bitpos 寻找第一个二进制位,start和end参数为可选参数，指定字节数。
```shell script
127.0.0.1:6379> BITPOS k1 1 0 1
(integer) 1
127.0.0.1:6379> BITPOS k1 1 1 1
(integer) 9
127.0.0.1:6379> BITPOS k1 0 1 1
(integer) 8
```
#### bitop 进行位运算，位运算有与/或运算,将两个值进行位运算后赋给一个新值。
```shell script
127.0.0.1:6379> set k2 A
OK
## A的二进制码为： 01000001
127.0.0.1:6379> set k3 C
OK
## C的二进制码为： 01000011
```
所以A^C=A;A|C=C
```shell script
127.0.0.1:6379> BITOP or k4 k2 k3
(integer) 1
127.0.0.1:6379> get k4
"C"
127.0.0.1:6379> bitop and k5 k2 k3
(integer) 1
127.0.0.1:6379> get k5
"A"
```
### getbit 获取二进制位的值
```shell script
127.0.0.1:6379> get k2
"A"
127.0.0.1:6379> GETBIT k2 7
(integer) 1
127.0.0.1:6379> GETBIT k2 6
(integer) 0
```
### bitfield 对字符串任意位置进行位运算
使用bitfield时，将字符串视为位数组
~~~
待处理 todo
~~~
## list类型
list 列表，特点：  
+ 有序，双向链表存储
+ 允许重复元素
使用命令`help @list`查看命令帮助
### lpush 从左边往列表中推数据
```shell script
127.0.0.1:6379> LPUSH k1 1 2 3 4 5 6
(integer) 6
```
### lrange 截取列表的元素
```shell script
127.0.0.1:6379> LRANGE k1 0 1
1) "6"
2) "5"
```
查询所有元素：
```shell script
127.0.0.1:6379> LRANGE k1 0 -1
1) "6"
2) "5"
3) "4"
4) "3"
5) "2"
6) "1"
```
### lset 设置指定索引的元素值
```shell script
127.0.0.1:6379> lset k1 0 8
OK
127.0.0.1:6379> LRANGE k1 0 -1
1) "8"
2) "5"
3) "4"
4) "3"
5) "2"
6) "1"
```
### linsert 插入数据
```shell script
127.0.0.1:6379> LINSERT k1 before 5 7
(integer) 7
127.0.0.1:6379> LRANGE k1 0 -1
1) "8"
2) "7"
3) "5"
4) "4"
5) "3"
6) "2"
7) "1"
```
linsert 插入可以选择before或after某一个元素，也就是在指定元素之前或者之后插入数据，不是根据下标操作。
### rpush 从右边向列表中推入元素
```shell script
127.0.0.1:6379> RPUSH k2 1 2 3 4 5 6
(integer) 6
127.0.0.1:6379> LRANGE k2 0 -1
1) "1"
2) "2"
3) "3"
4) "4"
5) "5"
6) "6"
```
在列表的命令以l开头，有两种意思，一种是表示列表，如：lrange，lset。另一种则表示left，从左边操作，如lpush，lpop等。
有左就有右，分别表示从列表头或从列表尾操作列表。
### llen 统计列表元素个数
```shell script
127.0.0.1:6379> llen k1
(integer) 7
```
### lpop 从头部弹出一个元素
```shell script
127.0.0.1:6379> lpop k1
"8"
127.0.0.1:6379> lpop k1
"7"
127.0.0.1:6379> lpop k1
"5"
```
### rpop 从尾部弹出一个元素
```shell script
127.0.0.1:6379> lrange k2 0 -1
1) "1"
2) "2"
3) "3"
4) "4"
5) "5"
6) "6"
127.0.0.1:6379> RPOP k2 
"6"
127.0.0.1:6379> RPOP k2 
"5"
127.0.0.1:6379> RPOP k2 
"4"
```
### 根据lpush，rpush，lpop，rpop可以组成常见数据结构，如栈、队列
````
栈：先进后出
 127.0.0.1:6379> lpush k3 1 2 3 4 5 6
 (integer) 6
 127.0.0.1:6379> lpop k3
 "6"
 127.0.0.1:6379> lpop k3
 "5"
 127.0.0.1:6379> lpop k3
 "4"
 127.0.0.1:6379> lpop k3
 "3"
 127.0.0.1:6379> lpop k3
 "2"
 127.0.0.1:6379> lpop k3
 "1"
队列：先进先出
 127.0.0.1:6379> lpush k4 1 2 3 4 5 6
 (integer) 6
 127.0.0.1:6379> rpop k4
 "1"
 127.0.0.1:6379> rpop k4
 "2"
 127.0.0.1:6379> rpop k4
 "3"
 127.0.0.1:6379> rpop k4
 "4"
 127.0.0.1:6379> rpop k4
 "5"
 127.0.0.1:6379> rpop k4
 "6"
````
### lpushx 当列表存在时，从头部添加数据
>>> 学习一个新命令： keys [pattern] 根据指定的模式匹配key，*匹配所有   
> 127.0.0.1:6379> keys *  
> 1) "k1"
> 2) "k2"  
> 表示当前有两个key，k1，k2
```shell script
127.0.0.1:6379> LRANGE k2 0 -1
1) "1"
2) "2"
3) "3"
127.0.0.1:6379> LPUSHX k2 4 5 6
(integer) 6
127.0.0.1:6379> LPUSHX k3 1 2 3
(integer) 0
```
### rpushx 当列表存在时，从尾部插入元素
```shell script
127.0.0.1:6379> RPUSHX k3 4 5 6
(integer) 0
127.0.0.1:6379> RPUSHX k2 7 8 9
(integer) 9
127.0.0.1:6379> LRANGE k2 0 -1
1) "6"
2) "5"
3) "4"
4) "1"
5) "2"
6) "3"
7) "7"
8) "8"
9) "9"
```
### rpopLpush 弹出列表的最后一个元素，并将它从头部插入新的列表。
```shell script
127.0.0.1:6379> RPOPLPUSH k2 k4
"9"
127.0.0.1:6379> LRANGE k4 0 -1
1) "9"
127.0.0.1:6379> RPOPLPUSH k2 k4
"8"
127.0.0.1:6379> LRANGE k4 0 -1
1) "8"
2) "9"
127.0.0.1:6379> RPOPLPUSH k2 k4
"7"
127.0.0.1:6379> LRANGE k4 0 -1
1) "7"
2) "8"
3) "9"
127.0.0.1:6379> LRANGE k2 0 -1
1) "6"
2) "5"
3) "4"
4) "1"
5) "2"
6) "3"
```
### lindex 根据索引查询元素
```shell script
127.0.0.1:6379> LINDEX k2 0
"6"
127.0.0.1:6379> LINDEX k2 4
"2"
```
### ltrim 将列表修剪到指定范围
这个命令和lrange看起来很像，lrange命令，截取指定索引的字符串，例如：
```shell script
127.0.0.1:6379> LRANGE k2 0 3
1) "6"
2) "5"
3) "4"
4) "1"
```
返回指定范围的数据，但是列表k2的长度并没有改变
```shell script
127.0.0.1:6379> lrange k2 0 -1
1) "6"
2) "5"
3) "4"
4) "1"
5) "2"
6) "3"
```
ltrim命令则会保留指定的返回的元素，删除范围之外的元素,执行结果输出为成功或失败。
```shell script
127.0.0.1:6379> LTRIM k2 2 4
OK
127.0.0.1:6379> LRANGE k2 0 -1
1) "4"
2) "1"
3) "2"
```
可以看出ltrim会修改列表的长度。ltrim和lrange两个命令的输出也是不一样的，lrange输出的是截取返回的元素值，而ltrim返回截取成功或失败。
### lrem 删除元素，指定元素的值及删除个数

```shell script
127.0.0.1:6379> lpush k5 1 2 3 2 1 2 2 1 3 2
(integer) 10
127.0.0.1:6379> LRANGE k5 0 -1
 1) "2"
 2) "3"
 3) "1"
 4) "2"
 5) "2"
 6) "1"
 7) "2"
 8) "3"
 9) "2"
10) "1"   
```
下面我们删掉3个2，会从左边开始统计，删除三个2结束
```shell script
127.0.0.1:6379> LREM k5 3 2
(integer) 3
127.0.0.1:6379> LRANGE k5 0 -1
1) "3"
2) "1"
3) "1"
4) "2"
5) "3"
6) "2"
7) "1"
```
再删除一个1
```shell script
127.0.0.1:6379> LRANGE k5 0 -1
1) "3"
2) "1"
3) "2"
4) "3"
5) "2"
6) "1"
```
lrem输出的是删除的元素个数，当元素不存在时，输出0，列表没变化
```shell script
127.0.0.1:6379> LREM k5 1 4
(integer) 0
127.0.0.1:6379> LRANGE k5 0 -1
1) "3"
2) "1"
3) "2"
4) "3"
5) "2"
6) "1"
```
当要删除的元素个数超过列表中的元素个数时，则删除所有并返回删除的元素个数
```shell script
127.0.0.1:6379> LREM k5 4 3
(integer) 2
127.0.0.1:6379> LRANGE k5 0 -1
1) "1"
2) "2"
3) "2"
4) "1"
```
### blpop,brpop,brpoplpush 阻塞的单播队列
启动三个客户端redis-cli,打开一个查看
```shell script
127.0.0.1:6379> keys *
1) "k4"
2) "k1"
3) "k2"
4) "k5"
```
查看没有k6的键，两个客户端输入
```shell script
127.0.0.1:6379> BLPOP k6 0

```
```shell script
127.0.0.1:6379> BLPOP k6 0

```
会阻塞
然后在另一台上往k6中推入一个元素
```shell scriptcl
127.0.0.1:6379> LPUSH k6 1
(integer) 1
```
查看阻塞的两个客户端
```shell script
127.0.0.1:6379> BLPOP k6 0
1) "k6"
2) "1"
(47.36s)
127.0.0.1:6379> 
```
```shell script
127.0.0.1:6379> BLPOP k6 0

```
一台客户端结束阻塞，另一个仍在阻塞
## set类型
list是列表，set是集合。
set的特征：
+ 无序
+ 去重
使用命令 `help @set`查看set命令的帮助文档
### sadd 添加集合
```shell script
127.0.0.1:6379> sadd k1 a b c d e 
(integer) 5
```
### smemebers 查看所有元素
```shell script
127.0.0.1:6379> SMEMBERS k1
1) "c"
2) "d"
3) "b"
4) "a"
5) "e"
```
### scard 获取集合元素总数（集合长度）
```shell script
127.0.0.1:6379> scard k1
(integer) 5
```
### srandmember key [count] 随机获取count个元素
随机获取元素，但是元素仍在集合中，并没有取出来
```shell script
127.0.0.1:6379> srandmember k1 
"c"
127.0.0.1:6379> srandmember k1 3
1) "a"
2) "d"
3) "e"
127.0.0.1:6379> SMEMBERS k1
1) "d"
2) "b"
3) "a"
4) "c"
5) "e"
```
### sdiff key [key1, key2..] 取多个集合的差集
从参数可以看出，查询的是key与 key1，key2..的差集
```shell script
127.0.0.1:6379> SADD k2 b c d e f
(integer) 5
127.0.0.1:6379> SDIFF k1 k2
1) "a"
```
### sdiffstore destination key [key1, key2..] 取差集并存入目标集合destination
```shell script
127.0.0.1:6379> sadd k3 c d e f g
(integer) 5
127.0.0.1:6379> SDIFFSTORE k4 k1 k3
(integer) 2
127.0.0.1:6379> SMEMBERS k4
1) "b"
2) "a"
127.0.0.1:6379> SDIFFSTORE k5 k3 k1
(integer) 2
127.0.0.1:6379> SMEMBERS k5
1) "f"
2) "g"
```
### sinter key [key1,key2..] 取key与多个集合的交集
```shell script
127.0.0.1:6379> SINTER k1 k2 k3
1) "c"
2) "d"
3) "e"
127.0.0.1:6379> SINTER k1 k2
1) "b"
2) "c"
3) "d"
4) "e"
```
### sinterstore destination key [key1,key2..] 取多个集合的交集并存入新的集合
```shell script
127.0.0.1:6379> SINTERSTORE k6 k1 k2 k3
(integer) 3
127.0.0.1:6379> SMEMBERS k6
1) "c"
2) "d"
3) "e"
```
### sunion key [key1,key2..] 取多个集合的并集
```shell script
127.0.0.1:6379> SUNION k1 k2
1) "b"
2) "a"
3) "f"
4) "c"
5) "d"
6) "e"
127.0.0.1:6379> SUNION k1 k2 k3
1) "b"
2) "a"
3) "f"
4) "c"
5) "g"
6) "d"
7) "e"
```
### sunionstore destination key [key1, key2..] 取多个集合的并集存入新的集合
```shell script
127.0.0.1:6379> SUNIONSTORE k7 k1 k2 k3
(integer) 7
127.0.0.1:6379> SMEMBERS k7
1) "b"
2) "a"
3) "f"
4) "c"
5) "g"
6) "d"
7) "e"
```
### sismember key member 判断集合中是否存在成员
```shell script
127.0.0.1:6379> SMEMBERS k1
1) "b"
2) "a"
3) "c"
4) "d"
5) "e"
127.0.0.1:6379> SISMEMBER k1 a
(integer) 1
127.0.0.1:6379> SISMEMBER k1 g
(integer) 0
```
### smove source destination member 移动source中的成员member到一个新的集合destination
```shell script
127.0.0.1:6379> SMEMBERS k4
1) "b"
2) "a"
127.0.0.1:6379> SMOVE k1 k4 c
(integer) 1
127.0.0.1:6379> SMEMBERS k4
1) "c"
2) "b"
3) "a"
127.0.0.1:6379> SMEMBERS k1
1) "b"
2) "a"
3) "d"
4) "e"
```
### spop key [count] 移除[count,默认1]个元素
```shell script
127.0.0.1:6379> SPOP k1 3
1) "b"
2) "a"
3) "d"
127.0.0.1:6379> SMEMBERS k1
1) "e"
127.0.0.1:6379> SMEMBERS k2
1) "b"
2) "c"
3) "f"
4) "d"
5) "e"
127.0.0.1:6379> SPOP k2
"d"
127.0.0.1:6379> SMEMBERS k2
1) "b"
2) "c"
3) "f"
4) "e"
```
### srem key member [member...] 删除指定元素
```shell script
127.0.0.1:6379> SREM k2 b c e
(integer) 3
127.0.0.1:6379> SMEMBERS k2
1) "f"
```
## sorted_set类型
sorted_set和set的区别？  
之前已经有了一个Set类型，为什么还要有一个sorted_set类型呢？这两个类型的具体区别：  
+ set是无序的，sorted_set是有序的，但是要注意这个有序是排序，与list不同，list的有序是指输入顺序与输出顺序一致，也就是存储的顺序。
而sorted_set每个元素都有一个score分值，根据这个分值对元素进行排序，因此通过更改分值就可以更改元素的位置。

使用命令 `help @sort_set`来查看帮助信息
### zadd key [nx|xx] [ch] [incr] score member[score member..] 新增命令
[nx|xx]: nx: 当member成员不存在时，插入成员信息
         xx: 当member成员存在时，更新成员分数
将班级学生按照语文成绩进行排序，那么这个集合应该是这样的
```shell script
127.0.0.1:6379> zadd k1 100 zhangsan 80 lisi 90 wangwu 50 maliu
(integer) 4
127.0.0.1:6379> ZADD k1 nx 75 xiaoming
(integer) 1
127.0.0.1:6379> ZADD k1 nx 84 lisi
(integer) 0
```
### zrange key start end [withsocres] 获取值
```shell script
127.0.0.1:6379> ZRANGE k1 0 -1 withscores
 1) "maliu"
 2) "50"
 3) "xiaoming"
 4) "75"
 5) "lisi"
 6) "80"
 7) "wangwu"
 8) "90"
 9) "zhangsan"
10) "100"
127.0.0.1:6379> ZRANGE k1 0 -1
1) "maliu"
2) "xiaoming"
3) "lisi"
4) "wangwu"
5) "zhangsan"
```
> 学习这个命令后再回头看zadd命令的xx命令
```shell script
 127.0.0.1:6379> ZADD k1 xx 70 maliu
 (integer) 0
 127.0.0.1:6379> ZRANGE k1 0 -1 withscores
  1) "maliu"
  2) "70"
  3) "xiaoming"
  4) "75"
  5) "lisi"
  6) "80"
  7) "wangwu"
  8) "90"
  9) "zhangsan"
 10) "100"
```
### zrangebyscore 上面是按照索引取，还可以按照分数
```shell script
127.0.0.1:6379> ZRANGEBYSCORE k1 0 100 withscores
 1) "xiaoming"
 2) "75"
 3) "lisi"
 4) "80"
 5) "wangwu"
 6) "90"
 7) "maliu"
 8) "100"
 9) "zhangsan"
10) "100"
127.0.0.1:6379> ZRANGEBYSCORE k1 75 90
1) "xiaoming"
2) "lisi"
3) "wangwu"
```
### zcard 获取元素总数
```shell script
127.0.0.1:6379> ZCARD k1
(integer) 5
```
### zcount key min max 获取某一个分数范围内的元素总数
```shell script
127.0.0.1:6379> ZCOUNT k1 50 80
(integer) 3
```
### zincyby key increment member 某一个成员增加分数
```shell script
127.0.0.1:6379> ZINCRBY k1 30 maliu
"100"
127.0.0.1:6379> ZRANGE k1 0 -1 withscores
 1) "xiaoming"
 2) "75"
 3) "lisi"
 4) "80"
 5) "wangwu"
 6) "90"
 7) "maliu"
 8) "100"
 9) "zhangsan"
10) "100"
```
### zrevrange 倒序按索引取数
```shell script
127.0.0.1:6379> ZREVRANGE k1 0 -1 withscores
 1) "zhangsan"
 2) "100"
 3) "maliu"
 4) "100"
 5) "wangwu"
 6) "90"
 7) "lisi"
 8) "80"
 9) "xiaoming"
10) "75"
```
### zrevrangebyscore 倒序按分数取数
```shell script
127.0.0.1:6379> ZREVRANGEBYSCORE k1 90 40
1) "wangwu"
2) "lisi"
3) "xiaoming"
```
### zscore 获取成员的分数
```shell script
127.0.0.1:6379> ZSCORE k1 zhangsan
"100"
```
### zrank 确定成员在排序集中的索引
```shell script
127.0.0.1:6379> ZRANGE k1 0 -1
1) "xiaoming"
2) "lisi"
3) "wangwu"
4) "xiaohong"
5) "zhaosi"
6) "maliu"
7) "zhangsan"
127.0.0.1:6379> ZRANK k1 xiaoming
(integer) 0
127.0.0.1:6379> ZRANK k1 lisi
(integer) 1
127.0.0.1:6379> ZRANK k1 zhangsan
(integer) 6
```
### zrevrank 确定成员在拍序集中的索引，排序集为倒序排序
```shell script
127.0.0.1:6379> ZREVRANK k1 zhangsan
(integer) 0
127.0.0.1:6379> ZREVRANK k1 xiaoming
(integer) 6
127.0.0.1:6379> ZREVRANK k1 maliu
(integer) 1
```
### zrem 移除指定成员
```shell script
127.0.0.1:6379> ZREM k1 xiaohong
(integer) 1
127.0.0.1:6379> ZRANGE k1 0 -1
1) "xiaoming"
2) "lisi"
3) "wangwu"
4) "zhaosi"
5) "maliu"
6) "zhangsan"
```
### zunionStore 两个排序集取并集并存入新的排序集
```shell script
127.0.0.1:6379> zadd k2 1 a 2 b 3 c
(integer) 4
127.0.0.1:6379> zadd k3 5 b 3 c 2 d
(integer) 4
127.0.0.1:6379> ZUNIONSTORE k4 2 k2 k3 
(integer) 4
127.0.0.1:6379> ZRANGE k4 0 -1
1) "a"
2) "c"
3) "d"
4) "b"
127.0.0.1:6379> ZRANGE k4 0 -1 withscores
1) "a"
2) "1"
3) "d"
4) "2"
5) "c"
6) "6"
7) "b"
8) "7"
```
两个集合取并集时，如果有相同的元素，会重新计算分值，计算规则有 min/max/sum，可以计算分值的权重
### zinterstore 取交集
```shell script
127.0.0.1:6379> ZINTERSTORE k5 2 k2 k3
(integer) 2
127.0.0.1:6379> ZRANGE k5 0 -1
1) "c"
2) "b"
```
### zpopmax 弹出分数最高的成员
```shell script
127.0.0.1:6379> ZRANGE k1 0 -1
 1) "a"
 2) "b"
 3) "c"
 4) "d"
 5) "xiaoming"
 6) "lisi"
 7) "wangwu"
 8) "zhaosi"
 9) "maliu"
10) "zhangsan"
127.0.0.1:6379> ZPOPMAX k1 2
1) "zhangsan"
2) "100"
3) "maliu"
4) "100"
```
### zpopmin 弹出分数最低的成员
```shell script
127.0.0.1:6379> ZPOPMIN k1 3
1) "a"
2) "1"
3) "b"
4) "2"
5) "c"
6) "3"
127.0.0.1:6379> ZRANGE k1 0 -1 withscores
 1) "d"
 2) "4"
 3) "xiaoming"
 4) "75"
 5) "lisi"
 6) "80"
 7) "wangwu"
 8) "90"
 9) "zhaosi"
10) "90"
```
### zlexcount key min max

>> sorted_set 使用skip_list跳表做存储保证增删改的速度
>
## hash类型
hash用来存放键值对，类似于java中的hashmap
使用命令 `help @hash`查看hash相关的命令
### hset key field value 设置hash类型的值
```shell script
127.0.0.1:6379> HSET h1 name zhangsan
(integer) 1
127.0.0.1:6379> HSET h1 age 18
(integer) 1
```
### hgetall key 查看所有的键值对
```shell script
127.0.0.1:6379> HGETALL h1
1) "name"
2) "zhangsan"
3) "age"
4) "18"
```
### hget key field 根据key获取value
```shell script
127.0.0.1:6379> hget h1 name
"zhangsan"
```
### hlen key 获取总的键值对数
```shell script
127.0.0.1:6379> HLEN h1
(integer) 2
```
### hkeys key 获取所有的key(keySet)
```shell script
127.0.0.1:6379> HKEYS h1
1) "name"
2) "age"
```
### hvals key 获取所有的值（valueSet）
```shell script
127.0.0.1:6379> HVALS h1
1) "zhangsan"
2) "18"
```
### hincrby key field increment 指定key对应的value增加指定数字
```shell script
 127.0.0.1:6379> HINCRBY h1 age 2
 (integer) 20
```
### hincrbyfloat key field increment 增加float浮点数
```shell script
127.0.0.1:6379> HINCRBYFLOAT h1 age 2.5
"22.5"
```
### hstrlen 值的长度
```shell script
127.0.0.1:6379> HSTRLEN h1 name
(integer) 8
```
### hsetnx 当key不存在时设置key，value，否则设置失败
```shell script
127.0.0.1:6379> HSETNX h1 name lisi
(integer) 0
127.0.0.1:6379> HSETNX h1 gender man
(integer) 1
```
### hexists key field 查询指定key是否存在
```shell script
127.0.0.1:6379> HEXISTS h1 name
(integer) 1
127.0.0.1:6379> HEXISTS h1 hobby
(integer) 0
```
### hmset 设置多个值
```shell script
127.0.0.1:6379> HMSET h2 yuwen 100 shuxu 90 yingyu 60
OK
127.0.0.1:6379> HGETALL h2
1) "yuwen"
2) "100"
3) "shuxu"
4) "90"
5) "yingyu"
6) "60"
```
### hmget 获取多个值
```shell script
127.0.0.1:6379> HMGET h2 yuwen shuxu yingyu 
1) "100"
2) "90"
3) "60"
```

# redis事务
redis是单线程的，所以是线程安全，redis对事务的支持仅仅支持原子性，也就是要么都执行，要么都不执行，不支持回滚事务。
使用help命令学习redis事务。 `help @transactions`
### multi 开启事务
```shell script
127.0.0.1:6379> multi
OK
```
如果使用redis事务，就要使用此命令开启事务，开启事务后可以输入redis存数据的命令
```shell script
127.0.0.1:6379> set s1 "hello"
QUEUED
127.0.0.1:6379> set s2 "hello"
QUEUED
```
可以看到开启事务后，命令全都存入了队列中，并没有被执行。
### exec 提交事务，执行队列中的所有redis命令
```shell script
127.0.0.1:6379> exec
1) OK
2) OK
```
### discard 丢弃当前队列中的所有命令
```shell script
127.0.0.1:6379> keys *
(empty list or set)
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> set k1 "hello"
QUEUED
127.0.0.1:6379> set k2 "java"
QUEUED
127.0.0.1:6379> DISCARD
OK
127.0.0.1:6379> exec
(error) ERR EXEC without MULTI
```
执行exec命令，报错没有可以执行的事务。因为discard将队列中的命令丢弃了。
>>> redis事务演示：
~~~
127.0.0.1:6379> FLUSHALL
OK
127.0.0.1:6379> keys *
(empty list or set)
127.0.0.1:6379> set k1 "hello"
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> SETNX k2 "a"
QUEUED
127.0.0.1:6379> SETNX k1 "b"
QUEUED
127.0.0.1:6379> exec
(error) EXECABORT Transaction discarded because of previous errors.
127.0.0.1:6379> keys *
1) "k1"
~~~
清空所有的key值，然后插入一个k1，值为hello，然后开启事务，插入k2，在setnx k1，此时已经k1已经有值了，因此这条命令会失败，然后exec提交事务
查询所有的key，发现在k1前的k2也没有插入成功，证明事务的原子性。
### watch 监控一个key
watch命令就像java的乐观锁一样，使用watch后会监控一个key，当这个key的值发生改变时，那么事务就会提交失败。
开启两台redis客户端。client1，client2：
client1：
```shell script
127.0.0.1:6379> set k1 1
OK
127.0.0.1:6379> WATCH k1
OK
127.0.0.1:6379> MULTI 
OK
127.0.0.1:6379> set k2 b
QUEUED
127.0.0.1:6379> set k3 c
QUEUED
127.0.0.1:6379> 
```
在client2中对k1的值做更改。此时client1还未提交事务。
client2：
```shell script
127.0.0.1:6379> INCR k1
(integer) 2
```
然后在client1中提交事务：
```shell script
127.0.0.1:6379> EXEC
(nil)
127.0.0.1:6379> keys *
1) "k1"
```
发现client1的事务提交失败。并没有插入数据成功。
### unwatch 取消对key的监控
a
# redis的订阅服务
在学习list时，学习了单播阻塞队列。多个客户端阻塞等待一个列表中的元素，当有列表中有元素时，只会有一个客户端取出数据，其他客户端仍然阻塞。
订阅就是当往列表中添加元素时，多个阻塞客户端都能收到数据。
类似聊天群，一个人发了信息，其他所有人都可以收到信息。  
使用help命令查看订阅服务的命令 `help @pubsub`
### subscribe 监听通道中的消息。
开启多个客户端，监听c1通道。
client1:
```shell script
1) "subscribe"
2) "c1"
3) (integer) 1

```
client2:
```shell script
1) "subscribe"
2) "c1"
3) (integer) 1

```
### publish 往通道中推送消息
新启一个客户端
```shell script
127.0.0.1:6379> PUBLISH c1 hello
(integer) 2
127.0.0.1:6379> PUBLISH c1 java
(integer) 2
127.0.0.1:6379> PUBLISH c1 "my channel message"
(integer) 2
```
然后查看其他的阻塞客户端：
client1：
```shell script
127.0.0.1:6379> SUBSCRIBE c1
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "c1"
3) (integer) 1
1) "message"
2) "c1"
3) "hello"
1) "message"
2) "c1"
3) "java"
1) "message"
2) "c1"
3) "my channel message"
```
client2:
```shell script
127.0.0.1:6379> SUBSCRIBE c1
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "c1"
3) (integer) 1
1) "message"
2) "c1"
3) "hello"
1) "message"
2) "c1"
3) "java"
1) "message"
2) "c1"
3) "my channel message"
```
可以看出，两个阻塞的客户端都收到了信息
### unsubscribe 取消监听
### psubscribe 监听所有匹配指定模式通道的信息
### punsubscribe 取消监听

# redis扩展库
进入 [redis官网](https://redis.io/modules)  
这个目录下都是redis的扩展库，以bloom扩展库为例，找到此拓展库，进入主页，就跳转进github的主页了。  
然后复制clone标签下的zip文件的链接。进入redis服务器主目录，执行 `wget 下载链接`就能下载到压缩包master.zip  
使用 `unzip master.zip`解压，然后进入目录，使用make编译。编译后的到.so文件。
然后修改redis启动的配置文件，添加loadmodule 刚编译得到的.so文件路径。
然后重启redis实例。查看日志:
```shell script
...
16568:M 24 Oct 2020 14:25:34.613 * Module 'bf' loaded from /usr/local/redis-5.0.5/RedisBloom-master/redisbloom.so
16568:M 24 Oct 2020 14:25:34.613 * DB loaded from disk: 0.000 seconds
16568:M 24 Oct 2020 14:25:34.613 * Ready to accept connections
```
可以看到加载了bf模块，就是bloom。
进入redis客户端，就可以使用这个模块的命令了。
bloom模块解决了一个问题：**缓存穿透**
## 什么是缓存穿透？
redis作为缓存时，用来存一些经常被查询的数据，那么当请求过来后，查询的内容没有时，就会到数据库中查询，然后再将数据库中的数据放入redis中一份，用来
防止减轻数据库的压力。但是在特殊情况--数据库查到的结果也是空。那么这种情况下，如果这个请求的并发量比较大的话(正常情况/恶意攻击)。那么缓存就会失效
所有的请求就会压到数据库层，导致数据库宕机。
** 缓存和数据库都没有数据 **
### 缓存穿透的解决办法：
以搜索某商品为例：
* 将所有的商品名都存入缓存，当请求过来时，首先查询redis中是否有此商品，找到的话，直接返回，没找到的话，提示没有。
* 当redis中没有找到数据时，查询数据库，如果数据库搜索结果也为空，那么将这个搜索的关键字存入一个null到redis中，并设置过期时间。
### 缓存击穿？
**缓存中没有数据，但是数据库中有** 
可能是缓存key值过期了，而正好这个key的并发查询比较大，那么就会导致所有的请求全都压到数据库层，导致数据库压力过大
### 缓存击穿解决办法：
* 设置热点数据永不过期
* 对查询数据库的操作加锁，dcl判断redis中是否有数据。
### 缓存雪崩？
**缓存击穿是一个key过期，这个key并发量大引起，而缓存雪崩是大量的key同时过期，导致数据库压力突增**
## 解决：
* 所有的key过期时间设置为随机
* 热点数据永不过期
* 将数据均匀分布多台服务器上

# redis缓存LRU
redis可以用来做缓存，也可以用来做数据库。
### 为什么要用redis做缓存？
因为redis的特性就是快，用来减少数据库的压力，所以redis中应该放的是热点数据。
### 缓存的特点：
* 快，缓存的数据是在内存中的，内存的速度要比磁盘快很多，这样就可以降低io的耗时，提高速度。
* 数据易丢失，因为缓存中的数据是放在内存中的，内存中的数据如果系统发生异常或服务器宕机就会丢失。
所以根据缓存的特性，我们数据都是全量存放在数据库中的，使用缓存来存放数据的副本，降低数据库的压力，因此缓存中的数据并"不重要"（允许数据出现不一致），
但是最终数据库中的数据一定要是准确的。
### 使用redis做缓存需要注意的地方--redis只能存放热点数据？
1. redis做缓存时，缓存中的热点数据并不是一成不变的，今天的访问量比较高的数据，可能明天就变了，以后都不会再访问了，那么就需要清理这些冷数据。
2. 当redis中存放的数据量过大时，就要淘汰部分数据，用来给新的数据腾位置
为什么？  
因为内存的大小时固定的，使用内存可以解决io的瓶颈，但是自身的大小也是一个瓶颈。在看上面两种情况，一种是可以明确确定，那儿些数据是过期的，应该扔掉
但是第二种情况，因为内存空间引起，导致没有空间存放新的数据，这时，并不能确定到底那儿些数据时没用的，可以清理的。那么就只能随机选取数据清理掉，为新的
数据腾位置。
### 当redis的内存满了之后怎么办？
1. 打开redis的配置文件可以查到下面几个配置：
```text
# 设置redis实例的最大内存大小
maxmemory <bytes> 
# redis到达最大内存限制时的删除策略
maxmemory-policy noeviction 
# 上面策略每次比较的key的个数
maxmemory-samples 5
```
关于maxmemory-policy配置，查看配置文件此配置的注释，可以看到如下内容：
~~~
MAXMEMORY POLICY: how Redis will select what to remove when maxmemory
is reached. You can select among five behaviors:

volatile-lru -> Evict using approximated LRU among the keys with an expire set.
allkeys-lru -> Evict any key using approximated LRU.
volatile-lfu -> Evict using approximated LFU among the keys with an expire set.
allkeys-lfu -> Evict any key using approximated LFU.
volatile-random -> Remove a random key among the ones with an expire set.
allkeys-random -> Remove a random key, any key.
volatile-ttl -> Remove the key with the nearest expire time (minor TTL)
noeviction -> Don't evict anything, just return an error on write operations.

LRU means Least Recently Used
LFU means Least Frequently Used

Both LRU, LFU and volatile-ttl are implemented using approximated
randomized algorithms.

Note: with any of the above policies, Redis will return an error on write
      operations, when there are no suitable keys for eviction.

      At the date of writing these commands are: set setnx setex append
      incr decr rpush lpush rpushx lpushx linsert lset rpoplpush sadd
      sinter sinterstore sunion sunionstore sdiff sdiffstore zadd zincrby
      zunionstore zinterstore hset hsetnx hmset hincrby incrby decrby
      getset mset msetnx exec sort
~~~
从上面的注释内容可以了解到，此配置的策略有如下选择：
>> 在看策略之前首先对LRU、LFU做一个解释： 上面的注释文档中也对这两个定义做了解释
> LRU 最近最少使用
> LFU 最少使用
> 同时在上面的策略中还能看到两个前缀：volatile、allkeys
> volatile：表示快要过期的key
> allkeys：表示所有的key

当redis的内存空间占满时，就会触发下面的策略：
* volatile-lru    ： 也就是说移除设置有过期时间的key中最近最少使用的
* allkeys-lru     ： 移除所有key中最近最少使用的
* volatile-lfu    ： 移除设置有过期时间的key中最少使用的
* allkeys-lfu     ： 移除所有key中最少使用的
* volatile-random ： 随机移除设置有过期时间的key
* allkeys-random  ： 随机移除所有的key
* volatile-ttl    ： 删除最接近到期时间的(快到期的)
* noeviction      ： 不删除任何的key，返回客户端错误
>查看上面的策略进行分析：
>如果redis作为数据库的话，那么只能使用noevication，因为要保证数据不回丢失，当内存满的时候，宁愿报错，通知开发人员，空间不足。也不能丢弃现在已有的数据。
>但是如果redis作为缓存的话，那么就可以选择其他的策略了。在实际应用中怎么进行选择？  
>首先看*-random这两个策略，这两个策略都是随机选择淘汰，因此可能出现一个热点数据刚存入就又被删除了，有点太随意了。volatile-ttl这种方式，它会查询
>所有key的过期时间，然后在通过比较删除最接近到期时间的，时间复杂度比较高。排除这些后，剩下的，就是两种情况 lru/lfu。根据需要选择一种策略后，再看
>volatile或allkeys，如果设置过期时间的key比价多就选择volatile，如果没有设置过期时间的key多的话，就选择allkeys。
### 关于redis的key过期原理
redis的key过期有两种方式，主动和被动。
* 被动：当redis的一个key过期后，如果一直没有新的请求访问这个key，那么这个key可能并不会被删除，redis一直访问他，也不管它，那么他就一直占着空间
不释放，当有用户访问它，然后发现这个key已经过期了，那么就会返回给客户端没有，并且清除它，这样就会有一个时间差，如果一个月一年不访问，那么他就会
一直占着空间。
* 主动：主动方式就代表着轮询。如果redis中有十万个key，那么就要遍历十万次，每个key都看一眼，就会阻塞响应客户端，影响性能。因此redis是间接式
主动遍历，具体就是redis每秒检测十次，每次随机抽20个key查看是否过期并删除已经过期的key，如果有多于25%的key过期，那么就重复上面的过程，直到过期key
低于25%，这意味着在任何时刻，最多删除25%的key。

# redis持久化
redis可以用来做缓存，也可以用来做数据库，做缓存时，数据不一定要求必须可靠，丢了就丢了，后边还会有一个mysql数据库存数据，无非就是重新在往缓存里
放一遍，但是做数据库时候，数据是绝对不能丢的。要保证数据的可靠性。如果作为数据库使用的话，就会引出一个问题：持久化  
为什么要持久化？  
因为redis是内存型数据库，内存的特点就是数据易丢失。
只要是与持久化，数据可靠性有关的，无关技术，无论是mysql还是redis等，只要是存储层技术都会有一个通用的知识点： 快照（副本）、日志  
* 快照： 将数据库中的全量数据取出来，写在一个文件中，要么放在硬盘，要么放在其他的服务器（异地可靠性存储）这样即使服务器硬件坏了，也可以通过读取快照
获取最近一次保存的数据。
* 日志： 当用户做写操作（增删改）的时候，就会将命令记录到日志中，只要日志足够完整，即使数据全丢失了，也可以通过执行日志中的所有命令，来恢复数据。

在redis中，快照这种方式叫做RDB，日志就叫做AOF
## RDB(快照)
在上面已经解释了什么是快照，使用快照存储数据就会牵扯出一个知识点：**数据的时点性**。  
### 什么是数据的时点性？
比如说：我现在每隔一个小时，数据落地一次(做一次快照存储)，那么现在我8点的时候，要进行一次快照，那么应该怎么实现呢？
方式一： save，阻塞当前可用服务，只进行数据存储的操作。  
方式二： bgsave，使用异步的方式在后台进行数据落地，不影响服务的使用。  
上面两种方式进行比较，很明显我们应该使用的是第二种方式--为了保证服务的可用性。但是第一种方式仍然是有必要的，比如说我明确知道我现在要对某一个服务器
扩容，或者除尘...那么就要将当前服务器停机，就需要手动的进行save操作，停用当前服务并保存数据。  
回到开始的问题，什么是数据的时点性？我们假设现在有10g的数据要进行快照存储，那么存储的时间假设需要半个小时，8点开始存储，八点半结束。如何保证我八点半
保存的数据就是八点整那个时刻数据库中的数据？  
就是说，假设我redis库中本来有个a=3，那么八点开始保存数据，在保存数据的过程中，也就是八点半之前，我又将a的值改为了a=8，那么我这次存储到本地磁盘的
a的值应该是3还是8？**这就是数据的时点性**  
### 数据的时点性问题如何解决？
为了方便理解，首先需要知道一个父子进程的概念。在linux服务器中，有一个管道的概念，就是使用了父子进程。
在linux中输入 `echo $$`可以打印当前进程的id。也就是我们建立这个shell连接的进程id。
```shell script
[root@zhaoshuai ~]# echo $$
56192
```
可以看到当前进程的id是56192。当使用管道 `|`的时候，其实左边和右边都是一个子进程，｜左边的输出会作为｜右边进程的输入。
那么进行测试，输入 `echo $$|more`,分析这个命令，左边会输出当前进程id，然后作为右边的数据，右边进程就会将这个进程id输出出来。执行结果如下：
```shell script
[root@zhaoshuai ~]# echo $$|more
56192
```
我们发现执行结果并没有变化，仍然是父进程的id。那是否推翻了上面的说法呢？  
与`$$`作用相同的还有一个环境变量`$BASHPID`,也表示当前进程的id。
```shell script
[root@zhaoshuai ~]# echo $BASHPID
56192
```
发现 `$BASHPID`的作用和 `$$`是一样的,使用 `$BASHPID`再次进行上面的测试。
```shell script
[root@zhaoshuai ~]# echo $BASHPID|more
56327
[root@zhaoshuai ~]# echo $BASHPID|more
56329
[root@zhaoshuai ~]# echo $BASHPID|more
56331
```
可以发现，每次打印的结果都是不一样的，说明每次左边的进程都是不一样的。为了验证 `|`两边每次的进程都是新的，我们编辑一个脚本进行测试。
```shell script
[root@zhaoshuai ~]# vi testpid.sh
```
在脚本中输入下面内容：
```shell script
#!/bin/bash
read a
echo left:$a
echo right:$BASHPID
```
将左边的输出作为右边的输入，赋给变量a，打印a和当前进程id，然后我们为这个脚本授权。  
````shell script
[root@zhaoshuai ~]# chmod +x testpid.sh 
````
执行这个脚本
```shell script
[root@zhaoshuai ~]# echo $BASHPID|./testpid.sh 
left:56397
right:56398
[root@zhaoshuai ~]# echo $BASHPID|./testpid.sh 
left:56399
right:56400
```
我们发现每次的结果都是不一样的，也就证明了上面的理论： `|`两边会各起两个自线程。  
那么为什么使用 `$$`的时候不行呢？  
因为 `$$`的优先级高于管道，因此优先执行了 `echo $$`，此时还没有执行管道，没有开辟新的进程，因此输出的是父进程的id，然后才执行 `|`开辟两个
新进程，因此输出还是父进程的id。
### 通过上面的学习，了解了父子进程。那么父子进程之间的数据是否可以互相访问？
首先在父进程中创建一个环境变量：
```shell script
[root@zhaoshuai ~]# export num=1
[root@zhaoshuai ~]# echo $num
1
```
使用管道在子进程中输出num：
```shell script
[root@zhaoshuai ~]# echo $num|more
1
```
可以看到子进程可以访问父进程的数据,那么子进程对数据做修改父进程是否能看到呢？。
学习一个自增的操作 `((num++))`
```shell script
[root@zhaoshuai ~]# ((num++))
[root@zhaoshuai ~]# echo $num
2
```
然后我们使用管道在子进程中对num进行++，查看父进程中变量是否被修改？
```shell script
[root@zhaoshuai ~]# ((num++))|echo $num
2
[root@zhaoshuai ~]# echo $num
2
```
左边子进程对变量进行了++操作，右边输出仍是2，两个子进程数据隔离的，然后父进程重新打印变量的值不变，说明子进程对变量的修改，父进程是看不到的。
那么父进程对变量的修改，子进程是否能看到？  
为了验证上面的问题，我们编写一个脚本：
```shell script
[root@zhaoshuai ~]# vi testdata.sh 

#!/bin/bash
echo old:$num
echo "waiting parent modify num"
sleep 20
echo new:$num
[root@zhaoshuai ~]# chmod +x testdata.sh 
```
然后使用后台运行的方式执行testdata.sh,方便在等待时间对父进程变量进行修改。
```shell script
[root@zhaoshuai ~]# nohup sh testdata.sh &
[2] 56503
[root@zhaoshuai ~]# nohup: ignoring input and appending output to `nohup.out'

[root@zhaoshuai ~]# ((num++))
[root@zhaoshuai ~]# ((num++))
```
然后我们查看日志： `tail -f nohup.out`
```
old:2
waiting parent modify num
new:2
```
可以看到父进程对变量的修改并不影响子进程。因此我们可以得出结论： **父子进程之间的数据是隔离的，数据修改互不干涉**
### 紧随而来的问题是：redis进行RDB时，创建子进程的速度以及内存空间大小。
基于上面的理论，我们是不是可以认为，父进程创建子进程时，也为子进程导出了一份变量的副本，然后两个进程各自修改各自的变量(**当然实际并不是这样，我们目前进行这样的猜想**)
#### 创建子进程的速度？
那么要保证数据的时点性，首先需要考虑的是创建子进程的速度。因为要导出变量的副本，首先需要创建子进程，如果子进程创建十分钟，那么导出的副本就是8点10粉的数据
了，也就不能保证数据的时点性了。  
怎么解决？  
linux系统有一个系统调用叫做fork(),fork()玩的是一个指针的引用，可以达到的效果：1.创建速度特别快。2.空间占用小。
怎么实现？  
计算机的内存，叫做物理内存，可以将其看成是一个线性数组，然后每个应用程序运行在内存中，都有一个虚拟地址空间，程序默认所有的内存都是自己可用的。
因此redis运行在内存中时，会有自己的虚拟地址，而且redis内部也有虚拟地址，比如定义一个变量a=8，那么8是数据，存放在物理内存的1位置，然后变量a指向虚拟内存5
的位置，虚拟内存5又存放的是物理内存的位置1，这样a就可以取出数据8了。  
##### 当创建一个子进程时，子进程也是一个进程，也有自己的虚拟空间，如果它要把父进程中的数据复制一份，他应该怎么做呢？  
1. 将父进程的物理内存再复制一份，比如说父进程中a->虚拟内存地址5->物理内存地址1(物理内存1中存放数据8)，然后在子进程中，将父进程的物理内存复制一份，比如说：
物理地址2(存放数据8)，然后子进程中a->虚拟内存地址5->物理内存地址2(物理内存2中存放数据8)。
2. 将父进程中的所有虚拟地址复制一遍，也就是说最终子进程和父进程的变量a指向同一个物理地址。
#### 内存空间的大小
比较上面两种创建子进程数据复制的方式，明显方式1是不可取的，因为那样会造成物理内存占用翻倍。而fork()调用使用的就是第二种方式。
这种方式物理内存的占用并没有改变，也达到了复制数据的目的。
#### 引出的一个新的问题，两个进程数据复制没有问题，那么数据修改怎么办？
我们之前证明了，两个进程的数据修改是互不影响的。也就是说父进程将变量a的值改为9时，子进程中应该还是8。但是现在两个进程变量指向同一个物理地址。
如果父进程对变量修改，将a的值改为9，那么如果改动物理地址的话，肯定子进程的值也就改变。这样不符合上面的验证结果。  
#### copy on write
为了解决上面出现的问题，使用了一个知识点叫做 `copy on write`，写时复制。我们上面已经详细讲解了，创建子进程时并不复制数据。而`copy on write`意思是
当父进程或子进程要修改数据时，才发生复制。也就是说原来父进程中变量a->虚拟地址5->物理地址1(数据为8)，那么创建子进程时复制这份指针。
当父进程或子进程需要对数据进行修改时，首先将物理地址复制一份，物理地址2(数据为8)，然后将进程的变量物理地址指向进行修改。变量a->虚拟地址5->物理地址2，
然后再在物理地址2中将数据8改为9。这样就保证了两个进程的数据修改互不影响。
> 上面详细说明了为了保证数据的时效性使用的方法以及遇到的问题，处理方式。因为复制指针，所以创建进程非常快，同时因为不可能父进程将所有数据都修改一遍。
>因此内存的占用非常少。

redis在进行RDB时，会使用fork()创建子进程，创建进程非常快，也就保证了子进程中的数据就是8点那一个时间点的数据，然后因为子进程是往磁盘落数据的，因此
子进程中的数据是只读的，不会发生修改。父进程对外提供服务，父进程的修改使用了写时复制，因此不会影响到子进程。
#### redis的RDB配置
打开redis的配置文件，关于RDB的配置如下：
````text
#快照文件存放路径
dir /var/lib/redis/6379
#快照文件名
dbfilename dump.rdb
#快照的触发条件
save 900 1
save 300 10
save 60 10000
#是否开启压缩
rdbchecksum yes
````
后台执行RDB操作的命令是bgsave，但是在配置文件中的配置确是save，也就是说配置文件中的save配置实际是对bgsave生效的。save后有两个参数，第一个是时间，
第二个是条数。save有多条配置，只要满足任意一个，就会写RDB。  
上面的配置，就是说当60秒的时候如果写操作数达到10000条就会执行RDB，如果没有达到10000，那么从61秒开始，就会进入300秒的判断，当写操作到达10条就会写RDB，
如果这个仍没有命中，那么就会再判断下一个，当到达900秒的时候，写操作是否达到1条，如果有一条写操作，就写RDB文件，防止时间过长，导致数据丢失。
如果想要关闭RDB，就写一个 `save ""`，默认是开启RDB的。
#### RDB的弊端
经过上面的了解，可以得出RDB落的是某一个时点的全量数据，那么RDB的弊端就是数据丢失相对多。比如说现在每隔一小时落一次数据，那么当8点落数据后，9点落RDB前，挂机了，
那么就丢失了一个小时的数据。
#### RDB的优点
rdb这种镜像，类似于java中的序列化，其实就是内存中的字节数组用最快的方式搬到磁盘中去，所以恢复数据的时候也相对快。
## AOF(日志)
AOF 就是 append on file,向文件中追加，追加的就是写操作。也就是说每个增删改操作都会写到文件里，这样的好处就是数据丢失会少。
**注意：当redis同时开启了aof和rdb，那么aof会落，rdb也会落，但是当服务重启时，只会从aof中恢复数据，因为aof恢复的数据相对完整，就不做rdb恢复了。
#### aof的优点：丢失数据少
#### aof的缺点：文件体积无限变大，恢复比较慢，因为日志中记录了很多的写操作，aof恢复就是将这些操作在执行一遍，所以速度比较慢。
关于aof和rdb的优缺点在上面都总结过了。再说一遍：rdb恢复快，数据丢失比较多；aof恢复慢，但是数据丢失比较少。
##### 那么如果我们就要想一种方法，怎么样能够既保证恢复速度，又保证数据丢失少的方案？  
想办法将两种方案组合起来：使用RDB的方式落时点文件，两个时点之间的文件使用aof来保存。通过这种组合方式，无论什么时间点宕机，我们都可以通过找到上一次rdb的文件
以及上一次rdb完到宕机的时间之间的aof文件来恢复数据，使用rdb恢复全量数据，保证恢复速度，而使用aof来对恢复数据进行补充，减少数据丢失。
例如：每隔一个小时进行一次RDB。8点的时候进行一次RDB，然后8点到9点之间的数据都通过aof记录在日志中，9点的时候将8点时的RDB文件及增量日志存入镜像中，重新进行RDB，然后
将日志清空，重新记录增量日志，这样就能保证日志文件足够小。当服务宕机时，只需要恢复最近的一次RDB文件，以及写入增量日志文件就可以恢复数据。  
但是前面说了，redis中如果开启了aof，那么就只会通过aof来恢复数据，不使用rdb文件，那么redis是如何恢复数据的？
* 在redis-4.0版本之前redis会有一种机制叫做重写。
    ```text
    什么是重写？  
       假如说我们redis中存了十年的数据，在这十年间就是不断的创建key，删除key，如果最后一次是创建key，没有删除，那么其实前面的都是可以抵消调的，只需要创建一次key就行了。
    如果最后一次是删除key，那么这个key就直接不需要创建，因为最终的数据是没有这条数据的。
       还有就是如果数据是一个集合类型，那么这十年间不断的往集合中添加元素，删除元素，例如执行了十万次push 1，那么其实最终可以使用push 10万个1，来达到相同的
    效果，前面执行了十万次push操作，而最终恢复数据只用了一次push操作，两个效果相同，很明显执行一个push的效率比较高。
    **重写就是抵消和整合命令，删除抵消的命令，合并重复的命令，归属到一个key里面**  
    重写的结果是多条指令合并程一条指令，然后将重写后的指令都放入一个纯指令的日志文件里面。
    但是因为日志文件大小是不断增加的，因此如果日志文件非常大的话，那么重写这个过程也是非常耗时的，因此恢复数据仍然很慢。因此在4.0版本之后进行了改进。
    ```
* 在redis-4.0之后
    ```text
    在4.0之后，重写会将所有的合并后指令放入一个纯指令文件，然后redis会将这个文件中的数据通过rdb的方式写入aof文件中，然后再将增量的数据以指令的方式append到aof，
    也就是说间接的出结论：aof当中包含rdb和增量日志。此时aof文件就是一个混合体，即利用了rdb恢复快，又利用了日志的全量，数据丢失少。如果8点触发，那么就
    把8点内存里的数据写成rdb，写到aof文件中，然后8点往后的所有新的增删改就开始追加。
    ```
 经过上面的学习，处理了持久化的问题，那么此时会引出一个新的问题：**I/O**。
 redis是内存型数据库，数据都存在内存中，那么持久化的话就必然会染指I/O。redis的特征就是快，但是当产生I/O后，那么就会影响redis的速度。  
 redis往aof文件写操作提供了三种I/O级别：always、no、everysec。下面打开redis的配置文件学习aof的相关配置。
 ```properties
# 默认关闭aof
appendonly no

# aof文件名
appendfilename "appendonly.aof"

# aof默认的I/O级别
# appendfsync always 
appendfsync everysec
# appendfsync no

# 当aof日志增加指定百分比时，开启重写
auto-aof-rewrite-percentage 100
# 重写aof文件的最小大小
auto-aof-rewrite-min-size 64mb
```
#### 关于三个I/O级别：
我们在写java代码时，如果需要向某一个文件中写数据，写完之后，关闭文件之前，需要做什么操作？  
**flush**  
为什么要调用flush？  
在计算机中，所有对硬盘的I/O操作都必须要调用内核，内核会对每一个I/O开出一个缓存空间buffer，然后如果想要向磁盘中写东西，会优先写到buffer中，buffer满了
之后，内核会调用flush向磁盘中刷写。而flush就是刷新缓冲区，将缓冲区的数据刷入磁盘中。所以说，如果向文件中写数据，最后一次写的数据没有占满buffer，
那么内核不会主动调用flush，那么如果没有调用flush，就会丢失这一块的数据，所以需要手动调用flush，刷新缓冲区。缓冲区的大小可以调，一般4k左右。
然后我们在看redis的三个I/O级别：
* no：redis不调flush。也就是说当redis来了四笔写操作，那么每一笔都会向buffer中写，buffer什么时候满了，就往磁盘中刷写。这种方式可能会丢失的最大数据量
就是buffer的大小。
* always： 每一笔写操作都调用flush。这个级别数据是最可靠的，最多可能就是有一笔数据过来调flush的时候停电了，那么最多丢失一笔数据。
* everysec： 每秒钟调用一次flush。这个级别的数据丢失量是多少呢？最坏的情况，在这一秒内，这个buffer内的数据差一点满了，但是还没满，因此没有调flush，
那么在下一秒到达，刚要调用flush时，停电了，那么最多就是丢失一个buffer的数据。但是如果写的比较快的话， buffer很快就满了，就会自动调用flush，因此一秒可能
会调用三四次flush。所以上面那种情况触发概率是非常低的。因此每秒这种是no这种最慢的和always这种最快的中间的一种，相对丢失数据较少的。

在配置文件中有这么个配置： `aof-use-rdb-preamble yes`默认是yes开启的，这个配置是在aof中写入rdb文件。
因为在老的版本中，如果触发重写，那么redis需要遍历老的aof文件，该抵消的抵消，该整合的整合，这是一个非常消耗cpu的计算判断过程。使用这种方式的话，就把cpu
判定的过程给取消掉了，直接对用内存对磁盘的aof文件做卸数，将内存的东西持久化，导成rdb这种方式写入到aof文件中，相当于加快了重写的过程。最终aof文件中同时
包含aof和rdb，aof是增量的。成一个混合体了。
查看aof文件，如果开头出现了"redis"这个字符串，则说明这是一个混合文件，如果没有则说明这是一个老的aof文件。
### rdb和aof实操
首先 `ps -ef |grep redis`保证没有redis实例在运行。  
修改redis的配置文件，将后台运行改为前台阻塞运行 `daemonize no`并注释掉日志文件 `# logfile /var/log/redis_6379.log`,因为日志文件打开的话，
就会将东西记到日志中，关了后会打到屏幕上。打开aof配置 `appendonly yes`。
然后先关闭 `aof-use-rdb-preamble no`,并删除 `/var/lib/redis/6379/dump.rdb`文件，这是以前的数据，清掉。然后启动redis
```shell script
[root@zhaoshuai ~]# service redis_6379 start
Starting Redis server...
62114:C 29 Oct 2020 02:36:19.138 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
62114:C 29 Oct 2020 02:36:19.138 # Redis version=5.0.5, bits=64, commit=00000000, modified=0, pid=62114, just started
62114:C 29 Oct 2020 02:36:19.138 # Configuration loaded
62114:M 29 Oct 2020 02:36:19.139 * Increased maximum number of open files to 10032 (it was originally set to 1024).
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 5.0.5 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 62114
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           http://redis.io        
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

62114:M 29 Oct 2020 02:36:19.143 # Server initialized
62114:M 29 Oct 2020 02:36:19.143 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
62114:M 29 Oct 2020 02:36:19.144 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
62114:M 29 Oct 2020 02:36:19.146 * Module 'bf' loaded from /usr/local/redis-5.0.5/RedisBloom-master/redisbloom.so
62114:M 29 Oct 2020 02:36:19.147 * Ready to accept connections
```
另启一个窗口，打开redis客户端，然后执行 `set k1 hello`
```shell script
[root@zhaoshuai ~]# redis-cli 
127.0.0.1:6379> set k1 hello
OK
```
查看持久化目录
```shell script
[root@zhaoshuai redis]# cd /var/lib/redis/6379
[root@zhaoshuai 6379]# ll
total 4
-rw-r--r--. 1 root root 55 Oct 29 02:37 appendonly.aof
```
然后打开这个文件，看到如下内容：
```text
*2
$6
SELECT
$1
0
*3
$3
set
$2
k1
$5
hello
```
*后面的数字表示一条完整的指令需要读取几行。$后边的数字表示下一行指令的长度。上面的内容就是aof的内容，而且这个文件是很纯净的，没有其他多余的东西。
再添加一条 `set k2 redis`,再次打开，可以看到每条命令都会追加到这个文件后面
然后我们会看到上面没有dump.rdb。
执行 `save`命令会阻塞服务，将数据打成dump.rdb；使用 `bgsave`后会重启一个子进程，将服务落成dump.rdb。查看前台阻塞窗口日志：
```shell script
64351:M 29 Oct 2020 18:17:45.712 * DB saved on disk
64351:M 29 Oct 2020 18:19:05.981 * Background saving started by pid 64373
64373:C 29 Oct 2020 18:19:06.038 * DB saved on disk
64373:C 29 Oct 2020 18:19:06.038 * RDB: 6 MB of memory used by copy-on-write
64351:M 29 Oct 2020 18:19:06.110 * Background saving terminated with success
```
进入 `/var/lib/redis/6379`,可以看到有aof和rdb两个文件。
```shell script
[root@zhaoshuai 6379]# ll
total 8
-rw-r--r--. 1 root root  87 Oct 29 18:17 appendonly.aof
-rw-r--r--. 1 root root 117 Oct 29 18:19 dump.rdb
```
使用vi命令，查看dump.rdb
```shell script
REDIS0009ú      redis-ver^E5.0.5ú
redis-bitsÀ@ú^EctimeÂ
j<9b>_ú^Hused-memÂ^H^^^M^@ú^Laof-preambleÀ^@þ^@û^B^@^@^Bk2^Eredis^@^Bk1^Ehelloÿ~<83>ED<85>þ·Q                                                                                         
```
dump.rdb是一个二进制文件，不太好看，可以使用命令： `redis-check-rdb dump.rdb`来检查rdb文件：
```shell script
[root@zhaoshuai 6379]# redis-check-rdb dump.rdb 
[offset 0] Checking RDB file dump.rdb
[offset 26] AUX FIELD redis-ver = '5.0.5'
[offset 40] AUX FIELD redis-bits = '64'
[offset 52] AUX FIELD ctime = '1604020746'
[offset 67] AUX FIELD used-mem = '859656'
[offset 83] AUX FIELD aof-preamble = '0'
[offset 85] Selecting DB ID 0
[offset 117] Checksum OK
[offset 117] \o/ RDB looks OK! \o/
[info] 2 keys read
[info] 0 expires
[info] 0 already expired
```
现在rdb和aof文件都有了，但是我们还没有验证 `aof-use-rdb-preamble`这个设置，我们在redis客户端中执行：`bgrewriteaof`命令，开启重写，重写完成后
查看aof文件：
```shell script
*2
$6
SELECT
$1
0
*3
$3
SET
$2
k2
$5
redis
*3
$3
SET
$2
k1
$5
hello
```
发现aof文件与之前没有变化，开头并没有出现redis，所以是老的aof文件。  
然后我们在验证重写整合命令：在这之前我们存入了k1-hello,k2-redis,往后追加命令
```shell script
127.0.0.1:6379> set k1 a
OK
127.0.0.1:6379> set k2 b
OK
127.0.0.1:6379> set k1 c
OK
```
查看aof文件：
```shell script
*2
$6
SELECT
$1
0
*3
$3
set
$2
k1
$5
hello
*3
$3
set
$2
k2
$5
world
*3
$3
set
$2
k1
$1
a
*3
$3
set
$2
k2
$1
b
*3
$3
set
$2
k1
$1
c
```
可以看到所有的命令都是往后追加，这样下去文件就会非常大。我们执行重写命令再次查看文件：
```shell script
*2
$6
SELECT
$1
0
*3
$3
SET
$2
k1
$1
c
*3
$3
SET
$2
k2
$1
b
```
可以看到重写之后文件简化了很多。

然后我们关闭redis前端阻塞服务(ctrl+C),删除aof和rdb文件，修改配置文件打开 `aof-use-rdb-preamble yes`,再次启动服务。重复上面操作。
```shell script
127.0.0.1:6379> set k1 hello
OK
127.0.0.1:6379> set k2 redis
OK
127.0.0.1:6379> save
OK
127.0.0.1:6379> BGREWRITEAOF
Background append only file rewriting started
```
再次查看aof文件：
```shell script
REDIS0009ú      redis-ver^E5.0.5ú
redis-bitsÀ@ú^EctimeÂ^Mr<9b>_ú^Hused-memÂ^P^^^M^@ú^Laof-preambleÀ^Aþ^@û^B^@^@^Bk2^Eredis^@^Bk1^Ehelloÿúî^Y<9c><8a>î¨<9e>
```
可以看到开头是redis开头的，而且我们不难发现，这写内容与上面看的rdb文件的内容一样。这也验证了我们上面的说法，然后在往后追加数据，查看日志：
```shell script
REDIS0009ú      redis-ver^E5.0.5ú
redis-bitsÀ@ú^EctimeÂ^Mr<9b>_ú^Hused-memÂ^P^^^M^@ú^Laof-preambleÀ^Aþ^@û^B^@^@^Bk2^Eredis^@^Bk1^Ehelloÿúî^Y<9c><8a>î¨<9e>*2^M
$6^M
SELECT^M
$1^M
0^M
*3^M
$3^M
set^M
$2^M
k3^M
$1^M
a^M
*3^M
$3^M
set^M
$2^M
k4^M
$1^M
b^M
```
我们可以看到，后续的操作都是追加在rdb文件后面的。此时我们再次执行： `bgrewriteaof`
```shell script
127.0.0.1:6379> BGREWRITEAOF
Background append only file rewriting started
```
然后查看rdb文件，可以看到这是一个新的rdb文件，再次整合了之前追加的操作为rdb，后面的操作将追加在此rdb文件后面：
```shell script
REDIS0009ú      redis-ver^E5.0.5ú
redis-bitsÀ@ú^EctimeÂõr<9b>_ú^Hused-memÂp^^^M^@ú^Laof-preambleÀ^Aþ^@û^D^@^@^Bk4^Ab^@^Bk3^Aa^@^Bk2^Eredis^@^Bk1^Ehelloÿ<9f><8a>ÿ²â@ûß
```
经过上面的操作我们知道了，无论何时，aof中的文件永远都是最近一次rdb全量数据+追加增量数据，而且每次都会重写这个文件--aof文件开头都是新的rdb文件，也就保证了
aof文件永远不会太大。利用了aof和rdb两个的优点。
> 需要记住的两个命令：
> save/bgsave：保存rdb文件/后台保存rdb文件
> bgrewriteaof: aof重写文件

##### 开启aof-use-rdb-preamble后，aof文件仍是正常的纯命令文件，之后执行重写后才会变成混合文件
之前我们学习rdb时，知道rdb有三个配置可以自动保存：
```shell script
save 900 1
save 300 10
save 60 10000
```
但是上面我们在使用aof的重写时，都是手动触发的，aof也有自动触发机制。回顾关于aof的配置：
```shell script
# Automatic rewrite of the append only file.
# Redis is able to automatically rewrite the log file implicitly calling
# BGREWRITEAOF when the AOF log size grows by the specified percentage.
#
# This is how it works: Redis remembers the size of the AOF file after the
# latest rewrite (if no rewrite has happened since the restart, the size of
# the AOF at startup is used).
#
# This base size is compared to the current size. If the current size is
# bigger than the specified percentage, the rewrite is triggered. Also
# you need to specify a minimal size for the AOF file to be rewritten, this
# is useful to avoid rewriting the AOF file even if the percentage increase
# is reached but it is still pretty small.
#
# Specify a percentage of zero in order to disable the automatic AOF
# rewrite feature.

auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```
我们大概可以翻译一下这个注释：意思就是自动重写aof文件。当aof文件的大小增长到指定的百分比时，redis会通过调用bgwriteaof来实现重写log文件。
然后讲了它是怎么工作的：redis会记住最后一次重写的文件大小（如果重启后没有重写发生过，那么就会记住重启时aof文件的大小）  
基础大小会与当前的大小进行比较，如果当前大小比指定的百分比大的话，就会触发重写，你也需要指定重写的最小aof文件大小，这样即使在百分比增加的情况下，也可以避免
重写的aof仍然很小。  
**指定百分比为0可以禁用自动重写**
说人话：就是说redis会记住你最后一次重写后文件的大小(如果我重启后，还没有触发过重写，那么就会使用当前aof文件的大小)，如果当前aof文件的大小与
我记住的aof文件大小相比增长了100%,那么就会触发重写。重写后文件的大小就会减少。比如说我最后一次重写后，文件大小是32M，那么当文件增加到64M后，就会触发重写，
然后再次重写后，文件大小是40M，那么我下次增加到80M我才会触发重写。 `auto-aof-rewrite-percentage 100`就是你来设定这个百分比的，当达到指定百分比就会
触发重写。那么还有一个问题，就是你要知道，如果我是第一次启动的话，那么我没有aof文件，这样的话，我就没办法知道我该跟谁比较，来决定触发重写。所以你需要再指定
`auto-aof-rewrite-min-size`这个配置，指定一个最小的重写大小，然后第一次就跟这个比较，第一次重写过之后，就会记住那个值，然后下次就是跟记住的大小进行比较。

# redis集群
之前一直学习使用的都是redis单机、单实例。单机会遇到的问题：
1. 单点故障。也就是说单节点如果服务一旦挂了，那么服务就不可用了。
2. 内存大小。单个服务器的内存大小是优先的，但是数据是随着时间不断增大的，内存就会不够用。
3. 压力。客户端连接数，socketI/O压力，cpu计算压力。
### 单节点的这三个问题怎么解决？
只要出现单实例，那么就可以通过**AKF拆分原则**解决。什么是AKF？  
AKF扩展立方，就是通过X，Y，Z三个坐标轴的方向来解决问题。
* X轴：用来解决单点故障，服务可用性的问题。也就是说沿着X轴，为服务多复制几个一样的实例(做备用节点)，平时一个实例对外提供服务，并将自己的数据复制到备份
节点，也就是一主多从的方式。主节点提供服务并将数据保存到备用节点，一旦主节点宕机，备用节点立马升级为主节点，保证服务的可用性。同时可以配置读写分离，降低主节点
的压力。
    >注意：基于X轴做主从节点这种方式，一般都是全量镜像(备用节点中保存有主节点的所有数据)
* Y轴：沿X轴做拓展只能解决单点故障的问题，也就是保证服务的可用性。但是并不能解决容量不足的情况。沿Y轴拓展就是将数据进行拆分，将数据按照类型或业务类型进行分类，
然后将数据分散在不同的节点上。这样就解决了容量不足的问题。
    > X和Y并不是必须发生的，可以只有X，也可以只有Y，还可以X和Y一起使用。但是一般按照Y拆分的话，如果某一个节点挂了，那么这个业务的数据就不可用了，所以会在X轴做备用节点
* Z轴：在Y轴中对数据进行了分类，那么当某一个分类的数据量非常大的时候，就可以在Z轴进行拓展。也就是说将Y轴某一个节点的数据再进行拆分，分散到多个节点上，
但是必须要配置一个规则，按照这个规则去拆分数据，保证数据能够分配到指定的节点。  

对单节点的问题，通过以上AKF的拆分，然后请求肯定是访问某一个类型的数据，这样就会将请求分散到访问数据的节点，也就间接减小了访问压力。
### CAP原则
上面使用AKF解决了单节点的问题，但是一般解决一个问题就会引来新的问题。配置集群会产生的问题：CAP  
#### 什么是CAP？ 
* C：Consistency(一致性)。后台提供服务的多个节点之间数据保持一致
* A: Availability(可用性)。用户访问服务，服务的响应时间在可以接受的范围内。因为客户端通过建立连接访问服务，一般都会设置一个超时时间，如果超过时间的话，连接就会被关闭，那么客户端就会返回失败，那么如果每次都因为超时返回失败的话，客户端就会认为这个服务是不可用的。
* P: Partition tolerance(分区容错性)。**高可用**，一个节点宕机，并不影响其他的节点。
> CAP原则就是值：上面这三个要素最多只能同时实现两个，不能三个都保证。

为什么？  
假设现在需要搭建一个redis服务，那么首先我们创建一个单实例的节点。为了解决单点故障的问题，我们就根据AKF原则沿X轴扩展，搭建主备节点，来提高可用性。
那么我们现在搭建一个一主两备的服务，单点故障问题解决了，可用性提高了。但是当出现主备或主从服务时，如果我们现在要往redis中存数据，那么就要保证数据的一致性。
也就是说我们要保证主节点和备用节点(或从节点)的数据时一样的。
> 关于主备和主从节点的区别：
> 主备：主节点对外提供读写服务，当主节点可用时，备节点就只是单纯的同步主节点的数据，并不对外提供服务。当主节点宕机时，我们可以启用备用节点，保证服务可用。
>也就是说当主节点可用时，备节点是不可用的。
> 主从：主节点对外提供读写服务，从节点只能对外提供读的服务，也就是说从节点是只读的。
  
主节点对外提供写的服务，备节点或从节点，要么不对外提供服务，要么提供读的服务，所以数据只能从主节点同步到备节点或从节点。  
当出现这种多个节点数据同步时，那么就会出现一个问题：**数据一致性**

什么是数据一致性？  
* 当服务为主备节点时，那么只有主节点对外提供读写的服务，当插入一条数据时，会保存到主节点中，并同时保存到备份节点中。当一条数据保存到主节点中，还没来的及
保存到备用节点，主节点宕机了，那么备用节点就会少一条数据，主节点和备用节点的数据就会不一致。
* 当服务为主从节点时，那么主节点发生写操作，从节点发生读操作，那么当网络延迟或其他情况，导致主节点中的数据没有及时的同步到从节点，就会造成从节点中的数据和
主节点不一致。

经过上面的分析，发现数据的不一致的原因都是因为主节点向从节点或备份节点同步数据时出现问题造成的。那么这个问题怎么解决？  
当有一条写操作到达主节点时，主节点会先保存到本地，然后再将数据保存到备份节点，那么此时有两种方案保存到备份节点：
1. 同步的方式： 主节点写入成功后，并不立刻响应客户端，而是会在主节点阻塞并同时向两个备份节点插入数据，只有两个备份节点都返回ok,主节点才会返回ok。
2. 异步的方式： 主节点写入成功后，立刻返回客户端ok，并通过异步的方式将数据写入从节点或备份节点。

我们分析上面两种同步数据的方式的优缺点：
1. 同步的方式：
    * 优点： 所有节点的数据都是一致的，因为主节点会阻塞，只有所有节点的数据都保存成功，才算保存成功。这是**强一致性**
    * 缺点： 发生阻塞，但是客户端会有一个超时的概念，也就是说，当阻塞时间过长时，客户端会发生超时，那么就会认为保存失败，这样的话，客户端就会认为服务不可用。
    因为我想要保存一条数据，但是保存失败。这样就会破坏服务的可用性。
2. 异步的方式： 
    * 优点： 主节点保存成功就会立即返回，响应速度快。而且不会影响服务的可用性。
    * 缺点： 无法保证数据的一致性。因为如果主节点在准备将数据保存到备用节点，但是还没有开始时，主节点宕机了，那么就会造成备用节点数据丢失。

分析了上面两种方式，我们发现，如果要保证数据的强一致性，就会破坏服务的可用性，如果要保证服务的可用性，那么就一定会影响数据的一致性。    
怎么解决上面的问题？  
看起来好像是一个死循环，企业都想要追求数据的强一致性，但是这样却会破坏可用性，为了解决这个问题，只能将数据一致性降级。  
数据一致性分为：
* 强一致性
* 弱一致性
* 最终一致性

强一致性会破坏服务可用性，那么往下降级，使用弱一致性，但是使用数据弱一致性就要忍受数据的丢失，只会丢失一点数据。为了解决弱一致性中出现数据丢失的问题，就出现了最终一致性。  
最终一致性的解决方案：  
就是客户端写操作进来后，首先肯定是进入主节点，然后我主节点往自己本地保存成功后，并不是先往备用节点保存，而是在主节点和备用节点中间加上一个类似kafka这种功能的服务，
这个服务首先要保证高可用，就是redis主节点往这个服务里写数据不能失败，还要保证够快；然后主节点将数据写入这个中间层后，就返回给客户端成功。然后备用节点从这个中间层
拿数据写到本地，这样就能保证最终各个节点的数据是一致的，这就是最终一致性。
> 注意：redis并没有使用这种方式，redis是怎么做的，后面讲解，现在讲的CAP原则的一致性。

最终一致性会带来的问题就是，在达到最终一致性之前，可能主从节点的数据可能会出现短暂的不一致，但是最终是一致的。所以最终一致性也是属于弱一致性的。  

上面我们说了一致性的问题，那么再想一下，现在无论我们是主备节点还是主从节点，都有一个什么？**主**，那么问题就是这个主也是个单节点。  
此时想一下，我们上面因为单节点的问题，又是做主备，又是做主从，然后还引出了数据一致性的问题，以及可用性的问题，绕了一大圈，又回到了单节点的问题：主节点是单节点。
主节点是单节点的问题： 如果是主备的话，那么主节点挂了的话，服务就不可用了，如果是主从节点，那么主节点挂了的话，只能读，不能写了。
所以我们就需要对主节点做**高可用**。也就是将一个备节点切换成主节点。
高可用是只对主节点做的，如果是备节点挂了也就挂了，并不会对客户端有影响，因为只有主节点对外提供服务。  
那么高可用强调的是我这个主节点永远不会出现问题。(当然这是不可能的)，所以高可用更倾向的是当主节点出现问题时，我能够立马出现一个新的主节点出来，对外表现
是并没有出现问题。  
那么出现新节点这个事情，应该怎么做呢？
肯定不能人来做，我不可能让一个人来时时刻刻看他有没有宕机，所以应该有一个程序来处理，实现故障转移，对外表现一种高可用的状态，也就是说通过程序监控主节点
当主节点出现问题时，能够及时的将备用节点切换成主节点，将出现故障的原主节点撤下来。  
那么程序如何来监控这个服务有没有出现问题？首先要想，这个程序肯定不能是一个，因为如果是一个的话，就会出现单点故障的问题，所以这个程序也应该是一个集群。那么这个
监控程序是集群，那么就会引出一个新问题：  
假设这个集群有三个实例，那么就是说三个程序监控一个redis节点，那么这三个监控如何确定这个redis有没有问题呢？  
三个监控去监控这一台redis主节点，首先，每个监控程序肯定要能与redis之间进行通信，不能通信还怎么监控。然后如果说我有一台监控程序因为网络问题，无法与redis主节点通信了，
但是另外两台都是可以正常与redis节点通信的，那么这时候我怎么能肯定这个主节点到底有没有问题呢？  
问题也就是说： 现在三个监控，有一个监控说redis挂了，因为他出现了网络问题，无法与主节点通信了，他认为redis主节点挂了，但是其实人家是正常的，另外两个都能正常访问的
那这时候该怎么办呢？不能你说他挂了他就挂了，所以这三个监控程序之间也应该能够通信。那么这三台之间通信的话，就会有数据一致性问题了，我现在三台一块商量这个redis到底有没有挂，
那如果有一个节点出现网络问题或其他问题，无法通信了，那我如果要保证强一致性，另外两台就会阻塞等待第三台也给出一个意见，但是第三个已经挂了，那么此时这个监控也就
不可用了。所以这三台监控之间不能是强一致性，因为强一致性会影响可用性。那么既然三台不能强一致性，那么我三台该怎么商量他到底有没有问题，也就是说该不该替换掉这个
主节点？既然不能三台都给出的话，那么只能是一部分节点给出这个决定，只要这一部分给出的结果是要替换，那么就把当前的主节点给替换掉。那这个一部分应该是几台呢？    
#### 很容易想到的就是半数选举。现在我们就推导这个半数是怎么来的：
假如说我们现在这个监控程序有五个节点，那么这个一部分能不能是一台呢？肯定不能。其实上面已经分析过了，一个节点给出决定的话，有可能是因为这个监控程序自身的问题，
其实主节点是正常的。自己的问题自己不知道还认为是别人的问题，这肯定是不行的。而且如果一台能够决定的话，会出现竞争，你说他挂了，但是我认为他没挂，到底该听谁的。
那么两台呢？也就是说，我有两台监控，然后我俩一商量都认为redis的主节点挂了，那这时候该不该换掉他呢？其实和一个节点的时候一样，有可能你这两个节点的网络网段变了，
人家另外三台还是可以正常连接的，所以此时，也不应该替换掉。为什么，因为你这两个节点结成一个势力了，但是人家另外三台节点也结成另一个势力了，你的势力没有人家的大。
人家三个人，你打不过人家，所以谁拳头大谁做主，人家三个说他没挂，那就是没挂，三个说他挂了，那他就是挂了。即使是这三个的网段变了，主节点真的没挂，但是人家三个
拳头大， 说他挂他就得挂。这样我们就推导出半数选举了。
#### 我们应该在搭建集群的时候都听过一句话: 奇数台最好。
无论是redis还是分布式里的注册中心，都是搭建奇数台，为什么奇数台最好？  
首先我们分析，当数量为3台时，那么此时允许出现故障的数量是1台，因为三台时只有两台才能结成势力作出决定。那么四台的时候呢？允许出现故障的数量也是1台。如果有两台出现故障，
那么出现故障的势力也是2，这样就无法决定到底该听哪儿个的了，无法给出最终决策，所以允许出现故障的数量也是1台。那么同样都是1台。3台和4台的成本可是不一样的，
四台更贵，而且四台比三台更容易出故障，所以根据经验，奇数台最好。
#### 脑裂问题(出现脑裂问题的前提是没有考虑过半机制)
什么是脑裂？我们人只有一个大脑，在分布式服务中，多个实例也只能有一个主节点，这个主节点就是大脑。那么脑裂就是现在一个大脑裂开了，变成了两个甚至三个大脑。
映射到服务中就是出现了多个主节点。
脑裂问题怎么出现的？
前面我们分析了，为了实现高可用，要对主从节点搭建监控服务，监控主节点的心跳状态。比如说我们现在有五台服务，假设现在网络出现问题了，然后五个监控程序分成了两个网段。
有三台是一组，这三台之间可以互相通信，另外两台为一组，这两台之间也可以互相通信。但是只有一组能够跟主节点通信，那么无法跟主节点通信的这一组，就会自成一派，然后俩人合计以分析
主节点挂了，然后就将一个从节点提成主节点了，那么这时就会造成一个服务中出现两个主节点，这两个主节点间无法通信，客户端每次访问时，可能访问这个主节点，也可能访问
另一个主节点，而且这两个主节点的信息是无法保证一致的，老得主节点中保存的是老得数据，新的主节点中保存新的数据。但是因为对外表现高可用，客户端是不知道你服务有两个主节点的。
那么对外表现是就是，我客户端调用服务，然后可能每次调用结果都不一样。
##### 分区容忍性
脑裂问题是由于网络分区引起的。其实这种服务对网络分区有一个分区容忍性的概念。也就是说能不能容忍脑裂时出现的数据不一致问题。
例如，在微服务中，都会有一个注册中心。注册中心就是用来存放提供服务的实例信息，例如有个订单业务，有十个实例，但是因为出现了网络分区的问题，此时这边注册了八台，
那边注册了两台，但是请求过来后，并不关心你总共多少台，只需要给我一个能用的实例我去调用就行了，不管是八台的还是两台的，都可以提供一个服务给客户端调用，而且能够调用
成功，这时就是可以容忍网络分区的。
> 这一块自己讲的时候总感觉被绕进去，其实就是要搞清楚一个点：
> 过半选举：如果是过半选举的话，肯定要保证，各个监控之间是能够正常通信的，多个监控共同监控一个主节点，可能某一个或几个节点无法与主节点通信了，才会进行选举新的主节点。
> 如果监控之间无法通信了，那就会出现脑裂了。
#### 上面我们一起分析了CAP三大原则，那么回到最初的问题，为什么这三个原则不能同时满足，最多只能同时满足两个？
CA：如果既要满足强一致性，又要满足可用性，那只能选择单机了。单机这两点都满足，但是高可用达不到，因为高可用要求就是集群。
CP：如果要满足高可用，就要搭建集群，搭建集群，如果要满足数据强一致性，那么就会破坏可用性(一致性里分析过)。
AP：如果要同时满足可用性，和高可用性。前面也说过了，要满足可用性，就会破坏强一致性。满足强一致性救护破坏可用性，只要满足P，那么C和A就只能选择一个。

### redis集群搭建
redis的集群搭建，采用的是主从复制集群，主从复制采用异步复制，因为异步的可用性更高。回顾一致性中，保证可用性的话，就会破坏一致性。因此redis是弱一致性的。
容易丢失数据。为什么不采用最终一致性？因为redis的特点就是快，为了快，就要减少技术整合。
#### redis集群的搭建
1. 首先停掉所有的redis实例， `ps -ef |grep redis`
2. 通过install_server创建三个redis实例（在一台服务器上面通过开启多个端口开启多实例）6379，6380，6381三个服务。并关闭服务。
3. 那么现在就有了三个redis实例，我们搭建一个以6379为主，6380和6381为从节点的主从服务。那么修改6380和6381的配置文件，修改参数 `5.0版本前是slaveof`
`5.0版本之后是replicaof`更改为 `replicaof 127.0.0.1 6379`，
4. 删除这三个实例的日志文件，并启动主节点以及两个从节点。 `rm -rf /var/log/redis*`
```shell script
[root@zhaoshuai log]# service redis_6379 start
Starting Redis server...
[root@zhaoshuai log]# service redis_6380 start
Starting Redis server...
[root@zhaoshuai log]# service redis_6381 start
Starting Redis server...
```
5. 使用tail监控6379日志：
```shell script
.....
74730:M 01 Nov 2020 23:58:30.422 * Module 'bf' loaded from /usr/local/redis-5.0.5/RedisBloom-master/redisbloom.so
74730:M 01 Nov 2020 23:58:30.423 * DB loaded from disk: 0.000 seconds
74730:M 01 Nov 2020 23:58:30.423 * Ready to accept connections
74730:M 01 Nov 2020 23:58:36.817 * Replica 127.0.0.1:6380 asks for synchronization
74730:M 01 Nov 2020 23:58:36.817 * Full resync requested by replica 127.0.0.1:6380
74730:M 01 Nov 2020 23:58:36.817 * Starting BGSAVE for SYNC with target: disk
74730:M 01 Nov 2020 23:58:36.818 * Background saving started by pid 74747
74747:C 01 Nov 2020 23:58:36.834 * DB saved on disk
74747:C 01 Nov 2020 23:58:36.835 * RDB: 6 MB of memory used by copy-on-write
74730:M 01 Nov 2020 23:58:36.912 * Background saving terminated with success
74730:M 01 Nov 2020 23:58:36.912 * Synchronization with replica 127.0.0.1:6380 succeeded
74730:M 01 Nov 2020 23:58:40.359 * Replica 127.0.0.1:6381 asks for synchronization
74730:M 01 Nov 2020 23:58:40.359 * Full resync requested by replica 127.0.0.1:6381
74730:M 01 Nov 2020 23:58:40.359 * Starting BGSAVE for SYNC with target: disk
74730:M 01 Nov 2020 23:58:40.360 * Background saving started by pid 74761
74761:C 01 Nov 2020 23:58:40.425 * DB saved on disk
74761:C 01 Nov 2020 23:58:40.425 * RDB: 6 MB of memory used by copy-on-write
74730:M 01 Nov 2020 23:58:40.428 * Background saving terminated with success
74730:M 01 Nov 2020 23:58:40.428 * Synchronization with replica 127.0.0.1:6381 succeeded
```
可以看到上面的日志，主节点同步从节点6380和6381。并落rdb文件到这两个实例。
6. 打开一个从节点日志：
```shell script
...
74743:S 01 Nov 2020 23:58:36.814 * DB loaded from disk: 0.002 seconds
74743:S 01 Nov 2020 23:58:36.814 * Ready to accept connections
74743:S 01 Nov 2020 23:58:36.814 * Connecting to MASTER 127.0.0.1:6379
74743:S 01 Nov 2020 23:58:36.817 * MASTER <-> REPLICA sync started
74743:S 01 Nov 2020 23:58:36.817 * Non blocking connect for SYNC fired the event.
74743:S 01 Nov 2020 23:58:36.817 * Master replied to PING, replication can continue...
74743:S 01 Nov 2020 23:58:36.817 * Partial resynchronization not possible (no cached master)
74743:S 01 Nov 2020 23:58:36.819 * Full resync from master: a6d3f64c1b125dc3178e7bbf4abd3abedbbdfaa3:0
74743:S 01 Nov 2020 23:58:36.912 * MASTER <-> REPLICA sync: receiving 192 bytes from master
74743:S 01 Nov 2020 23:58:36.912 * MASTER <-> REPLICA sync: Flushing old data
74743:S 01 Nov 2020 23:58:36.912 * MASTER <-> REPLICA sync: Loading DB in memory
74743:S 01 Nov 2020 23:58:36.912 * MASTER <-> REPLICA sync: Finished with success
```
可以注意到 `connecting to MASTER 127.0.0.1:6379`连接主节点以及`Flushing old data`清空就的数据，然后`loading DB in memory`加载主节点的数据。

至此一个redis主从复制集群就搭建成功了。在主节点中存入数据，然后到从节点中可以查看数据。主节点的数据通过rdb的方式传递给从节点，从节点加载rdb文件获取主节点的数据。
而且尝试在从节点中写数据会发现，从节点是只读的。不能写数据（可以在配置文件中调）
此时是一主两丛的集群，那么主节点可能会挂，从节点也可能会挂。我们现在考虑主节点健康，从节点挂的时候，它的数据同步方式是怎样的？
是主节点将所有的数据打一个rdb文件，从节点重新拉取数据？还是说从节点只同步主节点增量数据？
我们将从节点6381停掉。然后在主节点中写入数据k3，此时从节点6381是停掉的，
所以肯定不可能存入数据k3,然后我们再启动6381节点，查看k3。发现6381是能拿到k3的值的。查看6381的日志：
```shell script
....
74883:S 02 Nov 2020 00:27:24.481 * Ready to accept connections
74883:S 02 Nov 2020 00:27:24.481 * Connecting to MASTER 127.0.0.1:6379
74883:S 02 Nov 2020 00:27:24.482 * MASTER <-> REPLICA sync started
74883:S 02 Nov 2020 00:27:24.482 * Non blocking connect for SYNC fired the event.
74883:S 02 Nov 2020 00:27:24.482 * Master replied to PING, replication can continue...
74883:S 02 Nov 2020 00:27:24.483 * Trying a partial resynchronization (request a6d3f64c1b125dc3178e7bbf4abd3abedbbdfaa3:2304).
74883:S 02 Nov 2020 00:27:24.483 * Successful partial resynchronization with master.
74883:S 02 Nov 2020 00:27:24.483 * MASTER <-> REPLICA sync: Master accepted a Partial Resynchronization.
```
从节点重启后会重新尝试去主节点同步数据。而且重启后没有落rdb文件的事，也就是说，6381曾经追随过6379，现在重启后，只会同步增量的数据。
再此将6381停掉，然后修改配置文件，开启aof `appendonly yes`,然后发现，本来是不落rdb文件的，也就是说本来重启后我应该只同步增量数据的。但是现在我每次重启都会
落rdb文件
```shell script
75017:S 02 Nov 2020 01:11:38.103 * Ready to accept connections
75017:S 02 Nov 2020 01:11:38.103 * Connecting to MASTER 127.0.0.1:6379
75017:S 02 Nov 2020 01:11:38.105 * MASTER <-> REPLICA sync started
75017:S 02 Nov 2020 01:11:38.105 * Non blocking connect for SYNC fired the event.
75017:S 02 Nov 2020 01:11:38.106 * Master replied to PING, replication can continue...
75017:S 02 Nov 2020 01:11:38.106 * Partial resynchronization not possible (no cached master)
75017:S 02 Nov 2020 01:11:38.115 * Full resync from master: a6d3f64c1b125dc3178e7bbf4abd3abedbbdfaa3:6149
75017:S 02 Nov 2020 01:11:38.254 * MASTER <-> REPLICA sync: receiving 216 bytes from master
75017:S 02 Nov 2020 01:11:38.254 * MASTER <-> REPLICA sync: Flushing old data
75017:S 02 Nov 2020 01:11:38.254 * MASTER <-> REPLICA sync: Loading DB in memory
75017:S 02 Nov 2020 01:11:38.254 * MASTER <-> REPLICA sync: Finished with success
75017:S 02 Nov 2020 01:11:38.255 * Background append only file rewriting started by pid 75022
75017:S 02 Nov 2020 01:11:38.486 * AOF rewrite child asks to stop sending diffs.
75022:C 02 Nov 2020 01:11:38.486 * Parent agreed to stop sending diffs. Finalizing AOF...
75022:C 02 Nov 2020 01:11:38.486 * Concatenating 0.00 MB of AOF diff received from parent.
75022:C 02 Nov 2020 01:11:38.486 * SYNC append only file rewrite performed
75022:C 02 Nov 2020 01:11:38.486 * AOF rewrite: 6 MB of memory used by copy-on-write
75017:S 02 Nov 2020 01:11:38.535 * Background AOF rewrite terminated with success
75017:S 02 Nov 2020 01:11:38.535 * Residual parent diff successfully flushed to the rewritten AOF (0.00 MB)
75017:S 02 Nov 2020 01:11:38.535 * Background AOF rewrite finished successfully
```
又是删除旧的数据，拉master的所有数据。为什么开启aof后就落rdb文件了？  
前面说过，当开启aof后，rdb就不生效了，会直接加载aof文件。rdb可以保存曾经追随谁的信息，但是aof虽然是一个混合文件，但是aof文件并不包含曾经追随谁的信息。
所以开启aof后，每次重启都会重新落rdb文件。
查看rdb文件：
```text
REDIS0009ú      redis-ver^E5.0.5ú
redis-bitsÀ@ú^EctimeÂJÍ<9f>_ú^Hused-memÂàp^]^@ú^Nrepl-stream-dbÀ^@ú^Grepl-id(a6d3f64c1b125dc3178e7bbf4abd3abedbbdfaa3ú^Krepl-offsetÁ^E^Xú^Laof-preambleÀ^@þ^@û^C^@^@^Bk3
test_slave^@^Bk1^Ehello^@^Bk2^Eworldÿ^F^LeØ6_â®
```
注意上面这段信息： `repl-id(a6d3f64c1b125dc3178e7bbf4abd3abedbbdfaa3`
查看aof，aof中的rdb文件中是没有这个信息的，所以开启aof时，每次都重启都加载aof文件，然后aof文件并不记载曾经追随谁的信息，因此每次重启都要重新rdb。  
上面是从节点挂机的情况，从节点挂机并不影响主节点对外提供服务，而且在主节点的日志中，可以看到从节点连接信息，也就是说从节点挂了的话，在主节点也可以看到。
主节点可以知道主上连了多少从。那么当主节点宕机时，怎么处理？  
我们将主节点6379服务停掉， `service redis_6379 stop`，然后查看从节点的日记：
```text
...
74743:S 02 Nov 2020 02:15:46.525 # Error condition on socket for SYNC: Connection refused
74743:S 02 Nov 2020 02:15:47.550 * Connecting to MASTER 127.0.0.1:6379
74743:S 02 Nov 2020 02:15:47.551 * MASTER <-> REPLICA sync started
74743:S 02 Nov 2020 02:15:47.551 # Error condition on socket for SYNC: Connection refused
74743:S 02 Nov 2020 02:15:48.575 * Connecting to MASTER 127.0.0.1:6379
74743:S 02 Nov 2020 02:15:48.575 * MASTER <-> REPLICA sync started
74743:S 02 Nov 2020 02:15:48.575 # Error condition on socket for SYNC: Connection refused
```
可以看到从节点在一直不停的尝试与主节点建立连接，如果主节点此时重启的话，那么就又可以建立连接了。但是我们现在把主节点停了，也就是说主节点不会启动了。比如说主节点硬件
故障，启动不了了。那么此时剩两个从节点，从节点仍然可以被外部连接，可以提供读的服务，也就是可以读到以前写的数据，但是现在无法提供写的服务了。
那么当出现主节点出现故障时，我们是不是需要启动一个主节点。当没有监控程序自动去切换时，我们只能手动去将一个从节点升为主节点。
`redis-cli -p 6380`连接上6380服务，然后将6380升为主节点 `replicaof no one`  
```shell script
127.0.0.1:6380> REPLICAOF no one
OK
```
然后查看日志就可以看到，6380变成了主节点--MASTER MODE enabled
```shell script
74743:M 02 Nov 2020 02:21:26.020 # Setting secondary replication ID to a6d3f64c1b125dc3178e7bbf4abd3abedbbdfaa3, valid up to offset: 11372. New replication ID is 0f23bd1c62e6305778fe26e237e05cfbab645c50
74743:M 02 Nov 2020 02:21:26.020 * Discarding previously cached master state.
74743:M 02 Nov 2020 02:21:26.020 * MASTER MODE enabled (user request from 'id=4 addr=127.0.0.1:43396 fd=7 name= age=8 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=36 qbuf-free=32732 obl=0 oll=0 omem=0 events=r cmd=replicaof')
```
然后让6381追随6380节点，因为虽然6380已经变成了主节点，但是6381仍然追随的是6379节点，他并不知道主节点已经变了，6381仍在等待6379提供服务。
```shell script
[root@zhaoshuai ~]# redis-cli -p 6381
127.0.0.1:6381> REPLICAOF 127.0.0.1 6380
OK
```
查看6381节点日志：
```shell script
75200:S 02 Nov 2020 02:50:50.527 * REPLICAOF 127.0.0.1:6380 enabled (user request from 'id=4 addr=127.0.0.1:46092 fd=7 name= age=11 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=44 qbuf-free=32724 obl=0 oll=0 omem=0 events=r cmd=replicaof')
75200:S 02 Nov 2020 02:50:51.087 * Connecting to MASTER 127.0.0.1:6380
75200:S 02 Nov 2020 02:50:51.088 * MASTER <-> REPLICA sync started
75200:S 02 Nov 2020 02:50:51.088 * Non blocking connect for SYNC fired the event.
75200:S 02 Nov 2020 02:50:51.088 * Master replied to PING, replication can continue...
75200:S 02 Nov 2020 02:50:51.088 * Trying a partial resynchronization (request a6d3f64c1b125dc3178e7bbf4abd3abedbbdfaa3:11372).
75200:S 02 Nov 2020 02:50:51.088 * Successful partial resynchronization with master.
75200:S 02 Nov 2020 02:50:51.088 # Master replication ID changed to 0f23bd1c62e6305778fe26e237e05cfbab645c50
75200:S 02 Nov 2020 02:50:51.088 * MASTER <-> REPLICA sync: Master accepted a Partial Resynchronization.
```
这样当主节点挂机时，通过手动方式就将一个从节点升为了主机点。此时如果6379节点再次重新启动的话，那么只能让它作为一个从节点追随新的主节点。因为此时已经有主节点了。
在redis的配置文件中有与主从配置有关的：
```shell script
# 控制从节点是否是只读节点
replica-read-only yes
# 这个配置是当你的一个redis服务启动时，他会追随一个主节点，主节点数据量非常大的话，同步到从节点是需要时间的，那么在传输数据的时间内，从节点的数据是否对外支持
# 查询，yes的话就是支持查询老得数据，no就是不支持
replica-server-stale-data yes
# 复制策略：磁盘或套接字
# 磁盘IO： redis主节点数据落rdb，然后通过磁盘IO的方式同步到从节点。
# 网络IO: redis主节点落rdb文件后，直接将rdb文件传给副本集的套接字，不接触磁盘。
# 两种方式根据网络带宽选择，yes时表示开启套接字传输
repl-diskless-sync no
# 配置增量数据消息队列大小
# redis配置主从节点后，从节点会追随主节点，并在第一次启动时，加载主节点的rdb文件，后期如果从节点挂的话，会读取主节点的增量数据，这些增量数据就存放在消息队列中。
# 如果增加数据比消息队列大的话，数据就会丢失，从节点就需要重新加载主节点rdb。
repl-backlog-size 1mb

# 指定redis连接的最小副本集数量
min-replicas-to-write 3
# 每个副本集连接最大延迟时间
min-replicas-max-lag 10
# 当redis当前连接副本集小于指定数量并且延时大于最大延时数时，redis会拒绝写请求。

```
上面是通过手动切换主节点来实现故障转移，但是实现高可用的目标是自动故障转移。redis提供了哨兵机制--Sentinel来实现高可用  
#### redis哨兵
前面我们说CAP时，聊过通过监控来保证保证高可用，redis就是通过Sentinel哨兵来监控redis实例。
##### 如何启用哨兵？
首先我们先重启之前搭建的三台redis实例，恢复6379为主，6380、6381为从节点的主从复制副本集。查看日志，确保正常。   
在编译redis时，在 `/usr/local/bin`目录下，除了有 `redis-cli``redis-server`外，还有一个脚本`redis-sentinel`
```shell script
[root@zhaoshuai bin]# pwd
/usr/local/bin
[root@zhaoshuai bin]# ll
total 64500
-rwxr-xr-x. 1 root root 10423037 Oct 21 09:02 mysql
-rwxr-xr-x. 1 root root  9202939 Oct 20 14:46 redis-benchmark
-rwxr-xr-x. 1 root root 12277485 Oct 20 14:46 redis-check-aof
-rwxr-xr-x. 1 root root 12277485 Oct 20 14:46 redis-check-rdb
-rwxr-xr-x. 1 root root  9580048 Oct 20 14:46 redis-cli
lrwxrwxrwx. 1 root root       12 Oct 20 14:46 redis-sentinel -> redis-server
-rwxr-xr-x. 1 root root 12277485 Oct 20 14:46 redis-server
```
可以看到`redis-sentinel`是一个软连接，连接的时`redis-server`，也就是说也可以通过`redis-server`来启动一个哨兵。
一个哨兵监控一个redis实例，因此redis主从三个节点需要三个哨兵来监控。
##### 使用redis-sentinel来启动哨兵。
首先我们先执行一下 `redis-sentinel`脚本
```shell script
[root@zhaoshuai bin]# ./redis-sentinel 
77530:X 02 Nov 2020 18:56:14.552 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
77530:X 02 Nov 2020 18:56:14.552 # Redis version=5.0.5, bits=64, commit=00000000, modified=0, pid=77530, just started
77530:X 02 Nov 2020 18:56:14.552 # Warning: no config file specified, using the default config. In order to specify a config file use ./redis-sentinel /path/to/sentinel.conf
77530:X 02 Nov 2020 18:56:14.553 * Increased maximum number of open files to 10032 (it was originally set to 1024).
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 5.0.5 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in sentinel mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 26379
 |    `-._   `._    /     _.-'    |     PID: 77530
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           http://redis.io        
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

77530:X 02 Nov 2020 18:56:14.561 # Sentinel started without a config file. Exiting...
[root@zhaoshuai bin]# 
```
通过日志可以看到，启动失败，失败原因时没有配置文件，然后再网上看，发现默认的加载配置文件的路径是 `/path/to/sentinel.conf`。
那么我们就创建这个文件(这个文件的路径可以自定义), 每一个监控都有一个配置文件，我们就在redis的配置文件路径下创建哨兵的配置文件。
```shell script
[root@zhaoshuai /]# cd /etc/redis/
[root@zhaoshuai redis]# ll
total 192
-rw-r--r--. 1 root root 61916 Nov  1 23:40 6379.conf
-rw-r--r--. 1 root root 61898 Nov  1 23:56 6380.conf
-rw-r--r--. 1 root root 61898 Nov  2 01:55 6381.conf
[root@zhaoshuai redis]# touch 6379-sentinel.conf
[root@zhaoshuai redis]# vi 6379-sentinel.conf 
port 26379
sentinel monitor mymaster 127.0.0.1 6379 2
```
简单的配置文件，既然哨兵是一个特殊的redis-server，那么他也有端口号，因此要配置端口号， `sentinel monitor mymaster 127.0.0.1 6379 2`配置
sentinel表示这是一个哨兵配置，monitor监控，表示他要监控的节点是什么，mymaster表示给这个节点起的一个名字，随便起，127.0.0.1 6379 表示监控的节点信息
2表示权重，表示投票范围，也就是说，当主节点宕机时，有多少票说话才能管用。  
配置完成后启动哨兵
```shell script
[root@zhaoshuai redis]# redis-sentinel ./6379-sentinel.conf 
77680:X 02 Nov 2020 19:34:26.122 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
77680:X 02 Nov 2020 19:34:26.122 # Redis version=5.0.5, bits=64, commit=00000000, modified=0, pid=77680, just started
77680:X 02 Nov 2020 19:34:26.122 # Configuration loaded
77680:X 02 Nov 2020 19:34:26.123 * Increased maximum number of open files to 10032 (it was originally set to 1024).
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 5.0.5 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in sentinel mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 26379
 |    `-._   `._    /     _.-'    |     PID: 77680
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           http://redis.io        
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

77680:X 02 Nov 2020 19:34:26.126 # Sentinel ID is 2e7cbc211f52b61772e594a3e2d1bb1cf1a46322
77680:X 02 Nov 2020 19:34:26.126 # +monitor master mymaster 127.0.0.1 6379 quorum 2
77680:X 02 Nov 2020 19:34:26.127 * +slave slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6379
77680:X 02 Nov 2020 19:34:26.128 * +slave slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6379
```
启动成功了，而且我们只监控了主节点，打印的日志中自动加入了从节点的信息，因为从节点连接主节点，主节点中包含了从节点的信息。
监控肯定也是集群，不能是一台，否则的话无法投票，使用 `ctrl+C`退出当前哨兵。因为现在是前台阻塞的。  
删除原配置文件，因为打开看一下可以发现启动后添加了很多其他配置，我们删掉重新配置 ：
```shell script
port 26380
daemonize yes
logfile /var/log/sentinel-26380.log
sentinel monitor mymaster 127.0.0.1 6379 2
```
然后
```shell script
[root@zhaoshuai redis]# mv 6379-sentinel.conf 26379.conf
[root@zhaoshuai redis]# ll
total 196
-rw-r--r--. 1 root root   394 Nov  2 19:34 26379.conf
-rw-r--r--. 1 root root 61916 Nov  1 23:40 6379.conf
-rw-r--r--. 1 root root 61898 Nov  1 23:56 6380.conf
-rw-r--r--. 1 root root 61898 Nov  2 01:55 6381.conf
[root@zhaoshuai redis]# cp 26379.conf 26380.conf
[root@zhaoshuai redis]# cp 26379.conf 26381.conf
```
修改26380、26381配置文件中的端口号及日志文件名为相应端口号。
```shell script
[root@zhaoshuai redis]# ll
total 204
-rw-r--r--. 1 root root   444 Nov  2 19:48 26379.conf
-rw-r--r--. 1 root root   444 Nov  2 19:50 26380.conf
-rw-r--r--. 1 root root   444 Nov  2 19:50 26381.conf
-rw-r--r--. 1 root root 61916 Nov  1 23:40 6379.conf
-rw-r--r--. 1 root root 61898 Nov  1 23:56 6380.conf
-rw-r--r--. 1 root root 61898 Nov  2 01:55 6381.conf
[root@zhaoshuai redis]# vi 26380.conf 
port 26380
daemonize yes
logfile /var/log/sentinel-26380.log
sentinel monitor mymaster 127.0.0.1 6379 2
```
启动三个哨兵
```shell script
[root@zhaoshuai redis]# redis-sentinel 26379.conf 
[root@zhaoshuai redis]# redis-sentinel 26380.conf 
[root@zhaoshuai redis]# redis-sentinel 26381.conf 
[root@zhaoshuai redis]# ps -ef |grep redis-sentinel
root      77729      1  0 19:52 ?        00:00:00 redis-sentinel *:26379 [sentinel]
root      77734      1  0 19:52 ?        00:00:00 redis-sentinel *:26380 [sentinel]
root      77739      1  0 19:52 ?        00:00:00 redis-sentinel *:26381 [sentinel]
root      77744  77305  0 19:53 pts/1    00:00:00 grep redis-sentinel
```
此时再查看26379的日志：
```shell script
77890:X 02 Nov 2020 20:08:15.532 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
77890:X 02 Nov 2020 20:08:15.532 # Redis version=5.0.5, bits=64, commit=00000000, modified=0, pid=77890, just started
77890:X 02 Nov 2020 20:08:15.532 # Configuration loaded
77891:X 02 Nov 2020 20:08:15.535 * Increased maximum number of open files to 10032 (it was originally set to 1024).
77891:X 02 Nov 2020 20:08:15.535 * Running mode=sentinel, port=26379.
77891:X 02 Nov 2020 20:08:15.546 # Sentinel ID is da43e59879c95b6a5c2912856d67e9dafb25f2cb
77891:X 02 Nov 2020 20:08:15.546 # +monitor master mymaster 127.0.0.1 6379 quorum 2
77891:X 02 Nov 2020 20:08:15.546 * +slave slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6379
77891:X 02 Nov 2020 20:08:15.547 * +slave slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6379
77891:X 02 Nov 2020 20:08:21.408 * +sentinel sentinel 13d587768d58fa7d31a7be3bcec7634ef4e23cf8 127.0.0.1 26380 @ mymaster 127.0.0.1 6379
77891:X 02 Nov 2020 20:08:32.080 * +sentinel sentinel 342c6cdfe07f666fa860c59e56ba14f1fba07348 127.0.0.1 26381 @ mymaster 127.0.0.1 6379
```
另外两个监控的日志可以自己去看，发现启动多个哨兵实例时，会在日志后面加上添加新的哨兵。
此时我们就启动了三个节点，监控6379这个主节点。那么此时如果6379节点挂机呢？
```shell script
[root@zhaoshuai redis]# service redis_6379 stop
Stopping ...
Redis stopped
[root@zhaoshuai redis]# ps -ef |grep redis-server
root      77655      1  0 19:33 ?        00:00:01 /usr/local/bin/redis-server 127.0.0.1:6380      
root      77669      1  0 19:33 ?        00:00:01 /usr/local/bin/redis-server 127.0.0.1:6381      
root      77777  77305  0 19:58 pts/1    00:00:00 grep redis-server
```
可以看到6379节点已经停了，此时查看6380的日志：
```shell script
...
77871:S 02 Nov 2020 20:15:24.295 * Connecting to MASTER 127.0.0.1:6379
77871:S 02 Nov 2020 20:15:24.295 * MASTER <-> REPLICA sync started
77871:S 02 Nov 2020 20:15:24.295 # Error condition on socket for SYNC: Connection refused
77871:S 02 Nov 2020 20:15:25.314 * Connecting to MASTER 127.0.0.1:6379
77871:S 02 Nov 2020 20:15:25.314 * MASTER <-> REPLICA sync started
77871:S 02 Nov 2020 20:15:25.314 # Error condition on socket for SYNC: Connection refused
77871:M 02 Nov 2020 20:15:25.977 # Setting secondary replication ID to 308c17c2979464346c64b7d458292d0ef212b270, valid up to offset: 77458. New replication ID is c4d851bed85f97e717b50cc901068284d7e1c3fe
77871:M 02 Nov 2020 20:15:25.977 * Discarding previously cached master state.
77871:M 02 Nov 2020 20:15:25.977 * MASTER MODE enabled (user request from 'id=5 addr=127.0.0.1:44882 fd=8 name=sentinel-da43e598-cmd age=430 idle=0 flags=x db=0 sub=0 psub=0 multi=3 qbuf=140 qbuf-free=32628 obl=36 oll=0 omem=0 events=r cmd=exec')
77871:M 02 Nov 2020 20:15:25.978 # CONFIG REWRITE executed with success.
77871:M 02 Nov 2020 20:15:26.473 * Replica 127.0.0.1:6381 asks for synchronization
77871:M 02 Nov 2020 20:15:26.473 * Partial resynchronization request from 127.0.0.1:6381 accepted. Sending 289 bytes of backlog starting from offset 77458.
```
可以看到它会尝试连接6379，然后连几次后哨兵就讲6380给切换成了主节点，6381追随6380去了。
此时再重启6379节点：
```shell script
[root@zhaoshuai redis]# service redis_6379 start
Starting Redis server...
```
然后查看6379的日志：
```shell script
77982:S 02 Nov 2020 20:20:20.330 * Before turning into a replica, using my master parameters to synthesize a cached master: I may be able to synchronize with the new master with just a partial transfer.
77982:S 02 Nov 2020 20:20:20.330 * REPLICAOF 127.0.0.1:6380 enabled (user request from 'id=3 addr=127.0.0.1:37258 fd=8 name=sentinel-342c6cdf-cmd age=10 idle=0 flags=x db=0 sub=0 psub=0 multi=3 qbuf=148 qbuf-free=32620 obl=36 oll=0 omem=0 events=r cmd=exec')
77982:S 02 Nov 2020 20:20:20.331 # CONFIG REWRITE executed with success.
77982:S 02 Nov 2020 20:20:20.389 * Connecting to MASTER 127.0.0.1:6380
77982:S 02 Nov 2020 20:20:20.390 * MASTER <-> REPLICA sync started
77982:S 02 Nov 2020 20:20:20.390 * Non blocking connect for SYNC fired the event.
77982:S 02 Nov 2020 20:20:20.390 * Master replied to PING, replication can continue...
77982:S 02 Nov 2020 20:20:20.390 * Trying a partial resynchronization (request 6d075b3d0b0353bda449a4ae89551893eae917a0:1).
77982:S 02 Nov 2020 20:20:20.391 * Full resync from master: c4d851bed85f97e717b50cc901068284d7e1c3fe:135741
77982:S 02 Nov 2020 20:20:20.391 * Discarding previously cached master state.
77982:S 02 Nov 2020 20:20:20.523 * MASTER <-> REPLICA sync: receiving 218 bytes from master
77982:S 02 Nov 2020 20:20:20.523 * MASTER <-> REPLICA sync: Flushing old data
77982:S 02 Nov 2020 20:20:20.523 * MASTER <-> REPLICA sync: Loading DB in memory
77982:S 02 Nov 2020 20:20:20.523 * MASTER <-> REPLICA sync: Finished with success
```
可以看到重启后，6379也是追随6380，6380为主。
至此使用哨兵机制完整的实现了。然后我们在回过头打开`26379.conf`文件：
```shell script
port 26379
daemonize yes
logfile "/var/log/sentinel-26379.log"
sentinel myid da43e59879c95b6a5c2912856d67e9dafb25f2cb
# Generated by CONFIG REWRITE
dir "/etc/redis"
protected-mode no
sentinel deny-scripts-reconfig yes
sentinel monitor mymaster 127.0.0.1 6380 2
sentinel config-epoch mymaster 1
sentinel leader-epoch mymaster 1
sentinel known-replica mymaster 127.0.0.1 6379
sentinel known-replica mymaster 127.0.0.1 6381
sentinel known-replica mymaster 192.168.226.138 6379
sentinel known-sentinel mymaster 127.0.0.1 26380 13d587768d58fa7d31a7be3bcec7634ef4e23cf8
sentinel known-sentinel mymaster 127.0.0.1 26381 342c6cdfe07f666fa860c59e56ba14f1fba07348
sentinel current-epoch 1
```
注意，我们最开始配置，写的是监控主节点6379，但是经过一次重新选举后，6380成为了主节点master，因此哨兵自动修改了配置文件中主节点的信息从6379变成了6380。
至此完成了redis的哨兵配置。
##### redis的哨兵是如何发现其他哨兵的？
我们在配置哨兵的时候，只配置了主节点的信息，但是当主节点宕机时需要其他的哨兵发起投票选出新的master，那么一个哨兵是如何知道其他的哨兵的？
redis自带的功能就是发布订阅，当一个主节点启动的时候，哨兵就会在主节点身上进行发布订阅。
使用 `redis-cli -p 6380`连接redis的master。
然后使用psubscribe查看哨兵之间通信的消息：
````shell script
[root@zhaoshuai ~]# redis-cli -p 6380
127.0.0.1:6380> PSUBSCRIBE *
Reading messages... (press Ctrl-C to quit)
1) "psubscribe"
2) "*"
3) (integer) 1
1) "pmessage"
2) "*"
3) "__sentinel__:hello"
4) "127.0.0.1,26380,13d587768d58fa7d31a7be3bcec7634ef4e23cf8,1,mymaster,127.0.0.1,6380,1"
1) "pmessage"
2) "*"
3) "__sentinel__:hello"
4) "127.0.0.1,26381,342c6cdfe07f666fa860c59e56ba14f1fba07348,1,mymaster,127.0.0.1,6380,1"
1) "pmessage"
2) "*"
3) "__sentinel__:hello"
4) "127.0.0.1,26379,da43e59879c95b6a5c2912856d67e9dafb25f2cb,1,mymaster,127.0.0.1,6380,1"
1) "pmessage"
2) "*"
3) "__sentinel__:hello"
4) "127.0.0.1,26380,13d587768d58fa7d31a7be3bcec7634ef4e23cf8,1,mymaster,127.0.0.1,6380,1"
....
````
可以看到有一个 `__sentinel__:hello`的通道，然后26379，26380，26381都在这里面说话，所以只要有一个哨兵连接上主节点，那么其他的节点就能发现。
哨兵的配置文件在redis的源码中有sentinel.conf。
当哨兵通过master节点发现其他节点后，会在本地配置文件记录其他哨兵节点，然后哨兵之间除了通过master通信，也会有自己的发布订阅。
```shell script
[root@zhaoshuai redis-5.0.5]# redis-cli -p 26379
127.0.0.1:26379> PSUBSCRIBE *
Reading messages... (press Ctrl-C to quit)
1) "psubscribe"
2) "*"
3) (integer) 1
1) "pmessage"
2) "*"
3) "+sentinel-address-switch"
4) "master mymaster 127.0.0.1 6380 ip 192.168.226.138 port 26381 for 342c6cdfe07f666fa860c59e56ba14f1fba07348"
1) "pmessage"
2) "*"
...
```
哨兵之间通信的通道为 `+sentinel-address-switch`,查看此通道的信息：
```shell script
127.0.0.1:26379> SUBSCRIBE +sentinel-address-switch
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "+sentinel-address-switch"
3) (integer) 1
1) "message"
2) "+sentinel-address-switch"
3) "master mymaster 127.0.0.1 6380 ip 192.168.226.138 port 26380 for 13d587768d58fa7d31a7be3bcec7634ef4e23cf8"
1) "message"
2) "+sentinel-address-switch"
3) "master mymaster 127.0.0.1 6380 ip 127.0.0.1 port 26380 for 13d587768d58fa7d31a7be3bcec7634ef4e23cf8"
1) "message"
2) "+sentinel-address-switch"
3) "master mymaster 127.0.0.1 6380 ip 192.168.226.138 port 26380 for 13d587768d58fa7d31a7be3bcec7634ef4e23cf8"
1) "message"
2) "+sentinel-address-switch"
3) "master mymaster 127.0.0.1 6380 ip 127.0.0.1 port 26380 for 13d587768d58fa7d31a7be3bcec7634ef4e23cf8"
1) "message"
2) "+sentinel-address-switch"
3) "master mymaster 127.0.0.1 6380 ip 192.168.226.138 port 26381 for 342c6cdfe07f666fa860c59e56ba14f1fba07348"
1) "message"
2) "+sentinel-address-switch"
3) "master mymaster 127.0.0.1 6380 ip 127.0.0.1 port 26381 for 342c6cdfe07f666fa860c59e56ba14f1fba07348"
1) "message"
```
可以看到这个channel中订阅另外连个节点发布的信息。
### redis集群分片
上面解决了集群的主从复制以及高可用哨兵处理。也就是说根据AKF拆分原则，我们只实现了X轴的拓展。主从节点中的数据都是一样的，都是全量数据，那么容量的问题还没解决。  
回顾akf拆分原则，解决容量问题，可以向Y轴拓展，也就是根据业务进行数据分类，当某一类型数据量过大时，则向Z轴拓展，指定规则来对不同的数据指定存放节点。  
根据上面这种拆分原则的话，redis解决容量过大有如下方案：
+ 类型分类： 也就是向Y轴拓展，这样的话，就要求客户端对数据进行分类，比如说按照订单，支付等数据进行分类，然后将不同分类的数据存放到不同的redis服务。
也就是说，客户端需要对业务进行拆分，根据不同的业务选择不同的redis。
+ 根据规则分类： 也就是Z轴拓展，指定某一个规则来确定数据存放的位置。当某一业务的数据太大的时候，已经无法按照业务进行更详细的拆分了，只能向Z轴拓展，也就是说
指定一个算法，然后将数据按照这个算法均匀的分布到多个不同的节点。这个算法的实现有以下三种方案：
    - modula(hash+取模)： 也就是说对要存入的key先取hash值，然后在通过对redis节点的数量取模，确定要将数据存放到哪儿一个节点。但是这种方式的弊端就是无法进行分布式拓展。
    因为如果本来是两台节点，后期数据量变大了，变成三个，那么假设有一个key取hash后的值是11，取模后原来是存放在1节点上，然后现在拓展成3台，在对11取模就变成了2，
    也就是说数据取不出来了，因此这种方案的话，节点的数量就不能改变了。无法拓展。
    - random(lpush): random的意思是随机，也就是说我随机的往redis中存，随机就会有一个问题，客户端压根不知道存到哪儿台机器了，取得时候也不知道该去哪儿
    取，因此随机这种方案一般是存list，使用lpush，在每台服务器都会有一个一样的key，然后值的类型是list，通过lpush往里面存东西，另一个客户端不停的从这些list
    中取数据消费，不需要知道是从哪儿台机器取出来的，只要有就取，然后消费他。为了解决有可能消费失败的问题，还可以加一个缓冲。
    - kemata(一致性hash算法)：为每一个redis节点起一个id或者使用ip地址，每次计算时，key和node都要参与运算。一般会将它抽象成一个环形。  
    我们上面讲了hash取模，这里是一致性hash。两个都是hash，那么hash有什么特点？**映射**，也就是说无论你给我什么字符串，我最终都会给你映射成一个
    等长的数字。上面我们说了，一般使用环形来表示这个算法，就想象它是一个环，这个环上有很多的点，每一个点都是一个数字，这些点都是虚拟的。最终根据ip地址或
    node名/id等肯定能根据node信息经过hash运算在这个环上找到一个点，这个点就是物理的点。这个物理节点就表示一个redis服务节点，那么将key和node信息一起经过hash计算
    后，如果这个值距离这个物理节点更近，那么就讲这个值存入这个物理节点。
    这样的方案的缺点是：新增节点会有一小部分数据不能命中。  
    想象以下，本来两个物理节点，一个占圆的一半，那么node1占上半圆，node2占下半圆，然后现在新增一个节点，这样的话就是三个节点了，那么hash计算后发现离第三个
    节点较近，就去node3取数据，但是这个数据本来是存在node2上的，就会取不出来。
    解决方案有两种：
    + 第一种：缓存击穿。因为你本来能找到数据的，现在加了个节点找不到了，那只能将请求压到mysql，然后将mysql的数据重新在新的节点中存一份，这样以后就可以从redis中
    查到了。时间复杂度仍是O(1)。
    + 第二种：会增大时间复杂度。每次查的时候，会查命中的节点和节点两边的节点，这三个节点的数据挨个查一遍。那么如果一下增加两个节点的话，还是查不到。
    
    上面两种方式各有优缺点，只能人去选择怎么取舍。上面的方案还会带来一个新的问题，就是新增节点后，数据会存入新的节点，但是在原来节点中数据仍然存在，占着空间。
    所以这些数据必须清理，既然数据要清理，因此他只能用来作为缓存，因为数据库是不允许数据丢失的。数据清理策略有LRU，LFU等。
   
> 上面的方案都是在客户端计算的。方案只能对缓存使用。

客户端的问题解决了，但是作为客户端，肯定要与服务端进行连接，因为数据分片，数据有可能在node1中，也有可能在node2中，这时，每一个客户端既要与node1连接，又要与node2连接。
每一个连接都是十分损耗性能的。这个问题怎么解决？  
加入代理。代理服务器不干活，不存东西，不参与运算，只是建立连接。  
如果代理服务器的压力比较大，还可以做集群，keepalived+LVS
用的比较多的代理服务：twemproxy

#### cluster集群
进入utils目录，运行 `./create-cluster start`
```shell script
[root@zhaoshuai create-cluster]# ls
create-cluster  README
[root@zhaoshuai create-cluster]# ./create-cluster  start
Starting 30001
Starting 30002
Starting 30003
Starting 30004
Starting 30005
Starting 30006
```
就直接启动了一套redis集群

## 杂项补充：
对上面的内容中，没有牵扯到的内容进行一个补充。
### redis为什么快？
redis的速度非常快，单机的redis就可以支撑每秒十几万的并发，是mysql性能的几十倍。为什么这么快？
* redis是基于内存的，内存的i/o速度快。
* redis基于C语言开发，对数据结构做了优化。基于几种基础的数据结构，redis做了大量的优化，性能极高。
* redis是单线程的，没有上下文切换的开销
* 基于非阻塞IO的多路复用机制。
### 什么是热key，怎么解决？
热key问题就是，突然有几十万并发同时请求某个key，就会造成流量过于集中，达到物理网卡上限，导致redis服务器宕机引发雪崩。
解决办法：
1. 提前把热key打散到不同的服务器，降低压力
2. 加入二级缓存，提前把热key数据加载到本地内存中，当redis宕机时，走本地内存。
### redis为什么变慢了？
* 使用过于复杂的命令
* 存储大key
* 数据集中过期
* 实例内存达到上限
* fork耗时严重
* 绑定cpu
* aof分配不合理
* 使用swap













