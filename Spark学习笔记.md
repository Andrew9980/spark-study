## 一、基本概念

### 专业名词：

RDD（Resillient Distributed Dataset）：弹性分布式数据集，高度受限（只读）的共享模型，spark中数据抽象模型。

DAG（Directed Acyclic Graph）：有向无环图，反应RDD之间的依赖关系，spark对不同rdd之间的操作流程就会反应在DAG中。

Executor：运行在工作节点的进程。运行资源调度管理分配的很多任务。

Application： Spark应用程序。

任务（task）：运行在Executor上的工作单元。

作业（Job）：Spark应用程序会分成很多的作业，一个作业会有切分为很多个阶段，每个阶段就会有不同的任务。

阶段（Stage）：作业的基本调度单位。每个作业按照算法切分成若干个阶段，然后每个阶段包含有很多的任务，然后把任务分发的不同的机器上执行。

### Spark运行架构图：

![image-20200401122326512](https://github.com/Andrew9980/spark-study/blob/master/image/spark%E8%BF%90%E8%A1%8C%E6%B5%81%E7%A8%8B%E5%9B%BE.png)





### Spark执行流程

![image-20200401155400712](/Users/andrew/Library/Application Support/typora-user-images/image-20200401155400712.png)

## 二、RDD设计

### hadoop Map-Reduce缺陷

Map后将数据存入磁盘中，Reduce时需要从磁盘中读取数据，增加了磁盘IO和数据序列化和反序列化时间

### Spark方案

将数据抽象成RDD，存入内存中，避免磁盘IO消耗

### RDD概念

Spark从硬盘中读取到数据后，放入rdd中，因为数据量很大，所以会将数据分区到不同的机器上，在物理上是分布式对象集合，在抽象逻辑上是一个整体，分布式并行处理，高效计算。

RDD是一个只读的数据集合，如果要实现修改，则需要按照操作生成新的RDD。

RDD操作类型：

- Action（动作类型操作）：根据操作轨迹发生计算
- Transformation（转换类型操作）：只记录转化的轨迹，不会真正的发生计算，只能粗力度转化，即对整个rdd进行操作。不支持细粒度修改

![image-20200402180514655](/Users/andrew/Library/Application Support/typora-user-images/image-20200402180514655.png)

**RDD的依赖关系**

窄依赖：不划分阶段，一个分区对应一个分区或多个父分区对应一个子分区

![image-20200403123354738](/Users/andrew/Library/Application Support/typora-user-images/image-20200403123354738.png)

宽依赖：划分成多个阶段，一个父亲分区对应多个子分区

![image-20200403144028345](/Users/andrew/Library/Application Support/typora-user-images/image-20200403144028345.png)



**Shuffle操作**

分区的数据在网络当中大规模地来回传输以及不同节点之间互相传递数据

词频统计例子：

![image-20200403122418236](/Users/andrew/Library/Application Support/typora-user-images/image-20200403122418236.png)

首先读取文件，将数据分成3个分区，分发给三台Map机器，Map机器将数据整理后，发送给三台Reduce机器，因为每一台Map机器都存在(a,1),(b,1),(c,1)，所以需要在每台机器上都要发送一遍，涉及到大量的网络和磁盘消耗。将数据洗牌的这种操作成为**Shuffle**

**Spark对RDD的优化**

Spark对窄依赖的优化，在rdd的transform操作时，rdd对数据分区进行fork分发到不同的机器上，当每个机器执行完后再join结果输出。当存在多个transform操作时，分区不会再等待其他分区的操作结束，而且继续执行下个transform操作，节省等待时间。窄依赖不会交换其他分区的数据，而宽依赖则会等待其他分区的数据，这时数据就会shuffle。

![image-20200403152927827](/Users/andrew/Library/Application Support/typora-user-images/image-20200403152927827.png)

Spark只会对窄依赖进行优化，窄依赖不会等待数据，宽依赖则需要等待其他分区的数据交换。宽依赖交换数据时会发生shuffle，需要等待到多个分区之间的数据。只要发生shuffle一定会写磁盘。所以窄依赖可以**流水线**优化，宽依赖不能流水线优化。

Spark对DAG图进行反向解析，当发现窄依赖时，不断的把操作加入到阶段中去，分区之间不会等待，当发生宽依赖时，会分成不同的阶段，一个阶段执行完才能执行下一个阶段的任务。

![image-20200403154731090](/Users/andrew/Library/Application Support/typora-user-images/image-20200403154731090.png)

A->B宽依赖，分成两个Stage，CDEF窄依赖则分在一个Stage中







```
spark.read.jdbc("jdbc:mysql://drdshbga020328s4public.drds.aliyuncs.com:3306/timing_user", "t_im_netease", TimingUtil.getJdbcSpan("imID", 1589978733, 1589977851, 50000), RecommendUtil.odsTimingUserProp).createOrReplaceTempView("t_im_netease")
```

``` 
spark.read.jdbc(RecommendUtil.confConnStr, sql, RecommendUtil.cfgProp).where(s"table_name = 't_im_netease'").collect().foreach(line => {
                val target_table_name = line.getAs("target_table_name").toString
                var sql_expr = line.getAs("sql_expr").toString
                println(sql_expr)
                val df = spark.sql(sql_expr)
                kuduContext.upsertRows(df, "impala::timing_ods." + "t_im_netease")
            })
```

```
 val maxLine = spark.read.options(Map("kudu.master" -> RecommendUtil.kuduMaster, "kudu.table" -> s"impala::timing_ods.t_all_timing_record")).
            format("org.apache.kudu.spark.kudu").load.select("id").orderBy(col("id").desc).head(1)(0).get(0).toString.toLong
```

``` 
val max: Int = spark.read.jdbc(RecommendUtil.odsTimingServiceConnStr, s"(select max(id) from t_all_timing_record) as tmp", RecommendUtil.odsTimingServiceProp).head().get(0).toString.toInt
```

``` 
spark-shell --master yarn --name user-event --deploy-mode client --driver-memory 4G  --executor-memory 4G --executor-cores 4 --num-executors 4  --files /etc/ecm/spark-conf/hive-site.xml --jars hdfs:///user/hadoop/timing/jars/config-1.3.4.jar,hdfs:///user/hadoop/timing/jars/fastjson-1.2.58.jar,hdfs:///user/hadoop/timing/jars/druid-1.1.16.jar,hdfs:///user/hadoop/timing/jars/mysql-connector-java-8.0.16.jar,hdfs:///user/hadoop/timing/jars/kudu-spark2_2.11-1.9.0.jar,hdfs:///user/hadoop/timing/jars/process-core-1.0-SNAPSHOT.jar,hdfs:///user/hadoop/timing/jars/process-report-1.1.jar
```

``` 
spark.read.jdbc("jdbc:mysql://rr-bp12u85w22spt5976do.mysql.rds.aliyuncs.com:3306/timing", "", "id", 1589978733, 70000000, 200, RecommendUtil.odsProp).     write.mode(SaveMode.Overwrite).parquet(s"timing/t_learning_group_im")             spark.read.parquet(s"timing/${elem._2}").createOrReplaceTempView(elem._2)
```

```
val df = spark.read.options(Map("kudu.master" -> "192.168.2.131:7051,192.168.2.132:7051,192.168.2.133:7051", "kudu.table" -> s"impala::timing_ods.t_learning_group_im_user")).             format("org.apache.kudu.spark.kudu").load.select("id").orderBy(col("id").desc)
var maxLine = if (df.head(1).isEmpty) 1 else df.head(1)(0).get(0).toString.toLong
```

``` 
val max: Int = spark.read.jdbc(RecommendUtil.odsConnStr, s"(select max(id) from t_learning_group_im) as tmp", RecommendUtil.odsProp).head().get(0).toString.toInt
```

