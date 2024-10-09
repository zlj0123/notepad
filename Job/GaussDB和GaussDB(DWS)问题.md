
## 基本信息

GaussDB数据库和GaussDB(DWS)数据仓库是两个不同的产品，他们官网里使用的JDBC也不一样，GaussDB是用com.huawei.gaussdb.jdbc.Driver，GaussDB(DWS)是用com.huawei.gauss200.jdbc.Driver，url前缀都是jdbc:gaussdb://开头，上面是基本信息。

GaussDB JDBC驱动：
![[AE9C5BBB-1CDD-4056-ACE2-6B44CDF4A56E 1.png]]

GaussDB(DWS) JDBC驱动：
![[0FF1D777-5D61-49a8-8366-0F3BD5C9EDC0.png]]


## 我们组件现状
### 数据源：
1. 数据源以前支持GaussDB，但底层使用的是com.huawei.gauss200.jdbc.Driver的驱动，这个应该不对，但目前也没发现问题，建议这个版本不动，需不需要升级和改动，需架构评估。 
2. 数据源部署库支持GaussDB，上个版本使用了com.huawei.opengauss.jdbc.Driver驱动，这个也不对，这个月版本会改回来，这个已经在开发了。
3. 这个月需求里，数据源需要支持GaussDB(DWS)，部署库也需要支持GaussDB(DWS)，会用GaussDB(DWS)的驱动。

### 离线同步
1. 离线同步很早的时候，2021年2月份支持了GaussDB，但里面驱动使用的是com.huawei.gauss200.jdbc.Driver的驱动，这个不对，但这个同步前几年基本没人使用。
2. 上个月离线同步支持了GaussDB，使用的是com.huawei.opengauss.jdbc.Driver（这个也是客户提供的驱动）。
3. 这个月客户又提了需求支持GaussDB，但使用com.huawei.gaussdb.jdbc.Driver，这个月发版。
4. 重点：这个月还有支持GaussDB(DWS)读写的同步需求，按照官网，使用最新DWS驱动包，使用com.huawei.gauss200.jdbc.Driver的驱动，这个就有冲突了，因为在DataGo里，这个同名驱动类已经被占用了。
5. 针对上面4，我们的建议是，GaussDB就用GaussDB的使用方式；GaussDB(DWS)就用GaussDB(DWS)的使用方式，也就是新版里不再支持用com.huawei.gauss200.jdbc.Driver来同步GaussDB，需要改用com.huawei.gaussdb.jdbc.Driver；然后这个com.huawei.gauss200.jdbc.Driver升级驱动后给GaussDB(DWS)使用，需要有个发版声明。

总结：从这个版本开始，我们的使用方式改变，如果你以前用到了gaussdb，升级时需要修改任务：
1. 下降使用com.huawei.gauss200.jdbc.Driver来同步GaussDB数据库，改为com.huawei.gaussdb.jdbc.Driver来同步GaussDB数据库。
2. 用com.huawei.gauss200.jdbc.Driver来同步GaussDB(DWS)数据库。

| 数据库  | 以前版本 | 现在版本 |
| ------- | ---------- | ---------- |
| GuassDB | com.huawei.gauss200.jdbc.Driver | com.huawei.gaussdb.jdbc.Driver |
| GuassDB(DWS)|   不支持  |  com.huawei.gauss200.jdbc.Driver  |


### 最终结论
近跟产品(陈海云)沟通，需要统一处理，最终的处理方案如下：

common-driver包里：
GaussDB(旧)：lib/gauss/gsjdbc200-1.0.0.jar  (这个路径保持不动，但是不建议再引用，以后会移除)
GaussDB(新)：lib/gauss8/gaussdbjdbc-5.0.0-htrunk4.csi.gaussdb_kernel.opengaussjdbc.r2.jar  (新的GaussDB驱动，以后都改为引用这个)
GaussDB(DWS)：lib/gaussdws/huaweicloud-dws-jdbc-8.3.1.200-200.jar  (GaussDB(DWS)的驱动)

| 组件     | 数据库       | 以前版本                        | 现在版本                        |
| -------- | ------------ | ------------------------------- | ------------------------------- |
| 数据源   | GuassDB      | GaussDB(旧)里的com.huawei.gauss200.jdbc.Driver | GaussDB(新)里的com.huawei.gaussdb.jdbc.Driver  |
|          | GuassDB(DWS) | 不支持                          | GaussDB(DWS)里的com.huawei.gauss200.jdbc.Driver |
| 离线同步 | GuassDB      | GaussDB(旧)里的com.huawei.gauss200.jdbc.Driver | GaussDB(新)里的com.huawei.gaussdb.jdbc.Driver  |
|          |  GuassDB(DWS) | 不支持                          | GaussDB(DWS)里的com.huawei.gauss200.jdbc.Driver |

然后发版时，附一个发版声明，有用到GuassDB的，驱动需要更改。







