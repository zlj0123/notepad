## DataGo
### 离线同步所需端口

Hive MetaStore：9083
HiveServer2:   10000
HDFS NameNode （fs.defaultFS）: 8020 
HDFS NameNode （dfs.namenode.http-address）： 50070
HDFS DataNode（dfs.datanode.address）：50010
HBase 写入，需开通端口：Zookeeper：2181
HBase RegionServer： 60020
HBase Master： 60000

### DataGo升级

DataGo升级注意点：
1. BDATA1.0-DataGoDeploy-V202101.00.006(包含) ~ BDATA1.0-DataGoDeploy-V202101.00.016(包含)，如果是querySql语句，我们会自动在外面罩一层，实际执行变成select * from (你们自己的sql语句) t，但这样改，如果你们自己的sql语句涉及大表查询，就会很慢，所以BDATA1.0-DataGoDeploy-V202101.00.017版本里，我们把罩一层去掉了，这样如果你们的sql里有union all、left join等情况，需要你们自己在sql语句外面罩一层，写成select xx1,xx2,xx3 (你们原先的sql) t（Log里有SqlParser错误）。
2. ART1.0-DataGoDeploy-V202101-11-002.zip版本里把mysql驱动升级到到了mysql8，执行的过程中有可能会报SQLException: HOUR_OF_DAY: 2 -> 3问题，需要你们在同步的url里面加上&serverTimezone=Asia/Shanghai配置。

## DataSource