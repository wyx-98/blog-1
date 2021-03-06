# 一次坑爹的MySQL问题查找

清明假日前一天，我们进行了服务器升级，然后第二天，同事给我来了一个电话，网站访问异常卡顿，在MySQL里面发现大量的slow log。

于是我立刻放弃了休假的打算，连上了服务器，首先查看slow log，发现即使是单条记录的主键查询，也耗时将近1s多，这是完全不可能的事情。然后show processlist，发现大量的语句状态为statistics，根本不能快速的执行完成。

然后查看了机器的CPU，发现MySQL的CPU很高，但IO的负载异常的低，表明MySQL真的在处理很大量的请求。我们不停地show processlist，发现有些select语句查询，row examined竟然有几十万行之多，然后看语句，发现了很蛋疼的问题。类似如下:

```
SELECT * FROM tbl WHERE group = 123 and parent = 2
```

我们使用的是邻接表模型在MySQL里面存储树状结构，也就是每条记录需要存储父节点的ID，上面那条语句就是在一个group里面查询某个节点下面的子节点。

然后这个表里面的索引竟然只有group，至于原因，貌似是最开始设计的同学认为一个group里面的数据不可能很多，即使全examined一次也无所谓。可是偏偏，一个用户蛋疼的就往一个group里面放了几十万的数据，然后，任何一次查询都会极大地拉低整个MySQL的性能。经过这一次，再一次让我意识到，组里面的小盆友的MySQL知识还需要提升，如何用好索引，写过高效的查询语句真的不是一件简单的事情。

于是我们立刻更新了索引，然后慢查询马上就没有了。但没有了超时，我们仍然发现，MySQL的CPU异常的高，show processlist的时候出现了很多system lock的情况，表明写入并发量大，一直在争锁。通过日志发现，一个用户不停地往自己的group里面增加文件，而从升级之前到第二天下午，这个用户已经累计上传了50w的文件。

最开始我们怀疑是被攻击了，但通过日志，发现这个用户的文件都是很正常的文件，并且完全像是在正常使用的。但仍不能排除嫌疑，于是我们立刻升级了一台服务器，将LVS的流量全部切过去（幸好放假了，不然一台机器铁定顶不住），但令我们吃惊的是，记录的访问请求压根没有这个用户任何的信息，但是文件仍然在不停地新增。然后通过show processlist查看，MySQL的请求全部是另外几台机器发过来的，但另外几台机器现在已经完全没有流量了。

于是我们立刻想到了异步任务，会不会有某个异步任务死循环导致不停地插入，但监控rabbitmq，却发现仍然没有该用户的任何信息。这时候，一个同事看代码，突然发现，一个能操作死循环的操作根本没扔到异步任务里面，而是直接在服务里面go了一个goroutine跑了。于是我们立刻将其他几台机器重启，然后MySQL正常了。

从晚上升级，到第二天真正发现问题并解决，因为我们的代码bug，导致了我们整个服务几乎不可用，不可不说是一个严重的教训。

+ MySQL索引设计不合理，导致极端情况下面拖垮了整个性能。
+ 代码健壮性不足，对于可能引起死循环的应用没有做更多的检查处理。
+ 日志缺失，很多偷懒不写日志，觉得没啥用，但偏偏遇到问题了才会有用。
+ 统计缺失，没有详细的统计信息，用来监控整个服务。
+ 没有更健壮的限频限容策略，虽然我们开启了，但因为程序内部bug，导致完全没用。

有些时候，只有做好了完全的准备，才能更好的应对对突发情况，毕竟我们是在做产品，要对用户负责，也要对自己的职业负责。

