## 消息队列协议

### 为什么消息中间件不直接使用http协议呢？

1、因为http请求报文头和响应报文头是比较复杂的，包含了cookie，数据的加密解密，状态码，响应码等附加的功能，但是对于一个消息而言，我们并不需要这么复杂，也没有这个必要性，它其实就是负责数据传递，存储，分发就行，一定要追求的是高性能。尽量简洁，快速。
2、大部分情况下http大部分都是短链接，在实际的交互过程中，一个请求到响应很有可能会中断，中断以后就不会就行持久化，就会造成请求的丢失。这样就不利于消息中间件的业务场景，因为消息中间件可能是一个长期的获取消息的过程，出现问题和故障要对数据或消息就行持久化等，目的是为了保证消息和数据的高可靠和稳健的运行。
常见的消息中间件协议有：OpenWire、AMQP、MQTT、Kafka，OpenMessage协议。

### AMQP协议

AMQP：(全称：Advanced Message Queuing Protocol) 是高级消息队列协议。由摩根大通集团联合其他公司共同设计。是一个提供统一消息服务的应用层标准高级消息队列协议，是应用层协议的一个开放标准，为面向消息的中间件设计。基于此协议的客户端与消息中间件可传递消息，并不受客户端/中间件不同产品，不同的开发语言等条件的限制。Erlang中的实现有RabbitMQ等。
特性：
1、分布式事务支持。
2、消息的持久化支持。
3、高性能和高可靠的消息处理优势。

### MQTT协议

MQTT协议：（Message Queueing Telemetry Transport）消息队列是IBM开放的一个即时通讯协议，物联网系统架构中的重要组成部分。
特点：
1、轻量
2、结构简单
3、传输快，不支持事务
4、没有持久化设计。
应用场景：
1、适用于计算能力有限
2、低带宽
3、网络不稳定的场景。

### OpenMessage协议

是近几年由阿里、雅虎和滴滴出行、Stremalio等公司共同参与创立的分布式消息中间件、流处理等领域的应用开发标准。
特点：
1、结构简单
2、解析速度快
3、支持事务和持久化设计。

### Kafka协议

Kafka协议是基于TCP/IP的二进制协议。消息内部是通过长度来分割，由一些基本数据类型组成。
特点是：
1、结构简单
2、解析速度快
3、无事务支持
4、有持久化设计

## 消息队列持久化

简单来说就是将数据存入磁盘，而不是存在内存中随服务器重启断开而消失，使数据能够永久保存。

### 常见的持久化方式

|          | ActiveMQ | RabbitMQ | Kafka | RocketMQ |
| -------- | -------- | -------- | ----- | -------- |
| 文件存储 | 支持     | 支持     | 支持  | 支持     |
| 数据库   | 支持     | /        | /     | /        |

## 消息的分发策略

### 消息的分发策略

MQ消息队列有如下几个角色
1、生产者
2、存储消息
3、消费者
那么生产者生成消息以后，MQ进行存储，消费者是如何获取消息的呢？一般获取数据的方式无外乎推（push）或者拉（pull）两种方式，典型的git就有推拉机制，我们发送的http请求就是一种典型的拉取数据库数据返回的过程。而消息队列MQ是一种推送的过程，而这些推机制会适用到很多的业务场景也有很多对应推机制策略。

### 消息分发策略的机制和对比

|          | ActiveMQ | RabbitMQ | Kafka | RocketMQ |
| -------- | -------- | -------- | ----- | -------- |
| 发布订阅 | 支持     | 支持     | 支持  | 支持     |
| 轮询分发 | 支持     | 支持     | 支持  | /        |
| 公平分发 | /        | 支持     | 支持  | /        |
| 重发     | 支持     | 支持     | /     | 支持     |
| 消息拉取 | /        | 支持     | 支持  | 支持     |

## 消息队列高可用和高可靠

### 什么是高可用机制

所谓高可用：是指产品在规定的条件和规定的时刻或时间内处于可执行规定功能状态的能力。
当业务量增加时，请求也过大，一台消息中间件服务器的会触及硬件（CPU,内存，磁盘）的极限，一台消息服务器你已经无法满足业务的需求，所以消息中间件必须支持集群部署。来达到高可用的目的。

### 什么是高可靠机制

所谓高可用是指：是指系统可以无故障低持续运行，比如一个系统突然崩溃，报错，异常等等并不影响线上业务的正常运行，出错的几率极低，就称之为：高可靠。
在高并发的业务场景中，如果不能保证系统的高可靠，那造成的隐患和损失是非常严重的。
如何保证中间件消息的可靠性呢？可以从两个方面考虑：
1、消息的传输：通过协议来保证系统间数据解析的正确性。
2、消息的存储可靠：通过持久化来保证消息的可靠性。

### Master-slave主从共享数据的部署方式

![img](https://kuangstudy.oss-cn-beijing.aliyuncs.com/bbs/2022/02/09/kuangstudye20835d0-e533-46e1-a08c-a966790e2385.png)
生产者讲消费发送到Master节点，所有的都连接这个消息队列共享这块数据区域，Master节点负责写入，一旦Master挂掉，slave节点继续服务。从而形成高可用。

### Master- slave主从同步部署方式

![img](https://kuangstudy.oss-cn-beijing.aliyuncs.com/bbs/2022/02/09/kuangstudyc3c114ec-9cff-45a4-a99c-cfd300ae9137.png)
这种模式写入消息同样在Master主节点上，但是主节点会同步数据到slave节点形成副本，和zookeeper或者redis主从机制很类同。这样可以达到负载均衡的效果，如果消费者有多个这样就可以去不同的节点就行消费，以为消息的拷贝和同步会暂用很大的带宽和网络资源。在后续的rabbtmq中会有使用。

### 多主集群同步部署模式

![img](https://kuangstudy.oss-cn-beijing.aliyuncs.com/bbs/2022/02/09/kuangstudy5ae23dfe-ba37-42b3-b299-b9f2106e2b0e.png)
和上面的区别不是特别的大，但是它的写入可以往任意节点去写入。

### 多主集群转发部署模式

![img](https://kuangstudy.oss-cn-beijing.aliyuncs.com/bbs/2022/02/09/kuangstudy4c5f5b5f-6834-4a88-b3ce-a8f3a5ab74a6.png)
如果你插入的数据是broker-1中，元数据信息会存储数据的相关描述和记录存放的位置（队列）。
它会对描述信息也就是元数据信息就行同步，如果消费者在broker-2中进行消费，发现自己几点没有对应的消息，可以从对应的元数据信息中去查询，然后返回对应的消息信息，场景：比如买火车票或者黄牛买演唱会门票，比如第一个黄牛有顾客说要买的演唱会门票，但是没有但是他会去联系其他的黄牛询问，如果有就返回。

### Master-slave与Breoker-cluster组合的方案

![img](https://kuangstudy.oss-cn-beijing.aliyuncs.com/bbs/2022/02/09/kuangstudy731bae41-6cfc-4949-b019-8d8bec1f2421.png)
实现多主多从的热备机制来完成消息的高可用以及数据的热备机制，在生产规模达到一定的阶段的时候，这种使用的频率比较高。

### 总结

1、要么消息共享
2、要么消息同步
3、要么元数据共享

## RabbitMQ入门及安装

### 安装RabbitMQ

1、下载地址：https://www.rabbitmq.com/download.html
2、环境准备：CentOS7.x+ / Erlang
RabbitMQ是采用Erlang语言开发的，所以系统环境必须提供Erlang环境，第一步就是安装Erlang。
![img](https://kuangstudy.oss-cn-beijing.aliyuncs.com/bbs/2022/02/09/kuangstudycaf736fd-b427-483f-b480-49e852d40c0b.png)

### Erlang安装

#### 查看系统版本号

```
[root@iZm5eauu5f1ulwtdgwqnsbZ ~]# lsb_release -aLSB Version:    :core-4.1-amd64:core-4.1-noarchDistributor ID: CentOSDescription:    CentOS Linux release 8.3.2011Release:        8.3.2011Codename:       n/a
```

#### 安装下载

参考地址：https://www.erlang-solutions.com/downloads/

```
wget https://packages.erlang-solutions.com/erlang-solutions-2.0-1.noarch.rpmrpm -Uvh erlang-solutions-2.0-1.noarch.rpm
yum install -y erlang
```

#### 安装成功

```
erl -v
```

### 安装socat

```
yum install -y socat
```

### 安装rabbitmq

下载地址：https://www.rabbitmq.com/download.html
![img](https://kuangstudy.oss-cn-beijing.aliyuncs.com/bbs/2022/02/09/kuangstudyd2307692-d67a-469e-9ae0-3ebcdc470cec.png)

#### 下载rabbitmq

```
wget https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.8.13/rabbitmq-server-3.8.13-1.el8.noarch.rpmrpm -Uvh rabbitmq-server-3.8.13-1.el8.noarch.rpm
```

#### 启动rabbitmq服务

##### 启动服务

```
systemctl start rabbitmq-server
```

##### 查看服务状态

```
systemctl status rabbitmq-server
```

##### 停止服务

```
systemctl stop rabbitmq-server
```

##### 开机启动服务

```
systemctl enable rabbitmq-server
```

### RabbitMQ的配置

RabbitMQ默认情况下有一个配置文件，定义了RabbitMQ的相关配置信息，默认情况下能够满足日常的开发需求。如果需要修改需要，需要自己创建一个配置文件进行覆盖。
参考官网：
1、https://www.rabbitmq.com/documentation.html
2、https://www.rabbitmq.com/configure.html
3、https://www.rabbitmq.com/configure.html#config-items
4、https://github.com/rabbitmq/rabbitmq-server/blob/add-debug-messages-to-quorum_queue_SUITE/docs/rabbitmq.conf.example

##### 相关端口

5672:RabbitMQ的通讯端口
25672:RabbitMQ的节点间的CLI通讯端口是
15672:RabbitMQ HTTP_API的端口，管理员用户才能访问，用于管理RabbitMQ,需要启动Management插件。
1883，8883：MQTT插件启动时的端口。
61613、61614：STOMP客户端插件启用的时候的端口。
15674、15675：基于webscoket的STOMP端口和MOTT端口
tip：RabbitMQ 在安装完毕以后，会绑定一些端口，如果你购买的是阿里云或者腾讯云相关的服务器一定要在安全组中把对应的端口添加到防火墙。

## RabbitMQWeb管理界面及授权操作

默认情况下，rabbitmq是没有安装web端的客户端插件，需要安装才可以生效

```
rabbitmq-plugins enable rabbitmq_management
```

rabbitmq有一个默认账号和密码是：guest 默认情况只能在localhost本机下访问，所以需要添加一个远程登录的用户。

### 安装完毕以后，重启服务即可

```
systemctl restart rabbitmq-server
```

tip：一定要记住，在对应服务器(阿里云，腾讯云等)的安全组中开放15672的端口。

### 授权账号和密码

#### 新增用户

```
rabbitmqctl add_user admin admin
```

#### 设置用户分配操作权限

```
rabbitmqctl set_user_tags admin administrator
```

#### 用户级别

1、administrator 可以登录控制台、查看所有信息、可以对rabbitmq进行管理
2、monitoring 监控者 登录控制台，查看所有信息
3、policymaker 策略制定者 登录控制台,指定策略
4、managment 普通管理员 登录控制台

#### 为用户添加资源权限

```
rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
```

### 小结

rabbitmqctl add_user 账号 密码
rabbitmqctl set_user_tags 账号 administrator
rabbitmqctl change_password Username Newpassword 修改密码
rabbitmqctl delete_user Username 删除用户
rabbitmqctl list_users 查看用户清单
rabbitmqctl set_permissions -p / 用户名 “.*“ “.*“ “.*“ 为用户设置administrator角色
rabbitmqctl set_permissions -p / root “.*“ “.*“ “.*“

## Docker安装RabbitMQ

### 虚拟化容器技术—Docker的安装

```
（1）yum 包更新到最新> yum update（2）安装需要的软件包， yum-util 提供yum-config-manager功能，另外两个是devicemapper驱动依赖的> yum install -y yum-utils device-mapper-persistent-data lvm2（3）设置yum源为阿里云> yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo（4）安装docker> yum install docker-ce -y（5）安装后查看docker版本> docker -v (6) 安装加速镜像 sudo mkdir -p /etc/docker sudo tee /etc/docker/daemon.json <<-'EOF' {  "registry-mirrors": ["https://0wrdwnn6.mirror.aliyuncs.com"] } EOF sudo systemctl daemon-reload sudo systemctl restart docker
```

### docker的相关命令

```
# 启动docker：systemctl start docker# 停止docker：systemctl stop docker# 重启docker：systemctl restart docker# 查看docker状态：systemctl status docker# 开机启动：  systemctl enable dockersystemctl unenable docker# 查看docker概要信息docker info# 查看docker帮助文档docker --help
```

### 安装rabbitmq

#### 获取rabbit镜像

```
docker pull rabbitmq:management
```

#### 创建并运行容器

```
docker run -di --name=myrabbit -p 15672:15672 rabbitmq:management
```

—hostname：指定容器主机名称
—name：指定容器名称
-p：将mq端口号映射到本地
或者运行时设置用户和密码

```
docker run -di --name myrabbit -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=admin -p 15672:15672 -p 5672:5672 -p 25672:25672 -p 61613:61613 -p 1883:1883 rabbitmq:management
```

#### 查看日志

```
docker logs -f myrabbit
```

#### 容器运行正常

使用 [http://你的IP地址:15672](http://xn--ip-0p3ck01akcu41v:15672/) 访问rabbit控制台

#### 额外Linux相关排查命令

```
> more xxx.log  查看日记信息> netstat -naop | grep 5672 查看端口是否被占用> ps -ef | grep 5672  查看进程> systemctl stop 服务
```

## RabbitMQ的角色分类

### none

- 不能访问management plugin

  ### management

- 列出自己可以通过AMQP登入的虚拟机

- 查看自己的虚拟机节点 virtual hosts的queues,exchanges和bindings信息

- 查看和关闭自己的channels和connections

- 查看有关自己的虚拟机节点virtual hosts的统计信息。包括其他用户在这个节点virtual hosts中的活动信息。

  ### Policymaker

- 包含management所有权限

- 查看和创建和删除自己的virtual hosts所属的policies和parameters信息。

  ### Monitoring

- 包含management所有权限

- 罗列出所有的virtual hosts，包括不能登录的virtual hosts。

- 查看其他用户的connections和channels信息

- 查看节点级别的数据如clustering和memory使用情况

- 查看所有的virtual hosts的全局统计信息。

  ### Administrator

- 最高权限

- 可以创建和删除virtual hosts

- 可以查看，创建和删除users

- 查看创建permisssions

- 关闭所有用户的connections

  ## Simple 简单模式

  https://www.kuangstudy.com/zl/rabbitmq#1366642676568551426

  ## 什么是AMQP

  ### 什么是AMQP

  AMQP全称：Advanced Message Queuing Protocol(高级消息队列协议)。是应用层协议的一个开发标准，为面向消息的中间件设计。

  ### AMQP生产者流转过程

  

  ### AMQP消费者流转过程

  

  ## RabbitMQ的核心组成部分

  

  Server：又称Broker ,接受客户端的连接，实现AMQP实体服务。 安装rabbitmq-server。

  Connection：连接，应用程序与Broker的网络连接 TCP/IP/ 三次握手和四次挥手。

  Channel：网络信道，几乎所有的操作都在Channel中进行，Channel是进行消息读写的通道，客户端可以建立多个Channel，每个Channel代表一个会话任务。

  Message：消息，服务与应用程序之间传送的数据，由Properties和body组成，Properties可是对消息进行修饰，比如消息的优先级，延迟等高级特性，Body则就是消息体的内容。

  Virtual Host：虚拟地址，用于进行逻辑隔离，最上层的消息路由，一个虚拟主机理由可以有若干个Exhange和Queueu，同一个虚拟主机里面不能有相同名字的Exchange。

  Exchange：交换机，接受消息，根据路由键发送消息到绑定的队列。(==不具备消息存储的能力==)

  Bindings：Exchange和Queue之间的虚拟连接，binding中可以保护多个routing key。

  Routing key：是一个路由规则，虚拟机可以用它来确定如何路由一个特定消息。

  Queue：队列，也成为Message Queue,消息队列，保存消息并将它们转发给消费者。

  ## RabbitMQ支持消息的模式

  参考官网：

  https://www.rabbitmq.com/getstarted.html

  ### RabbitMQ的模式之发布订阅模式

  Fanout—发布与订阅模式，是一种广播机制，它是没有路由key的模式。

  ### RabbitMQ的模式之Direct模式

  Direct模式是fanout模式上的一种叠加，增加了路由RoutingKey的模式。

  ### RabbitMQ的模式之Topic模式

  Topic模式是direct模式上的一种叠加，增加了模糊路由RoutingKey的模式。

  .#代表没有或者一级或者多级 0~N

  .*代表至少一级 1

  ### RabbitMQ的模式之Headers模式

  参数匹配模式

  ### RabbitMQ的模式之Work模式

  1、轮询模式的分发：一个消费者一条，按均分配；

  2、公平分发：根据消费者的消费能力进行公平分发，处理快的处理的多，处理慢的处理的少；按劳分配；

  该模式接收消息是当有多个消费者接入时，消息的分配模式是一个消费者分配一条，直至消息消费完成；

  ## 死信队列

  DLX，全称为Dead-Letter-Exchange , 可以称之为死信交换机，也有人称之为死信邮箱。当消息在一个队列中变成死信(dead message)之后，它能被重新发送到另一个交换机中，这个交换机就是DLX ，绑定DLX的队列就称之为死信队列。

  消息变成死信，可能是由于以下的原因：

- 消息被拒绝

- 消息过期

- 队列达到最大长度

  DLX也是一个正常的交换机，和一般的交换机没有区别，它能在任何的队列上被指定，实际上就是设置某一个队列的属性。当这个队列中存在死信时，Rabbitmq就会自动地将这个消息重新发布到设置的DLX上去，进而被路由到另一个队列，即死信队列。

  要想使用死信队列，只需要在定义队列的时候设置队列参数 x-dead-letter-exchange 指定交换机即可。

  

  ## 持久化机制和内存磁盘的监控

  ### RibbitMQ持久化

  持久化就把信息写入到磁盘的过程。

  ### RabbitMQ持久化消息

  

  把消息默认放在内存中是为了加快传输和消费的速度，存入磁盘是保证消息数据的持久化。

  ### RabbitMQ非持久化消息

  非持久消息：是指当内存不够用的时候，会把消息和数据转移到磁盘，但是重启以后非持久化队列消息就丢失。

  ### RabbitMQ持久化分类

  RabbitMQ的持久化队列分为：

  1、队列持久化

  2、消息持久化

  3、交换机持久化

  不论是持久化的消息还是非持久化的消息都可以写入到磁盘中，只不过非持久的是等内存不足的情况下才会被写入到磁盘中。

  ### 内存磁盘的监控

  #### RabbitMQ的内存警告

  当内存使用超过配置的阈值或者磁盘空间剩余空间对于配置的阈值时，RabbitMQ会暂时阻塞客户端的连接，并且停止接收从客户端发来的消息，以此避免服务器的崩溃，客户端与服务端的心态检测机制也会失效。

  

  #### RabbitMQ的内存控制

  参考帮助文档：

  https://www.rabbitmq.com/configure.html

  当出现警告的时候，可以通过配置去修改和调整

  ##### 命令的方式

  ```
  rabbitmqctl set_vm_memory_high_watermark <fraction>rabbitmqctl set_vm_memory_high_watermark absolute 50MB
  ```

  fraction/value 为内存阈值。默认情况是：0.4/2GB，代表的含义是：当RabbitMQ的内存超过40%时，就会产生警告并且阻塞所有生产者的连接。通过此命令修改阈值在Broker重启以后将会失效，通过修改配置文件方式设置的阈值则不会随着重启而消失，但修改了配置文件一样要重启broker才会生效。

  ##### 配置文件方式 rabbitmq.conf

  当前配置文件：/etc/rabbitmq/rabbitmq.conf

  ```
  #默认
  #vm_memory_high_watermark.relative = 0.4
  # 使用relative相对值进行设置fraction,建议取值在04~0.7之间，不建议超过0.7.vm_memory_high_watermark.relative = 0.6# 使用absolute的绝对值的方式，但是是KB,MB,GB对应的命令如下
  vm_memory_high_watermark.absolute = 2GB
  ```

  #### RabbitMQ的内存换页

  在某个Broker节点及内存阻塞生产者之前，它会尝试将队列中的消息换页到磁盘以释放内存空间，持久化和非持久化的消息都会写入磁盘中，其中持久化的消息本身就在磁盘中有一个副本，所以在转移的过程中持久化的消息会先从内存中清除掉。

  ```
  默认情况下，内存到达的阈值是50%时就会换页处理。也就是说，在默认情况下该内存的阈值是0.4的情况下，当内存超过0.4*0.5=0.2时，会进行换页动作。
  ```

  比如有1000MB内存，当内存的使用率达到了400MB,已经达到了极限，但是因为配置的换页内存0.5，这个时候会在达到极限400mb之前，会把内存中的200MB进行转移到磁盘中。从而达到稳健的运行。

  可以通过设置 vm_memory_high_watermark_paging_ratio 来进行调整

  ```
  vm_memory_high_watermark.relative = 0.4vm_memory_high_watermark_paging_ratio = 0.7（设置小于1的值）
  ```

  #### RabbitMQ的磁盘预警

  当磁盘的剩余空间低于确定的阈值时，RabbitMQ同样会阻塞生产者，这样可以避免因非持久化的消息持续换页而耗尽磁盘空间导致服务器崩溃。

  ```
  默认情况下：磁盘预警为50MB的时候会进行预警。表示当前磁盘空间第50MB的时候会阻塞生产者并且停止内存消息换页到磁盘的过程。这个阈值可以减小，但是不能完全的消除因磁盘耗尽而导致崩溃的可能性。比如在两次磁盘空间的检查空隙内，第一次检查是：60MB ，第二检查可能就是1MB,就会出现警告。
  ```

  通过命令方式修改如下：

  ```
  rabbitmqctl set_disk_free_limit  <disk_limit>rabbitmqctl set_disk_free_limit memory_limit  <fraction>disk_limit：固定单位 KB MB GBfraction ：是相对阈值，建议范围在:1.0~2.0之间。（相对于内存）
  ```

  通过配置文件配置如下：

  ```
  disk_free_limit.relative = 3.0disk_free_limit.absolute = 50mb
  ```

  ## 集群

  ### RabbitMQ 集群

  RabbitMQ这款消息队列中间件产品本身是基于Erlang编写，Erlang语言天生具备分布式特性（通过同步Erlang集群各节点的magic cookie来实现）。因此，RabbitMQ天然支持Clustering。这使得RabbitMQ本身不需要像ActiveMQ、Kafka那样通过ZooKeeper分别来实现HA方案和保存集群的元数据。集群是保证可靠性的一种方式，同时可以通过水平扩展以达到增加消息吞吐量能力的目的。

  在实际使用过程中多采取多机多实例部署方式，为了便于同学们练习搭建，有时候你不得不在一台机器上去搭建一个rabbitmq集群，本章主要针对单机多实例这种方式来进行开展。

  ### 单机多实例搭建

  场景：假设有两个rabbitmq节点，分别为rabbit-1, rabbit-2，rabbit-1作为主节点，rabbit-2作为从节点。

  启动命令：RABBITMQ_NODE_PORT=5672 RABBITMQ_NODENAME=rabbit-1 rabbitmq-server -detached

  结束命令：rabbitmqctl -n rabbit-1 stop

  #### 第一步：启动第一个节点rabbit-1

  ```
  > sudo RABBITMQ_NODE_PORT=5672 RABBITMQ_NODENAME=rabbit-1 rabbitmq-server start &...............省略...................##########  Logs: /var/log/rabbitmq/rabbit-1.log######  ##        /var/log/rabbitmq/rabbit-1-sasl.log##########            Starting broker...completed with 7 plugins.
  ```

  至此节点rabbit-1启动完成。

  #### 启动第二个节点rabbit-2

  注意：web管理插件端口占用,所以还要指定其web插件占用的端口号

  RABBITMQ_SERVER_START_ARGS=”-rabbitmq_management listener [{port,15673}]”

  ```
  sudo RABBITMQ_NODE_PORT=5673 RABBITMQ_SERVER_START_ARGS="-rabbitmq_management listener [{port,15673}]" RABBITMQ_NODENAME=rabbit-2 rabbitmq-server start &..............省略..................##########  Logs: /var/log/rabbitmq/rabbit-2.log######  ##        /var/log/rabbitmq/rabbit-2-sasl.log##########            Starting broker...completed with 7 plugins.
  ```

  至此节点rabbit-2启动完成

  #### 验证启动 “ps aux|grep rabbitmq”

  ```
  rabbitmq  2022  2.7  0.4 5349380 77020 ?       Sl   11:03   0:06 /usr/lib/erlang/erts-9.2/bin/beam.smp -W w -A 128 -P 1048576 -t 5000000 -stbt db -zdbbl 128000 -K true -B i -- -root /usr/lib/erlang -progname erl -- -home /var/lib/rabbitmq -- -pa /usr/lib/rabbitmq/lib/rabbitmq_server-3.6.15/ebin -noshell -noinput -s rabbit boot -sname rabbit-1 -boot start_sasl -kernel inet_default_connect_options [{nodelay,true}] -rabbit tcp_listeners [{"auto",5672}] -sasl errlog_type error -sasl sasl_error_logger false -rabbit error_logger {file,"/var/log/rabbitmq/rabbit-1.log"} -rabbit sasl_error_logger {file,"/var/log/rabbitmq/rabbit-1-sasl.log"} -rabbit enabled_plugins_file "/etc/rabbitmq/enabled_plugins" -rabbit plugins_dir "/usr/lib/rabbitmq/plugins:/usr/lib/rabbitmq/lib/rabbitmq_server-3.6.15/plugins" -rabbit plugins_expand_dir "/var/lib/rabbitmq/mnesia/rabbit-1-plugins-expand" -os_mon start_cpu_sup false -os_mon start_disksup false -os_mon start_memsup false -mnesia dir "/var/lib/rabbitmq/mnesia/rabbit-1" -kernel inet_dist_listen_min 25672 -kernel inet_dist_listen_max 25672 startrabbitmq  2402  4.2  0.4 5352196 77196 ?       Sl   11:05   0:05 /usr/lib/erlang/erts-9.2/bin/beam.smp -W w -A 128 -P 1048576 -t 5000000 -stbt db -zdbbl 128000 -K true -B i -- -root /usr/lib/erlang -progname erl -- -home /var/lib/rabbitmq -- -pa /usr/lib/rabbitmq/lib/rabbitmq_server-3.6.15/ebin -noshell -noinput -s rabbit boot -sname rabbit-2 -boot start_sasl -kernel inet_default_connect_options [{nodelay,true}] -rabbit tcp_listeners [{"auto",5673}] -sasl errlog_type error -sasl sasl_error_logger false -rabbit error_logger {file,"/var/log/rabbitmq/rabbit-2.log"} -rabbit sasl_error_logger {file,"/var/log/rabbitmq/rabbit-2-sasl.log"} -rabbit enabled_plugins_file "/etc/rabbitmq/enabled_plugins" -rabbit plugins_dir "/usr/lib/rabbitmq/plugins:/usr/lib/rabbitmq/lib/rabbitmq_server-3.6.15/plugins" -rabbit plugins_expand_dir "/var/lib/rabbitmq/mnesia/rabbit-2-plugins-expand" -os_mon start_cpu_sup false -os_mon start_disksup false -os_mon start_memsup false -mnesia dir "/var/lib/rabbitmq/mnesia/rabbit-2" -rabbitmq_management listener [{port,15673}] -kernel inet_dist_listen_min 25673 -kernel inet_dist_listen_max 25673 start
  ```

  #### rabbit-1操作作为主节点

  ```
  #停止应用> sudo rabbitmqctl -n rabbit-1 stop_app#目的是清除节点上的历史数据（如果不清除，无法将节点加入到集群）> sudo rabbitmqctl -n rabbit-1 reset#启动应用> sudo rabbitmqctl -n rabbit-1 start_app
  ```

  #### rabbit2操作为从节点

  ```
  # 停止应用> sudo rabbitmqctl -n rabbit-2 stop_app# 目的是清除节点上的历史数据（如果不清除，无法将节点加入到集群）> sudo rabbitmqctl -n rabbit-2 reset# 将rabbit2节点加入到rabbit1（主节点）集群当中【Server-node服务器的主机名】> sudo rabbitmqctl -n rabbit-2 join_cluster rabbit-1@'Server-node'# 启动应用> sudo rabbitmqctl -n rabbit-2 start_app
  ```

  #### 验证集群状态

  ```
  > sudo rabbitmqctl cluster_status -n rabbit-1//集群有两个节点：rabbit-1@Server-node、rabbit-2@Server-node[{nodes,[{disc,['rabbit-1@Server-node','rabbit-2@Server-node']}]},{running_nodes,['rabbit-2@Server-node','rabbit-1@Server-node']},{cluster_name,<<"rabbit-1@Server-node.localdomain">>},{partitions,[]},{alarms,[{'rabbit-2@Server-node',[]},{'rabbit-1@Server-node',[]}]}]
  ```

  ```
  Tips：如果采用多机部署方式，需读取其中一个节点的cookie, 并复制到其他节点（节点之间通过cookie确定相互是否可通信）。cookie存放在/var/lib/rabbitmq/.erlang.cookie。例如：主机名分别为rabbit-1、rabbit-21、逐个启动各节点2、配置各节点的hosts文件( vim /etc/hosts) ip1：rabbit-1 ip2：rabbit-2其它步骤雷同单机部署方式
  ```

  ## 分布式事务

  ### 可靠生产

  

  ### 可靠消费

  

  ### 基于MQ的分布式事务解决方案缺点

  1、基于消息中间件，只适合异步场景

  2、消息会延迟处理，需要业务上能够容忍

  ### 建议

  1、尽量去避免分布式事务

  2、尽量将非核心业务做成异步