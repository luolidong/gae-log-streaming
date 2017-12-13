# log-streaming

使用spark-streaming实时join拼接检索日志和曝光日志，并生成扣费消费投递到kafka中。spark版本为1.6.3。



## JOIN过程

此spark作业会创建两条kafka数据流，分别为检索日志和曝光日志流。其中检索日志使用滑动窗口，批次间隔为10s, 窗口长度为20s, 步长为10s，即每个窗口总是会有两个批次的数据。曝光日志没有窗口，批次间隔为10s。

![flow](http://ovbyjzegm.bkt.clouddn.com/spark-log.png)

因此，当执行join计算时，T2批次的曝光日志总是会与T2、T1批次的检索日志进行join, 即总是与前两个批次进行计算。T3批次的曝光日志会与T3, T2批次的检索进行join。为了防止重复曝光，每个批次的曝光日志先会进行批次内去重，然后再减去上一个批次内已经存在过的日志。

检索日志和曝光日志通过`sid`字段进行join，`sid`由GAE生成并添加到曝光监控地址的参数中。

使用这种计算方式在业务上的表现为: 广告检索之后必须在20s内曝光，否则曝光会被丢弃；如果有多次曝光，则最早的曝光为有效曝光，后续曝光丢弃。