---
layout: post
title: Rabbitmq 高可用
slug: Rabbitmq_HA
date: 2020-1-19 15:23
status: publish
author: 酒后的阿bill
categories:
  - message
tags:
  - Rabbitmq
excerpt: Rabbitmq 高可用
---

# 一 HA方案
使用rabbitmq集群，一方面是想通过集群提升消息的处理速度；另一方面，防止一个节点不可用导致整个平台出现故障。使用集群后，当一个节点出现故障后，需要某种方法使得客户端感知，切换到无故障的节点，达到HA 的目的。

### rabbimq 有两种HA 方案：

- 增加集群管理器（如：Pacemaker）在集群node 出故障时感知并处理

- 客户端感知故障，自动切换

对于以上两种，增加pacemaker在编写客户端时，不用考虑怎么处理集群故障。客户端实现比较简单。增加pacemaker 相当于增加了一层，所以又要考虑pacemaker的HA，pacemaker成为了单点，其出现故障将会是整个集群的故障。

客户端自行感知故障，需要实现 pacemaker 类似的功能： 感知故障，切换到正确节点，恢复状态。给客户端的编写带来复杂度。但是也因为少了一层，故障域会减少。

以上两种功能，出现故障，客户端的连接切换到另一个节点时，可以得到故障节点相同的信息。rabbitmq 本身提供了这方面的功能：镜像队列。

镜像队列可以保证多个节点有相同的信息，这样就使得客户端发现故障在重连时恢复至故障之前的状态。

## 1.1 rabbitmq 镜像队列
从 3.6.0 版本开始，RabbitMQ 支持镜像队列功能，官方文档在这里。与普通集群相比，其实质和普通模式不同之处在于，消息实体会主动在镜像节点间同步，而不是在 consumer 取数据时临时拉取。该模式带来的副作用也很明显，除了降低系统性能外，如果镜像队列数量过多，加之大量的消息进入，集群内部的网络带宽将会被这种同步通讯大大消耗掉。所以在对可靠性要求较高的场合中适用。 

- Mirrorred queue 是 RabbitMQ 高可用的一种方案，相对于普通的集群方案来讲，queue中的消息每个节点都会存在一份 copy, 这个在单个节点失效的情况下，整个集群仍旧可以提供服务。但是由于数据需要在多个节点复制，在增加可用性的同时，系统的吞吐量会有所下降。 

- 选举机制：mirror queue 内部实现了一套选举算法，有一个 master 和多个slave，queue 中的消息以 master 为主。镜像队列有主从之分，一个主节点(master)，0个或多个从节点(slave)。当master宕掉后，会在 slave 中选举新的master，其选举算法为最早启动的节点。 若master节点失效，则 mirror queue 会自动选举出一个节点（slave中消息队列最长者）作为master，作为消息消费的基准参考; 在这种情况下可能存在ack消息未同步到所有节点的情况(默认异步)，若 slave 节点失效，mirror queue 集群中其他节点的状态无需改变。所以，看起来可以使用两节点的镜像队列集群。 

- 使用：对于publish，可以选择任意一个节点进行连接，rabbitmq内部若该节点不是master，则转发给master，master向其他slave节点发送该消息，后进行消息本地化处理，并组播复制消息到其他节点存储；对于consumer，可以选择任意一个节点进行连接，消费的请求会转发给master,为保证消息的可靠性，consumer需要进行ack确认，master收到ack后，才会删除消息，ack消息会同步(默认异步)到其他各个节点，进行slave节点删除消息。   


## 1.2 镜像队列的A/A + Haproxy

http://greenstack.die.upm.es/2015/03/02/improving-ha-failures-with-tcp-timeouts/ 这篇文章详细讲解了当一个节点down掉后,怎么处理.这里依然要启用镜像队列,且引入了HAproxy这个组件,增加了风险
好处：
haproxy 可以分散连接，不至于一个节点负载过重。

## 1.3 镜像队列A/A + 客户端 heartbeat
客户端保证HA，
openstack 对于mq 的使用，封装了一个 sdk–oslo_message,并对mq 的集群场景做了优化，如下：

- 配置多个mq node
- heartbeat 功能检测异常(rpc server 使用另一种检查方式，类似 heartbeat )
- 打散连接防止一个节点负担过重
以上三个功能，可以使得客户端自行判断集群是否出了故障。这样就可以不使用LB ,减少了因Haproxy故障带来的问题。也因为没有中间件，缩小了故障域，客户端保证HA 

# 二 rabbitmq HA

## 2.1 集群要求

一般我们集群使用三个节点，在集群能力不够时，我们可以适当增加  rabbitmq 节点个数，横向扩展增加集群吞吐量。
针对三个节点，需要保证其中一个节点或者两个节点down掉，集群依然可以继续对外提供服务。所以要求

- 尽量在三个不同的机柜上
- 连接三个不同的交换机

## 2.2 集群配置

三个节点，两个DISK节点，一个RAM节点


# 三 集群降级策略
思考的问题：

#### 1 在HA 已经满足的条件下，什么时候需要切换到备用集群
rabbitmq node 尽量在三个不同的机柜上，连接三个不同的交换机。如果在这种情况下发生故障，如果需要对外继续提供您服务，需要考虑的连同 controller 节点一起，做到机房的HA。这种情况下，api 上层负载均衡可以做到流量切换。mq 降级到备份集群。

#### 2 备用集群怎么自动切换
- 手动切换oslo 配置
- 修改oslo message 组件，当满足一定条件后自动切换
