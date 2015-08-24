---
layout: post
title: redis概要
author: 灵元
---
 



###常用指令

**集合**

1. scard(key) 集合的基数，即大小
2. sadd(key,member) 添加成员
3. srem(key,member) 删除成员
4. smembers(key) 获取所有成员
5. sismember(key,member) 判断是否是成员


**pipeline**
本指令可以用来执行乐观事务，或者批量执行指令以便提高性能
当redis.pipelined()表明即将开启一个事务。事务中的多个命令处于MULTI/EXEC命令对之间。在执行MULTI之前，执行WATCH命令，可以监听在事务期间被WATCH的数据是否发生了变更，若发生了变更就跑出异常，终止事务。用户可以根据需要再次重启事务或者终止尝试。在执行完毕时需要关闭pipe.
        
        pipe=redis.pipelined();
        pipe.watch("data1","data2")
        pipe.multi()
        ......
        pipe.execute()
        pipe.close()

若redis.pipeline(false)则不开启事务，纯粹将各个命令打包在前一起提高性能，不需要执行pipe.multi()方法。

###主从复制
###集群