# 数据订阅(TMQ)

<cite>
**本文档引用的文件**
- [tmq.c](file://examples/c/tmq.c)
- [taos.h](file://include/client/taos.h)
- [clientTmq.c](file://source/client/src/clientTmq.c)
</cite>

## 目录
1. [引言](#引言)
2. [TMQ设计原理与系统角色](#tmq设计原理与系统角色)
3. [与传统消息队列的异同](#与传统消息队列的异同)
4. [主题创建与SQL语法](#主题创建与sql语法)
5. [生产者与消费者编程](#生产者与消费者编程)
6. [同步与异步消费模式](#同步与异步消费模式)
7. [持久化机制与分区策略](#持久化机制与分区策略)
8. [消费组概念](#消费组概念)
9. [应用场景](#应用场景)
10. [配置参数说明](#配置参数说明)
11. [性能调优建议](#性能调优建议)
12. [故障恢复策略](#故障恢复策略)

## 引言
TDengine消息队列（TMQ）是TDengine数据库内置的实时数据订阅和消息传递系统。它允许用户通过创建主题（Topic）来订阅数据库中的数据变更，实现数据的实时流式处理。TMQ的设计旨在为时序数据提供高效、可靠的消息传递机制，支持多种编程语言的连接器，并提供丰富的API来满足不同的应用场景需求。

## TMQ设计原理与系统角色
TMQ的设计原理基于发布-订阅模式，它将数据库中的数据变更作为消息发布到主题中，消费者可以订阅这些主题来接收数据。TMQ在系统中扮演着数据管道的角色，连接数据生产者和消费者，实现数据的解耦和异步处理。通过TMQ，用户可以构建实时数据处理应用，如实时监控、告警系统和数据同步等。

**Section sources**
- [taos.h](file://include/client/taos.h#L394-L428)
- [clientTmq.c](file://source/client/src/clientTmq.c#L333-L554)

## 与传统消息队列的异同
TMQ与Kafka等传统消息队列在设计上有相似之处，如都支持发布-订阅模式和消费组概念。然而，TMQ是专门为时序数据设计的，它直接集成在TDengine数据库中，无需额外部署消息队列服务。这使得TMQ在处理时序数据时具有更低的延迟和更高的效率。此外，TMQ的主题是通过SQL语句创建的，这使得它与数据库的集成更加紧密。

**Section sources**
- [taos.h](file://include/client/taos.h#L394-L428)
- [clientTmq.c](file://source/client/src/clientTmq.c#L333-L554)

## 主题创建与SQL语法
在TDengine中，可以通过SQL语句创建主题。使用`CREATE TOPIC`语句，可以指定主题名称和查询语句，该查询语句定义了哪些数据将被发布到主题中。例如，`CREATE TOPIC topicname AS SELECT ts, c1, c2, c3, tbname FROM tmqdb.stb WHERE c1 > 1`创建了一个名为`topicname`的主题，它将`tmqdb.stb`表中`c1`大于1的数据发布到主题中。

**Section sources**
- [tmq.c](file://examples/c/tmq.c#L150-L155)

## 生产者与消费者编程
TMQ的生产者是数据库本身，当数据被写入数据库时，如果存在匹配的主题，数据将自动被发布到相应的主题中。消费者则需要通过C API或各语言连接器来实现。使用C API，可以通过`taos_consume`函数来消费消息。首先，需要创建一个消费者配置，然后使用该配置创建消费者实例，最后通过`tmq_consumer_poll`函数来轮询消息。

**Section sources**
- [taos.h](file://include/client/taos.h#L402-L409)
- [clientTmq.c](file://source/client/src/clientTmq.c#L1675-L2626)

## 同步与异步消费模式
TMQ支持同步和异步两种消费模式。在同步模式下，消费者通过`tmq_consumer_poll`函数轮询消息，该函数会阻塞直到有消息到达或超时。在异步模式下，消费者可以注册一个回调函数，当有消息到达时，系统会自动调用该回调函数来处理消息。异步模式适合处理高吞吐量的消息流，可以避免轮询带来的资源浪费。

**Section sources**
- [taos.h](file://include/client/taos.h#L406-L407)
- [clientTmq.c](file://source/client/src/clientTmq.c#L2582-L2626)

## 持久化机制与分区策略
TMQ的持久化机制依赖于TDengine数据库的WAL（Write-Ahead Log）机制。当数据被写入数据库时，首先会被写入WAL，然后才被应用到数据库中。TMQ通过读取WAL来获取数据变更，确保了数据的持久性和一致性。TMQ的分区策略基于vgroup（虚拟组），每个vgroup可以独立处理数据，实现了水平扩展和负载均衡。

**Section sources**
- [clientTmq.c](file://source/client/src/clientTmq.c#L3368-L3574)
- [clientTmq.c](file://source/client/src/clientTmq.c#L3602-L3712)

## 消费组概念
消费组是TMQ中的一个重要概念，它允许多个消费者实例协同工作，共同消费一个主题中的消息。每个消费组内的消费者实例会被分配不同的分区，确保每条消息只被消费一次。消费组还支持动态加入和退出，系统会自动重新分配分区，保证了系统的高可用性和弹性。

**Section sources**
- [taos.h](file://include/client/taos.h#L403-L404)
- [clientTmq.c](file://source/client/src/clientTmq.c#L1835-L1986)

## 应用场景
TMQ在数据管道、微服务通信和事件溯源等场景下有广泛的应用。在数据管道中，TMQ可以作为数据源和数据目的地之间的桥梁，实现实时数据同步。在微服务通信中，TMQ可以作为服务间通信的中间件，实现服务的解耦和异步处理。在事件溯源中，TMQ可以记录系统的状态变更，为系统提供审计和回溯能力。

**Section sources**
- [tmq.c](file://examples/c/tmq.c#L1-L333)

## 配置参数说明
TMQ提供了丰富的配置参数来满足不同的需求。例如，`enable.auto.commit`参数控制是否自动提交消费偏移量，`auto.commit.interval.ms`参数设置自动提交的间隔时间，`group.id`参数指定消费组的ID。这些参数可以通过`tmq_conf_set`函数来设置，为消费者提供灵活的配置选项。

**Section sources**
- [taos.h](file://include/client/taos.h#L394-L396)
- [clientTmq.c](file://source/client/src/clientTmq.c#L333-L554)

## 性能调优建议
为了获得最佳性能，建议根据实际应用场景调整TMQ的配置参数。例如，对于高吞吐量的场景，可以增加`auto.commit.interval.ms`的值，减少自动提交的频率，从而降低系统开销。对于低延迟的场景，可以减小`fetch.max.wait.ms`的值，使消费者更快地获取到新消息。此外，合理设置消费组的大小，可以平衡负载和资源利用率。

**Section sources**
- [taos.h](file://include/client/taos.h#L410-L412)
- [clientTmq.c](file://source/client/src/clientTmq.c#L2917-L3019)

## 故障恢复策略
TMQ的故障恢复策略依赖于消费偏移量的持久化。当消费者重启时，可以从上次提交的偏移量处继续消费，确保不会丢失消息。此外，通过合理设置`session.timeout.ms`和`heartbeat.interval.ms`参数，可以及时检测到消费者故障，并触发重新平衡，将故障消费者的分区重新分配给其他消费者。

**Section sources**
- [taos.h](file://include/client/taos.h#L417-L419)
- [clientTmq.c](file://source/client/src/clientTmq.c#L3259-L3366)