## spark调优 ##
    1、参数调优：  
        Executor core，Executor memory
    2、高性能算子调优：  
        a、mapPartition  
        b、foreachPartition
        c、filter后使用coalsesce减少分区数量  
        d、persist / cache 对数据持久化  
        e、reduceByKey和aggrateByKey取代groupByKey  
        f、repartition解决sparksql低并行度的性能问题
    3、开发调优：
        a、广播变量(broadcast) 结合业务说明  
        b、尽量避免复用同一个RDD，如果需要，则进行cache持久化  
        c、尽量避免shuffle类算子(reduceByKey, join, distinct, repartition)  
        d、使用kryo优化序列化性能

