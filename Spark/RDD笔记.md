## 一、RDD

#### 一、简介

RDD（resilient distributed dataset,RDD）：弹性分布式数据集，spark数据逻辑的实体，在集群中的多台机器上进行了数据分区，通过对多台机器上不同RDD分区的控制，就能减少机器之间的数据重拍（data shuffling）

#### 二、创建方式

从本地文件读取**

``` 
val lines = sc.textFile("file:///文件路径")
```

**从分布式文件系统读取**

``` 
val lines = sc.textFile("hdfs://localhost:[9000 hadoop配置的端口号]/文件路径")
```

**从父RDD转换得到新的RDD**

```
val lines = rdd.transform()
```

#### 四、操作方式

- Transformation: 操作是延迟计算，rdd的转换操作不会立即执行，当遇到Actions操作时，才会真正触发
- Action:触发Spark提交作业（job），并讲数据输出到Spark系统

**Value型Transformation算子**：

1. 输入分区与输出分区一对一型

   (1)map:将原来RDD的每个数据项通过map中的用户自定义函数f映射转变为一个新的RDD。新RDD叫作`MappedRDD(this,sc.clean(f))`

   (2)flatMap:将原来RDD中的每个元素通过函数f转换为新的元素，并将生成的RDD的每个集合中的元素合并为一个集合。内部创建`FlatMappedRDD(this,sc.clean)`

   (3)mapPartitions:获取到每个分区的迭代器，在函数中通过这个分区整体的迭代器对整个分区的元素进行操作。生成MapPartitionsRDD

   (4)glom:将每个分区形成一个数组，内部返回GlommedRDD

2. 输入分区与输出分区多对一型

   (1)union:返回的rdd数据类型和被合并的rdd数据元素类型相同， 并不进行去重操作，保存所有元素。如果想去重，可以使用distinct()。++符号相当于union操作。使用函数时，需要保证两个rdd元素的数据类型相同

   (2)cartesian:对两个rdd内的所有元素进行笛卡尔积操作，操作后，内部实现返回CartesianRdd。

3. 输入分区与输出分期多对多型

   (1)groupBy:讲元素通过函数生成相应的Key，数据就转化为Key-Value格式，之后将Key相同的元素分为一组

4. 输出分区为输入分区子集型

   (1)filter:对元素进行过滤，对每个元素应用f函数，返回值为true的元素在rdd中保留，返回为false的将过滤掉，内部实现为FilterRDD

   (2)distinct:将rdd中的元素进行去重操作

   (3)subtract:进行集合的差集操作，RDD1去除RDD1和RDD2交集中的所有元素

   (4)sample:将rdd集合内的元素进行采样，获取所有元素的子集。可以设定是否放回的抽样，百分比，随机种子

   (5)takeSample:不使用相对比例采样，而是按照指定的采样个数进行采样，同事返回结果不是rdd，而是相当于对采样后的数据进行collect(),返回结果的集合为单机的数组

5. 输入与输出分区一对一的算子：Cache算子对RDD分区进行缓存

   (1)cache将rdd元素从磁盘缓存到内存，相当于persist(MEMORY_ONLY)函数的功能

   (2)persist:将一个rdd标记为持久化, 当遇到第一个动作类型的操作时才会真正持久化。

   - MEMORY_ONLY:存入内存，内存不足，替换内容 = rdd.cache()
   - MEMORY_AND_DISK:内存不足存入磁盘
   - DISK_ONLY:存入磁盘

**Key-Value型Transformation算子**

1. 输入分区与输出分区一对一

   (1)mapValues:针对(key,value)型数据中的Value进行Map操作，而不对key进行处理

2. 对单个rdd或两个rdd聚集

   (1)单个rdd聚集

   ​	1)combineByKey

   ​	2)reduceByKey:根据相同key的value的两个值合并成一个值

   ​	3)partitionBy:对rdd进行分区操作，如果原有rdd的分区器和现有分区器(partitioner)一致，则不重分区，如果不一致，则相当于根据分区器生成一个新的ShuffedRDD

   (2)对两个rdd进行聚集

   ​	cogroup: 将两个rdd进行协同划分，对两个rdd中的key-value类型的元素，每个rdd相同key的元素分别聚合为一个集合，并且返回两个rdd中对应key的元素集合的迭代器

3. 连接

   1. join:将两个需要连接的rdd进行cogroup函数操作之后形成新的rdd，对每个key下的元素进行笛卡尔积操作，返回的结果再展平，对应key下的所有元组形成一个集合，最后返回`rdd[k,(v,w)]`
   2. leftOutJoin和rightOutJoin:在join的基础上先判断一侧的rdd元素是否为空，如果为空则填充为空，不为空，则将数据进行连接运算，返回结果

**Actions算子**

Action算子中通过SparkContext执行提交作业的runJob操作，触发了RDD DAG的执行。

1. 无输出：

   foreach:对rdd中的每个元素都应用f函数操作，不返回rdd和array

2. hdfs：

   (1)saveAsTextFile:将数据存储到hdfs指定目录，将rdd中的每个元素转变为(Null,x.toString)，然后写入hdfs

   (2)saveAsObjectFile:将分区中的每10个元素组成一个array，然后将这个array序列化，映射为(Null, BytesWritable(Y))的元素，写入hdfs为SequenceFile的格式

3. Scala集合和数据类型

   1. collect：相当于toArray,collect将分布式的rdd返回为一个单机的scala Array数组。
   2. collectAsMap：对(k,v)型的rdd数据返回一个单机HashMap。对于重复的k的rdd元素，后面的元素会覆盖前面的元素
   3. reduceByKeyLocally:先reduce再collectAsMap的功能
   4. lookup:对(key,value)型的rdd操作。返回指定key对应的元素形成的seq。这个函数处理的优化在于，如果这个rdd包含分区器，则只会对应处理k所在的分区。然后返回由(k,v)形成的seq，如果rdd不包含分区器，则需要对全rdd元素进行暴力扫描处理，搜索指定k对应的元素。
   5. count:返回整个rdd的元素个数
   6. top:返回最大的k个元素
   7. take:返回最小的k个元素
   8. take:Ordered:返回最小的k个元素，并且返回的数组中保持元素顺序
   9. first:相当于top(1)返回整个rdd中的前k个元素，可以定义排序方式Ordering[T]。返回的是一个含前k个元素的数组。

































