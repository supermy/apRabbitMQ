rabbitmq
====================

介绍
---------------------
##一键启动

*   fig up -d && fig ps
*   执行 test *sh 脚本测试
*   配置查看队列信息 http://127.0.0.1:15672/#/  guest/guest

##构建镜像包

    docker build -t supermy/ap-rabbitmq  rabbitmq


##业务场景1-数据同步

*   实现一个topic 的路由，实现消息的自动匹配
*   详见 test-sync-data.sh

##业务场景2-日志记录

*   定义日志的路由与消息队列
*   log_by_lua_file 调用 rabbitmq-插入日志数据；
*   详见 test-sync-log.sh

##业务场景3-消息事件触发

*   定义消息时间路由与队列
*   消息推送消费API的绑定； todo 通用方式
*   非阻塞消息推送
*   同日志记录，实现方式一样

##业务场景3-RabbitMQ15672的 nginx 代理配置
    
    location /rabbitmq {
                rewrite         /rabbitmq/(.*) /$1 break;
                proxy_pass      http://127.0.0.1:15672;
                proxy_redirect  off;
                proxy_set_header Host $host;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

## 有api 接口，可以前置WAF进行负载均衡和安全防护。

       location /mqapi {
                client_body_buffer_size 128k;
                proxy_send_timeout   90;
                proxy_read_timeout   90;
                proxy_buffer_size    4k;
                proxy_buffers     16 32k;
                proxy_busy_buffers_size 64k;
                proxy_temp_file_write_size 64k;
                proxy_connect_timeout 30s;
                proxy_pass   http://192.168.102.110:15672/api;
                proxy_set_header   Host   $host;
                proxy_set_header   X-Real-IP  $remote_addr;
                proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        }

###RabbitMQ
+ RabbitMQ是实现AMQP（高级消息队列协议）的消息中间件的一种，最初起源于金融系统，用于在分布式系统中存储转发消息，在易用性、扩展性、高可用
性等方面表现不俗。消息中间件主要用于组件之间的解耦，消息的发送者无需知道消息使用者的存在，反之亦然。

###Redis
+ 是一个Key-Value的NoSQL数据库，开发维护很活跃，虽然它是一个Key-Value数据库存储系统，但它本身支持MQ功能，所以完全可以当做一个轻量级的队
列服务来使用。

###可靠消费

+ Redis：没有相应的机制保证消息的消费，当消费者消费失败的时候，消息体丢失，需要手动处理
+ RabbitMQ：具有消息消费确认，即使消费者消费失败，也会自动使消息体返回原队列，同时可全程持久化，保证消息体被正确消费

###可靠发布

+ Reids：不提供，需自行实现
+ RabbitMQ：具有发布确认功能，保证消息被发布到服务器

###高可用

+ Redis：采用主从模式，读写分离，但是故障转移还没有非常完善的官方解决方案
+ RabbitMQ：集群采用磁盘、内存节点，任意单点故障都不会影响整个队列的操作

###持久化

+ Redis：将整个Redis实例持久化到磁盘
+ RabbitMQ：队列，消息，都可以选择是否持久化

###消费者负载均衡

+ Redis：不提供，需自行实现
+ RabbitMQ：根据消费者情况，进行消息的均衡分发

###队列监控

+ Redis：不提供，需自行实现
+ RabbitMQ：后台可以监控某个队列的所有信息，（内存，磁盘，消费者，生产者，速率等）

###流量控制

+ Redis：不提供，需自行实现
+ RabbitMQ：服务器过载的情况，对生产者速率会进行限制，保证服务可靠性

###出入队性能

+ 对于RabbitMQ和Redis的入队和出队操作，各执行100万次，每10万次记录一次执行时间。
测试数据分为128Bytes、512Bytes、1K和10K四个不同大小的数据。

###应用场景分析

+ Redis：轻量级，高并发，延迟敏感
即时数据分析、秒杀计数器、缓存等

+ RabbitMQ：重量级，高并发，异步
批量数据异步处理、并行任务串行化，高负载任务的负载均衡等
