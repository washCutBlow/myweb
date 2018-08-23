---
title: nginx看这一篇就够了！
---

- [1 什么是Nginx](#什么是nginx)
- [2 为什么是Nginx](#为什么是nginx)
  - [因为很吊](#1因为很吊体现在如下几个方面)
  - [为什么这么吊](#2为什么这么吊)
- [3 手摸手教你使用Nginx](#手摸手教你使用nginx)
  - [准备工作，可以不看](#准备工作)
  - [源码编译安装，不关心源码安装可以跳过](#源码编译安装)
  - [先上车掌握一些必会命令](#必须知道的命令)
  - [解剖nginx.conf文件配置，看不明白包打😄](#nginx配置)
  - [nginx反向代理&负载均衡的具体配置](#反向代理服务器配置)
- [4 Nginx进阶,敬请期待](#nginx进阶)
## 什么是nginx

- 2012年成长为世界第二大web服务器
- 业内高性能web服务器代名词
- 竞争对手
```
1 Apache
2 Lighttped（受欧美界青睐的，与nginx有的一拼的）
3 Tomcat("java语言web服务器    先天就是重量级性能跟nginx没法比")
4 Jetty("java语言web服务器     先天就是重量级性能跟nginx没法比")
5 IIS(window系统)
```
- 基于REST架构风格，以统一资源定位符(URI)货统一资源描述符(URL)作为沟通依据
- 基于事件驱动
- 高度模块化的设计----->第三方模块众多
- 可运行在众多平台
```
可以使用当前操作系统的高效API来提高自己的性能
支持linux上的epoll，epoll是大并网络连接的利器
```
## 为什么是nginx
##### 1.因为很吊，体现在如下几个方面
> 非要用一句话总结，那就是能够支持高并发请求的同时保持高效的服务
- ==更快== 单次请求得到更快的响应
- ==高扩展性==nginx设计极具扩展性，完全是由多个不同层次/不同功能/不同类型且耦合度极低的模块组成
    并且Nginx的模块都是嵌入到二进制文件中执行的

```
-- --比如HTTP模块中，还设计了HTTP过滤模块，一个正常的HTTP模块处理完请求后，会有一连串的HTTP过滤模块再对其进行过滤。
-- --我们开发一个新的HTTP模块时，可以使用HTTP核心模块  events模块  log模块等 还可以自由的复用各种过滤器模块
```
- ==高可靠性==
- ==低内存消耗==
    
```
一般情况下，10000个非活跃的HTTP Keep-Alive连接在Nginx中仅消耗2.5M内存
```
- ==单机支持10万以上的并发连接==
```
    理论上，Nginx的并发连接数仅取决于内存，10万远未封顶，当然与业务特点也紧密相连

```
- ==热部署==

```
由于master管理进程与worker工作进程的分离设计
使得nginx可以不停止服务就能升级可执行文件／更新配置项目／更换日志文件等
```
- ==nginx的架构设计很高明==

```
先天的事件驱动型设计
全异步的网络I/O处理机制
极少的进程间切换

```

- ==强大的开源社区== 数以万计的码畜们为nginx添砖加瓦
##### 2.为什么这么吊
这里我们着重讲解一下nginx使用的事件驱动架构，简单来说如下所示

```
graph LR
A[事件发生源]--产生事件-->B[事件收集器收集]
B--分发事件-->C[时件处理器注册自己感兴趣的事件并消费之]
```
事件源: 一般由网卡和磁盘产生

事件收集器: nginx的事件模块，如ngx_epoll_module

消费者: 所有其它模块
    
    消费者首先向事件模块注册自己感兴趣的事件类型，当该类型事件产生时，事件模块就会把事件分发到相应消费者模块

==nginx采用完全的事件驱动架构来处理业务，那它与传统的web服务器有哪些不同呢？==
- 传统web服务器（比如Apache）
    - Apache采用的所谓事件驱动仅仅是体现在TCP连接的建立和关闭事件上
        - 一个连接建立以后到关闭之前，所有的操作不再是事件操作，退化成了按序执行每个操作
        - 整个请求在连接期间始终占用cpu 内存资源，及时没有作任何有意义的事
        - ==把一个进程或线程作为事件消费者==，当一个请求产生事件被该进程处理时，直到请求处理结束时进程资源都将被这一个请求占用
- nginx服务器
    - ==不会使用进程或者线程作为事件消费者==，所谓的事件消费者只能是某个模块(在这里没有进程的概念)
    - 只有事件收集和分发器才有资格占用进程资源
- 重要差别
    - 前者是每个事件消费者独占一个进程资源，后者的事件消费者只是被事件分发者进程短期调用而已
- nginx这种设计的一个弊端
    - 即每个事件消费者都不能有阻塞行为，否则会长时间占用事件分发者进程而导致其它事件得不到及时响应，==尤其是每个消费者不可以让进程转为休眠或等待状态==，这都增加了码畜们的开发难度


## 手摸手教你使用nginx
##### 准备工作
==安装nginx开发所需的最基本的库==
以下是完成web服务器功能所需要的基本包
1. 使用uname -a 查看linuxe内核是否时2.6及以上版本
    >因为只有linux2.6及以上版本才支持epoll，能够更大限度发挥nginx的威力
2.  GCC编译器 使用yum install -y gcc安装
    > GCC(GNU Compile Collection)可以用来编译C语言程序，因为有时候nginx不会直接提供二进制可执行程序，需要自己编译
3. PCRE库
    >Perl正则兼容表达式包，pcre-devel时PCRE做二次开发时所需要的开发库，也是nginx开发所必需的
4. zlib库   yum install -y gzip gzip-devel
    >nginx.conf中配置了gzip on对Http 包的内容作gzip格式压缩，需要用
    zlip-devel是二次开发所需要的库
5. OpenSSL库 yum install -y openssl openssl-devel
    > 如果想使用更安全的SSL 协议传输HTTP，就需要该包，如果要使用MD5或者SHA1包，也需要Openssl包

==需要了解的几个目录==
1. nginx源码存放目录，随便放，没人管你，看个人癖好
2. Nginx编译阶段产生的中间目录

```
该目录用于存放configure和make命令执行后，生成的中间目录，默认情况下，生产objs目录存放在源码目录中
```

3. 部署目录默认 ==/usr/local/nginx==
```
存放nginx运行期间所需的二进制文件，配置文件等。
```

4. 日志文件存放目录

```
如果你要研究nginx的底层架构，那么打开debug级别日志后，会产生大量日志，所以最好弄一个大点的磁盘
```
##### 源码编译安装
- 官方下载nginx源码并解压后，cd到源码目录 ./configure && make && make install
   - 下面我们来看看这几个命令做了哪些见不得人的勾当
```
1 大部分工作其实都是configure命令做了，使用config  --help来查看都是有哪些命令 我们一般只关心以下几个
2 --prefix=PATH 安装目录，默认是/usr/local/nginx
  --sbin-path=PATH可执行文件防止路径,默认是?<prefix>/sbin/nginx
  --conf-path=PATH配置文件路径，默认是<prefix>/conf/nginx.conf
  --error-log-path=PATH错误日志文件，默认<prefix>/logs/error.log
    后面会在nginx.conf中详细介绍把不同请求的错误日志打到不同的log文件中
  --pid-path=PATH pid文件存放目录,默认<prefix>/logs/nginx.pid    这个文件以ascii码存放着nginx master的进程id
  
  更多的请自行搜索～～～～
  
```
##### 必须知道的命令
==/usr/local/nginx/sbin/nginx命令==  默认加载/usr/local/nginx/conf/nginx.conf

     /usr/local/nginx/sbin/nginx -c <配置文件目录>    来启动非默认的配置
    /usr/local/nginx/sbin/nginx -p <目录>           来指定nginx的安装目录 
    /usr/local/nginx/sbin/nginx -g 来临时指定一些全局配置项
        /usr/local/nginx/sbin/nginx -g "pid /var/nginx/test.pid;"
        意味着把pid写到另一个文件中，-g指定的不能与默认冲突，另外以-g启动的ngix在停止事也需要加上-g
        /usr/local/nginx/sbin/nginx -g "pid /var/nginx/test.pid;" -s stop.  如果不加-g 就找不到pid文件了
 ==/usr/local/nginx/sbin/nginx -t== 不启动nginx情况下，测试配置文件将是否有误
 
 ==/usr/local/nginx/sbin/nginx -V== 显示版本信息
 
 ==/usr/local/nginx/sbin/nginx -s stop== 快速停止服务处理完当前正在处理的请求后，关闭服务
 
 ==/usr/local/nginx/sbin/nginx -s reload== 运行中的nginx重新加载nginix.conf   等效于kill -s SIGHUP  <nginx master pid>
 
  ==/usr/local/nginx/sbin/nginx -s reopen== 等效于kill -s SIGUSR1  <nginx master pid>
  可以重新打开配置文件，这样我们就可以把当前的日志文件改名或者移动，使其不会太大
  
  ==平滑升级nginx==
  
  1.kill -s SIGUSR2  <nginx master pid> 会将nginx.pid重命名为nginx.pid.oldbin
  
  2.使用命令启动nginx
  
  3.使用kill -s SIGQUIT <旧版本的master pid> 关闭旧版本服务
  
  ##### nginx配置
  
```
1 生产环境一般都是一个master进程管理多个worker进程，worker进程与cpu核心数相等，每个worker都能繁忙的提供服务处理。
2 master进程只负责worker的管理。worker进程之间通过共享内存，原子操作等进程间通信机制实现负载均衡等功能
3 nginx是支持单进程 （只有一个master）提供服务的。使用master+worker的优势如下
    1master只提供纯管理工作，只提供命令行服务。
    2多worker进程可以提高服务的健壮型，可以充分利用多核cpu
```
==为什么需要把nginx的worker 进程数量设置的根cpu核心数一致呢==
1. 在apache上每个进程每一时刻只能处理一个请求，因此要想并发处理更多的并发请求数就要设置很多个进程，而大量的进程间切换很消耗内存资源
2. 而nginx的一个worker进程处理的请求数只受限于内存大小，并且worker进程之间处理并发请求时几乎没有同步锁的限制，worker进程也不会进入睡眠状态，因此当nginx进程与cpu核数一致时（最好每一个worker绑定一个内核），进程切换的代价是最小的
3. 将进程与cpu绑定，这样不会出现多个进程抢占一个cpu的问题，就不会出现同步问题，这样在内核的调度上就实现了完全的并发

==nginx.conf文件说明==
nginx的配置文件采用块配置项的组织方式，如下所示

```
全局配置
块配置项名1 {
    配置项名  配置项值1  配置项值1;
}
块配置项名2  参数 {
    配置项名  配置项值1  配置项值1;
}
基本的块配置项有：events  http  server  location  upstreams  快配置项可以嵌套
配置项名必须是nginx的某一个配置模块中想要处理的，否则会出错
如果配置项值中包括语法符号，比如空格，需要用引号括注配置项值
========
配置项的单位，如果是空间大小是 k或者m
有些模块允许配置项值中使用变量，变量前加上$
======
```
==一个具体的nginx.conf配置说明如下==可能包括以下几部分，这里我尽可能多的写几个哈～方便自己以后回头查阅
- 全局配置
    - user username [groupname];
        - 用于当master进程启动后,fork出的worker进程运行在哪个用户和用户组下
        - 需要在configure时使用参数 --user=username --group=groupname
    - daemo on|off;  是否以守护进程方式运行服务，默认on
    - master_process on|off;  是否以master/worker方式工作 默认on
    - error_log /path/file level; 错误日志  默认logs/error.log error
        - 可以是/dev/null,这样就不会有任何日志了，这也是关闭error日志的唯一手段了
        - 如果日志级别写成debug，那么在最初configure时需要加上--with-debug配置项
    - woker_rlimit_nofile limit; 一个worker进程可以打开的最大句柄描述符个数
    - worker_rlimit_core size;
    - worker_directory path;
        - 当进程意外终止时，nginx会把进程执行时的内存内容转储到一个core文件中，方便我们查看寄存器堆栈来定位问题，上面两个配置设定这个文件的大小和目录
    - env VAR|VAR=VALUE  这个配置项可以让用户直接操作系统变量
    - include ／path/file
        - 将其它配置文件嵌入到nginx.conf中，它的目录可以是绝对的，也可以是相对的，相对nginx.conf所在目录
    - worker_process 4；
    - worder_cpu_affinity 1000 0100 0010 0001
        - 上面两个配置将worder进程与cpu实现绑定
    - worker_priority nice  nginx 的进程优先级配置s
 - 事件类配置
 
    events {
        
    ```
    debug_connection IP;  只对该ip的请求才输出debug级别的日志，可以通过该方法定位bug
    accept_mutex [on|off]; 负载均衡锁，默认打开，如果关闭建立TCP链接的耗时更短，但每个worker的负载会非常不均衡
    lock_file path/file;  accept锁需要这个文件，如果由于程序的编译和操作系统的架构等因素导致nginx不支持原子锁, 就会用文件锁来实现accept锁。如果支持原子锁，这个文件就没意义了
    accept_mutex_delay Nms; 使用文件锁后，同一时间只有一个worker能获得到这个锁,这个锁不是阻塞所,worker获取不到，会立即返回,然后间隔这个时间之后再去获取。
    multi_accept on|off;默认关闭，当事件模型通知有新连接时，尽可能的对本次调度中客户端发起的所有的TCP请求都去建立连接
    use [poll| select | epoll| kqueue]; Nginx 默认会选择最合适的事件模型
    woker_connections  number; 每个worker进程可以同时处理的最大连接数
    ```
    }
- http模块

    ==静态web服务器主要由nginx中的ngx_http_core_module实现==
    http模块是一个最小静态web服务器的基本配置
    
    http {
        
    ```
   gzip on;
   server {
         listen address:port;  address可以是ip或者hostname
                    在port后可以加上一些参数 如下所示
                    listen 443 default_server ssl deferred; 
                    deault_server当一个请求无法匹配所有的域名后，使用这个作为默认处理域名
                    ssl 在当前端口的连接必须是基于SSL协议
                    deferred用户发起建立连接请求，完成TCP三次握手之后，内核也不会调度worker进程来处理这次链接，
                    只有用户真的发了数据(网卡收到请求数据包)内核才会唤醒worker进程去处理,
        server_name name ;后面可以跟多个主机名称,用顿号隔开
                   'nginx 收到请求后，会先取出Header头中的host，根server中去比较，如果匹配了多个server,会根据匹配优先级来选择用哪个server,
                    如果都找不到，就用server_name 为空的server块'
        server_name_in_redirect on|off 默认为on
                   '如果为开启，那么首先查找server_name，如果没有找到，查找请求头的HOST字段，如果没有，则以当前服务器的IP进行拼接
        location [=|~|~*|^~|@] /uri/ {} uri参数里可以用正则
                   location = / {} 用户请求是/时匹配
                   ～ URI大小写敏感
                   ~* 匹配URI时忽略大小写
                   ^~前半部分大小写敏感匹配，如
                        location ^~ /images/ {}以/images/ 开始的请求都会匹配
                    @表示nginx内部请求之间的重定向,不直接处理用户请求
        root path   文件路径，默认是root html
                    root配置还可以位于http模块下或者location下，如果位于location下含义如下
                        location /download/ {
                            root /opt/web/html/;
                        }
                        如果用户的请求是/download/test.html,web服务器将会返回服务器上的/opt/web/html/download/test.html文件中内容
         location /conf { location下配置说明
            alias /usr/local/nginx/conf/
                如果用户请求是/conf/nginx.conf,用户实际想访问/usr/local/nginx/conf/nginx.conf,就可以使用alias配置
                alias只能放在location中
            root path;
            index 首页文件;
         }
         error_page code uri|@named_location  也可以在location块中配置
            可以作如下配置
                error_code 404 /404.html
                error_code 501 502 504 /50x.html
                error_code 403 http://example.com/forbidden.html
                error_code 404 = @fetch
            也可以改变错误码
                error_page 404 =200 /empty.gif
                或者不指定错误码 ，由重定向后的实际处理决定
                error_page 404  /empty.gif
            如果不想修改uri,只是想定向到另一个location中处理，可以如下配置
                location / {
                    error_code 404 @fallback
                }
                
                location @fallback {
                    proxy_pass http://backend;
                }
        try_files path1 path2 uri 尝试访问每一个path找到了就结束请求 ，都找不到久落到了uri上,所以uri必须存在
        type {  MIME类型设置,可以位于server location块中
            type说白了就是不同的文件类型用不同的应用程序打开。
        }
            
   }
   "http中还有贼多的配置 比如tcp网络链接 内存资源管理 对客户端请求的限制  文件操作的优化等等，等用到了可以具体研究"
    ```
    }
    
    ##### 反向代理服务器配置
>   nginx的高并发高负载能力相当彪悍，一般可以直接作为web服务器向用户提供静态文件服务。但有些复杂业务不适合直接放在nginx服务器上，这时候会用Apache等服务器来处理,使用nginx作为静态web服务器和反反向代理服务器，不适合nginx处理的请求直接转到上游服务器处理

==nginx反向代理方式==

    HTTP请求--->nginx将请求内容落地到所在服务器硬盘或内存---->向上游服务器发起连接
- 优点：降低上游业务服务器的负载,尽量把压力放在nginx服务器上
    - 因为客户端与代理服务器之间走公网，网络环境复杂，而代理服务器与上游业务服务器走的是内部专网，在接收完用户请求后，在内网转发会相当快。如果是一边接收一边转发，外网的烂速度就会拖垮内网。
- 缺点: 延长了请求处理时间，增加了nginx服务器的内存和磁盘空间

==负载均衡配置==在http模块中

http {
    
```
upstream backen {
    ip_hash;
    server backend1.example.com  weight=5;
    server backend2.example.com  max_failes3  fail_timeout=30s;
}
    #定义了一个叫作backend的上游服务器集群
    #server 后可以跟域名 IP地址端口等
    #weight转发权重
    #max_failem默认为1,0表示不检查失败次数;faile_timeout默认为10s   30s内转发失败3次，认为该服务器不可用
    #ip_hash保证同一个ip的请求落到同一个服务器上,不能与weight同时使用，
    并且如果服务器集群中有一台机器挂掉，不能直接删除该配置, 而要使用down进行标示，以保证转发策略的一致性
    #反向代理还提供了一些变量如:$remote_addr $time_local  $request 等等 可以在access_log的log_format日志格式配置中使用？？？
```

} 

==反向代理配置==在location模块中,配置如下


```
upstream backen {
.....
}

server{
    location /{
        proxy_pass http://backend;
        proxy_set_header Host $host
            #默认情况反向代理是不会转发请求中的host头部,如果需要转发，使用proxy_set_header配置
    }
}
```
## nginx进阶

1. Nginx 采用的是多进程（单线程） & 多路IO复用模型。并发事件驱动
- 流程说明
1. 主进程（master 进程）首先通过 socket() 来创建一个 sock 文件描述符用来监听，然后fork生成子进程
2. 子进程将继承父进程的 sockfd
3. 之后子进程 accept() 后将创建已连接描述，然后通过已连接描述符来与客户端通信
```
graph TB
A[管理员]--信号-->B[master进程]
B--信号-->C[worker进程]
B--信号-->D[worker进程]
B--信号-->E[worker进程]
```
==由于所有子进程都继承了父进程的 sockfd，那么当连接进来时，所有子进程都将收到通知并“争着”与它建立连接，这就叫“惊群现象”。大量的进程被激活又挂起，只有一个进程可以accept() 到这个连接，这当然会消耗系统资源==

#### Nginx对惊群现象的处理
> Nginx 提供了一个 accept_mutex 这个东西，这是一个加在accept上的一把共享锁。即每个 worker 进程在执行 accept 之前都需要先获取锁，获取不到就放弃执行 accept()。有了这把锁之后，同一时刻，就只会有一个进程去 accpet()，这样就不会有惊群问题了。accept_mutex 是一个可控选项，我们可以显示地关掉，默认是打开的

#### 多进程模型每个进程/线程只能处理一路IO，那么 Nginx是如何处理多路IO呢？

>1 如果不使用 IO 多路复用，那么在一个进程中，同时只能处理一个请求，比如执行 accept()，如果没有连接过来，那么程序会阻塞在这里，直到有一个连接过来，才能继续向下执行

>2 Nginx采用的 IO多路复用模型epoll

>3 epoll通过在Linux内核中申请一个简易的文件系统（文件系统一般用什么数据结构实现？B+树），其工作流程分为三部分

```
1、调用 int epoll_create(int size)建立一个epoll对象，内核会创建一个eventpoll结构体，用于存放通过epoll_ctl()向epoll对象中添加进来
	的事件，这些事件都会挂载在红黑树中。
2、调用 int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event) 在 epoll 对象中为 fd 注册事件，所有添加到epoll中的件
	都会与设备驱动程序建立回调关系，也就是说，当相应的事件发生时会调用这个sockfd的回调方法，将sockfd添加到eventpoll 中的双链表
3、调用 int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout) 来等待事件的发生，timeout 为 -1 时，该
	调用会阻塞知道有事件发生
```
这样，注册好事件之后，只要有 fd 上事件发生，epoll_wait() 就能检测到并返回给用户，用户就能”非阻塞“地进行 I/O 了。

>4 epoll() 中内核则维护一个链表，epoll_wait 直接检查链表是不是空就知道是否有文件描述符准备好了。（epoll 与 select 相比最大的优点是不会随着 sockfd 数目增长而降低效率，使用 select() 时，内核采用轮训的方法来查看是否有fd 准备好，其中的保存 sockfd 的是类似数组的数据结构 fd_set，key 为 fd，value 为 0 或者 1。）

>5 能达到这种效果，是因为在内核实现中 epoll 是根据每个 sockfd 上面的与设备驱动程序建立起来的回调函数实现的。那么，某个 sockfd 上的事件发生时，与它对应的回调函数就会被调用，来把这个 sockfd 加入链表，其他处于“空闲的”状态的则不会。在这点上，epoll 实现了一个”伪”AIO。但是如果绝大部分的 I/O 都是“活跃的”，每个 socket 使用率很高的话，epoll效率不一定比 select 高（可能是要维护队列复杂）。

>6 例子：Nginx 会注册一个事件：“如果来自一个新客户端的连接请求到来了，再通知我”，此后只有连接请求到来，服务器才会执行 accept() 来接收请求。又比如向上游服务器（比如 PHP-FPM）转发请求，并等待请求返回时，这个处理的 worker 不会在这阻塞，它会在发送完请求后，注册一个事件：“如果缓冲区接收到数据了，告诉我一声，我再将它读进来”，于是进程就空闲下来等待事件发生。

这样，基于 多进程+epoll， Nginx 便能实现高并发。
#### Nginx 与 多进程模式 Apache 的比较
>1 对于Apache，每个请求都会独占一个工作线程，当并发数到达几千时，就同时有几千的线程在处理请求了。这对于操作系统来说，占用的内存非常大，线程的上下文切换带来的cpu开销也很大，性能就难以上去，同时这些开销是完全没有意义的

 对于Nginx来讲，一个进程只有一个主线程，通过异步非阻塞的事件处理机制，实现了循环处理多个准备好的事件，从而实现轻量级和高并发
 
 #### 事件驱动适合于I/O密集型服务，多进程或线程适合于CPU密集型服务
 1 Nginx 更主要是作为反向代理，而非Web服务器使用。其模式是事件驱动
 
 2 事件驱动服务器，最适合做的就是这种 I/O 密集型工作，如反向代理，它在客户端与WEB服务器之间起一个数据中转作用，纯粹是 I/O 操作，自身并不涉及到复杂计算。因为进程在一个地方进行计算时，那么这个进程就不能处理其他事件了
 
 3 Nginx 只需要少量进程配合事件驱动，几个进程跑 libevent，不像 Apache 多进程模型那样动辄数百的进程数
 
 4 Nginx 处理静态文件效果也很好，那是因为读写文件和网络通信其实都是 I/O操作，处理过程一样
