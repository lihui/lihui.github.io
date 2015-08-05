---
layout: post
title: note
author: 灵元
---


编程模型，并发、分区复制机制，实现原理，决定配置参数。

综合博文
注意：调研流处理的主要问题， 常用算法，常用模式。

然后叙述比较storm，samza的实现。

storm 叙述编程模型
所支持的功能，如送达保证，管理
实现原理


大数据

streamid： task可能产生多个stream，通过使用一个string来表示区分stream。在构造topology的时候，消息接收task通过制定消息生产端的taskid和streamid来唯一确定stream，如果没有设置streamid，则默认为defaultstream。消息生产端的task在发送tuple的时候，也需要指定streamid，若未指定，则为默认defaultstream。


storm的ack机制

当spout需要构造一个需要确认的消息包时，消息包可以是一个tuple，但也可以是多个tuple，以便获得某种程度的原子性，若果spout有多个下游task时，会有多个tuple。为此消息包生成一个messageId，称为rootId，其他多个tuple也各自生成messageId。将rootId，messageIds的xor值，和spoutId一起打包发送给acker，acker记录下rootId，ackVal，以及对应的spoutId。给下游bolt task发送的tuple添加了messageId，该messageId，由rootId和tuple自身对应的messageId组成。

当每一个bolt task接收到tuple时

##MVCC多版本并发控制机制

在事务的并发读写不依赖于锁。记录的更改操作是创建一个新的副本，删除操作只是打个删除标记，因此记录会有多个版本。每个事务都关联着一个开始时间戳或者id序号。每条记录的创建、以及每次更改都有关联着一个事务序号。读取也关联这个一个事务序号。

那么当事务读取记录的时候，先确定比自身事务序号早的最近的事务序号，然后读取所有不大于该事务序号的版本。

当事务写记录的时候，则比较记录被最新访问的事务序号，如果该序号大于本序号，则abort本事务，否则创建一个新版本。

（把事务想象成一个需要0时间执行的操作，其开始时间=提交时间，故而关联其更改或者读取记录的序号就是事务序号，无需关心开始、结束时间）
