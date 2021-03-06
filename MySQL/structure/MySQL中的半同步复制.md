## MySQL中的半同步复制

来源：[https://blog.haohtml.com/archives/18328](https://blog.haohtml.com/archives/18328)

时间 2018-10-06 19:44:37

 
MySQL当前存在的三种复制模式有：异步模式、半同步模式和组复制模式。也有说是 同步复制、异步复制和半同步复制。
 
从MySQL5.5开始，MySQL以插件的形式支持半同步复制。
 
#### 异步复制（Asynchronous replication）
 
MySQL默认的复制即是异步的  ，主库在执行完客户端提交的事务后会立即将结果返给给客户端，并不关心从库是否已经接收并处理，这样就会有一个问题，主如果crash掉了，此时主上已经提交的事务可能并没有传到从上，如果此时，强行将从提升为主，可能导致新主上的数据不完整。
 
异步复制是MySQL最早的也是当前使用最多的复制模式，异步复制提供了一种简单的主-从复制方法，包含一个主库（master）和备库（一个，或者多个） 之间，主库执行并提交了事务，在这之后（因此才称之为异步），这些事务才在从库上重新执行一遍（基于statement）或者变更数据内容（基于 row），主库不检测其从库上的同步情况。在服务器负载高、服务压力大的情况下主从产生延迟一直是其诟病。工作流程简图如下：
 
![][0]
 
#### 全同步复制（Fully synchronous replication）
 
指当主库执行完一个事务， **`所有的从库`**    都执行了该事务才返回给客户端。因为需要等待所有从库执行完该事务才能返回，所以全同步复制的性能必然会收到严重的影响。
 
#### 半同步复制（Semisynchronous replication）
 
介于异步复制和全同步复制之间，主库在执行完客户端提交的事务后不是立刻返回给客户端，而是等待 **`至少一个`**    从库接收到并写到relay log中才返回给客户端。相对于异步复制，半同步复制提高了数据的安全性，同时它也造成了一定程度的延迟，这个延迟最少是一个TCP/IP往返的时间。所以，半同步复制最好在低延时的网络中使用。
 
MySQL5.5 的版本在异步同步的基础之上，以 **`插件`**  的形式实现了一个变种的同步方案，称之为半同步（semi-sync replication）。这个插件在原生的异步复制上，添加了一个同步的过程：当从库接收到了主库的变更（即事务）时，会通知主库。主库上的操作有两种： **`接收到这个通知以后才去commit事务`**  ； **`接受到之后释放session`**  。这两种方式是由主库上的具体配置决定的。当主库收不到从库的变更通知超时时，由半同步复制自动切换到异步同步  ，这样就极大了保证了数据的一致性（至少一个从库），但是在性能上有所下降，特别是在网络不稳定的情况下，半同步和同步之间来回切换，对正常的业务是有影响的。其工作流程简图如下：
 
![][1]
 
下面来看看半同步复制的原理图：
 
![][2]
 
#### 3、Group Replication（组复制）
 
不 论是异步复制还是半同步复制，都是一个主下面一个从或是多个从的模式，在高并发下高负载下，都存在延迟情况，此时如果主节点出现异常，那么就会出现数据不 一致的情况，数据可能会丢，在金融级数据库中是不能容忍的。在这种情况下，急需出现一种模式来解决这些问题。在MySQL5.7.17的版本中，带着这些 期待，新的复制模式组复制产生并GA了（本文的测试等数据均基于MySQL5.7.17）。
 
组复制的工作流程图如下： 
![][3]
 组复制的工作原理:  [http://www.sohu.com/a/124913450_354963][4]
 
半同步复制的潜在问题
 
客户端事务在存储引擎层提交后，在得到从库确认的过程中，主库宕机了，此时，可能的情况有两种：
 
#### 1. 事务还没发送到从库上
 
此时，客户端会收到事务提交失败的信息，客户端会重新提交该事务到新的主上，当宕机的主库重新启动后，以从库的身份重新加入到该主从结构中，会发现，该事务在从库中被提交了两次，一次是之前作为主的时候，一次是被新主同步过来的。
 
#### 2. 事务已经发送到从库上
 
此时，从库已经收到并应用了该事务，但是客户端仍然会收到事务提交失败的信息，重新提交该事务到新的主上。
 
无数据丢失的半同步复制
 
针对上述潜在问题，MySQL 5.7引入了一种新的半同步方案：Loss-Less半同步复制。
 
针对上面这个图，“Waiting Slave dump”被调整到“Storage Commit  ”之前。
 
当然，之前的半同步方案同样支持，MySQL 5.7.2引入了一个新的参数进行控制-rpl_semi_sync_master_wait_point
 
rpl_semi_sync_master_wait_point有两种取值
 
#### AFTER_SYNC
 
这个即新的半同步方案，Waiting Slave dump在Storage Commit之前。
 
#### AFTER_COMMIT
 
老的半同步方案，如图所示。
 
#### 三种复制方案的区别
 
  

#### 异步复制
 
MySQL复制默认是异步复制，Master将事件写入binlog，提交事务，自身并不知道slave是否接收是否处理；
 
缺点：`不能保证所有事务都被所有slave接收`。
 

#### 同步复制
 
Master提交事务，直到事务在所有slave都已提交，才会返回客户端事务执行完毕信息；
 
缺点：`完成一个事务可能造成延迟`。
 

#### 半同步复制
 
当Master上开启半同步复制功能时，至少有一个slave开启其功能。当Master向slave提交事务，且事务已写入relay-log中并刷新到磁盘上，slave才会告知Master已收到；若Master提交事务受到阻塞，出现等待超时，在一定时间内Master 没被告知已收到，此时Master自动转换为异步复制机制  ；
 
注：半同步复制功能要在Master和slave上开启才会起作用，只开启一边，依然是异步复制。


[4]: http://www.sohu.com/a/124913450_354963
[0]: ./img/6ZRJbuI.jpg
[1]: ./img/aqaA3qF.jpg
[2]: ./img/6fU3qqr.jpg
[3]: ./img/VBzMryj.jpg