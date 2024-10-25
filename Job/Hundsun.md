## 1 DataGo
### 1.1 离线同步所需端口

Hive MetaStore：9083
HiveServer2:   10000
HDFS NameNode （fs.defaultFS）: 8020 
HDFS NameNode （dfs.namenode.http-address）： 50070
HDFS DataNode（dfs.datanode.address）：50010
HBase 写入，需开通端口：Zookeeper：2181
HBase RegionServer： 60020
HBase Master： 60000

### 1.2 DataGo升级

DataGo升级注意点：
1. BDATA1.0-DataGoDeploy-V202101.00.006(包含) ~ BDATA1.0-DataGoDeploy-V202101.00.016(包含)，如果是querySql语句，我们会自动在外面罩一层，实际执行变成select * from (你们自己的sql语句) t，但这样改，如果你们自己的sql语句涉及大表查询，就会很慢，所以BDATA1.0-DataGoDeploy-V202101.00.017版本里，我们把罩一层去掉了，这样如果你们的sql里有union all、left join等情况，需要你们自己在sql语句外面罩一层，写成select xx1,xx2,xx3 (你们原先的sql) t（Log里有SqlParser错误）。
2. ART1.0-DataGoDeploy-V202101-11-002.zip版本里把mysql驱动升级到到了mysql8，执行的过程中有可能会报SQLException: HOUR_OF_DAY: 2 -> 3问题，需要你们在同步的url里面加上&serverTimezone=Asia/Shanghai配置。

### 1.3 GaussDB驱动
```
GaussDB 旧版本:
jdbc 驱动名：com.huawei.opengauss.jdbc.Driver
JDBC URL: jdbc:opengauss：//
对应底层的datax读写插件为gaussdb8xreader和gaussdb8xwriter
MAVEN 坐标:
<dependency> 
	<groupId>com.huawei.opengauss.jdbc</groupId>
	<artifactId>gaussdbDriver</artifactId>
	<version>5.0.0-htrunk3.-uf30</version>
</dependency>
  
GaussDB 新版本：
jdbc 驱动名：com.huawei.gaussdb.jdbc.Driver
JDBC URL: jdbc:gaussdb://
对应底层的datax读写插件为gaussdb5xxreader和gaussdb5xxwriter
MAVEN 坐标：
<dependency>  
	<groupId>com.huawei.gaussdb</groupId>  
	<artifactId>gaussdbjdbc</artifactId>  
	<version>5.0.0-htrunk4.csi.gaussdb_kernel.opengaussjdbc.r2_20240920_gaussdbjdbc</version>  
</dependency>

GaussDBDWS：
jdbc 驱动名：com.huawei.gauss200.jdbc.Driver
JDBC URL: jdbc:gaussdb://
对应底层的datax读写插件为gaussdbdwsreader和gaussdbdwswriter
MAVEN 坐标：
<dependency>  
	<groupId>com.huaweicloud.dws</groupId>  
	<artifactId>huaweicloud-dws-jdbc</artifactId>  
	<version>8.3.1-200</version>  
</dependency>

openGauss:
jdbc 驱动名：org.opengauss.Driver
JDBC URL: jdbc:opengauss://
maven 坐标：
对应底层的datax读写插件为opengaussreader和opengausswriter
<dependency>
	<groupId>org.opengauss</groupId>
	<artifactId>opengauss-jdbc</artifactId>
	<version>5.1.0</version>
</dependency>
```

目前DataGo对GaussDB的支持如下：
1. 驱动com.huawei.gauss200.jdbc.Driver 搭配 jdbc:gaussdb://ip:port/dbname的url (这个是很早的GaussDB版本，目前不用了，目前这个给GaussDB(DWS)使用了)
2. 驱动com.huawei.opengauss.jdbc.Driver 搭配 jdbc:opengauss://ip:port/dbname的url (已经支持，但官网好像没有推荐用这个，这个底层有mergeinto代码，测试发版)
3. 驱动com.huawei.gaussdb.jdbc.Driver 搭配 jdbc:gaussdb://ip:port/dbname的url (使用最新的GaussDB驱动，官网推荐这个，这个底层也有mergeinto代码，但是没有测试和发版)

DataGoApi层会把type为gauss的映射到上面3，type为gaussdws的映射为上面的1。2只有组件层面支持。


### 1.4 测试环境
10.20.194.39:8000/uf30
hs_fil/UF30.hundsun


### 1.5 mogdb
className为io.mogdb.Driver，URL为 jdbc:mogdb://192.168.86.218:26000/
mogdb/mogdb@123
<dependency>
    <groupId>io.mogdb</groupId>
    <artifactId>mogdb-jdbc</artifactId>
    <version>5.0.0.8.mg</version>
</dependency>

## 2 DataSource

### 2.1 InfluxDB
离线同步的就两个吧。
1. timeout，这个已经发了联调包，经过测试，也是可行的 
2. influxdb的字段，主要是filed字段，influx返回是integer类型，但里面实际存的是double类型，导致类型转换失败，这个需要架构师决定一下方案。这里面的原因就是，数据源的influxdb连接器，用show field keys from measurement命令返回field的name和type，这个type，influx返回的是integer，但里面存储的实际值却是double，比如1.0和12.0这些值。离线同步这边，会把这个值转换到相应的type，转换出错。这里有几个处理方案，1. 通过改动数据源来修复，对返回的field的type，跟tag一样处理，都是返回String。 2. 由离线同步修复，如果type是integer，那么当成string来处理 3. 由大数据开发平台修复，就是那个字段类型可以选择和编辑，由用户选择类型

## 3 问题
1. 在你们的url后面加上&autoReconnect=true&trackSessionState=true后，执行成功，你们的问题跟这个一模一样，包括堆栈、驱动也一样，用mysql去连mariadb，https://bugs.mysql.com/bug.php?id=105706，但是这里没给出解决办法
Caused by: java.lang.ArrayIndexOutOfBoundsException: 0
at com.mysql.cj.protocol.a.NativePacketPayload.readInteger(NativePacketPayload.java:386)

2. 