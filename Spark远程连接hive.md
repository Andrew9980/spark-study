## Spark远程连接hive

### 一、JDBC连接

``` scala
val spark = SparkSession.builder().master("local[*]").enableHiveSupport().getOrCreate()
        val properties = new Properties()
        properties.put("user","root")
        properties.put("password","EMRroot1234")
        properties.put("driver","org.apache.hive.jdbc.HiveDriver")
        spark.read.jdbc("jdbc:hive2://192.168.2.131:10000/timing","t_feed_type",properties).toDF()
```



### 二、hive-site 官方推荐

``` xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>hive.metastore.local</name>
        <value>false</value>
    </property>
    <property>
        <name>hive.metastore.uris</name>
        <value>thrift://192.168.2.131:9083</value>
    </property>
    <property>
        <name>hive.server2.thrift.port</name>
        <value>10000</value>
    </property>
    <!-- <property>
        <name>fs.defaultFS</name>
        <value>hdfs://hive服务器IP:8020</value>
    </property> -->
    <property>
        <name>hive.metastore.warehouse.dir</name>
        <value>/usr/hive/warehouse</value>
    </property>
    <property>
        <name>beeline.hs2.connection.user</name>
        <value>root</value> 
    </property>
    <property>
        <name>beeline.hs2.connection.password</name>
        <value>EMRroot1234</value>
    </property>
  </configuration>
```

注意：阿里云的机器会有域名访问不到，可以将域名对应的ip加入到本机的hosts文件中