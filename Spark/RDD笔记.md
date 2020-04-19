## 一、RDD创建

**从本地文件读取**

``` 
val lines = sc.textFile("file:///文件路径")
```

**从分布式文件系统读取**

``` 
val lines = sc.textFile("hdfs://localhost:[9000 hadoop配置的端口号]/文件路径")
```

**持久化**

persist() - 将一个rdd标记为持久化, 当遇到第一个动作类型的操作时才会真正持久化。

- MEMORY_ONLY:存入内存，内存不足，替换内容 = rdd.cache()
- MEMORY_AND_DISK:内存不足存入磁盘
- DISK_ONLY:存入磁盘

**RDD 分区的作用**

- 增加并行度：RDD分区被保存到不同的节点上
- 减少通信开销

**RDD分区原则**

分区数=集群CPU核心数