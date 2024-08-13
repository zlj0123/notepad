## Dataflow Programming Model

### Levels of Abstraction

![53c0a491b159b238feb5c13681a25bb5.png](en-resource://database/532:1)

  

### Programs and Dataflows

![01affc71fdd77760ea4e627d03a0b34b.png](en-resource://database/521:1)

  

### Parallel Dataflows

![c89665665c9a8c2018f4071842e24980.png](en-resource://database/539:1)

  

### Time

![11d17312cae5d8a428a9b87ab8ae62c0.png](en-resource://database/524:1)

  

### Statefull Operations

![bbf0232610f5a658418d9de5e60e42ee.png](en-resource://database/538:1)

  

### Tasks and Operator Chains

  

For distributed execution, Flink chains operator subtasks together into tasks. Each task is executed by one thread.

  

![3c813baeeaf93d4eb8c3d2f03a2eff86.png](en-resource://database/529:1)

  

### Job Managers, Task Managers, Clients

![e2780c828f3fc68f429b967900ff487b.png](en-resource://database/541:1)

  

* Program Code：我们编写的 Flink 应用程序代码

* Job Client：Job Client 不是 Flink 程序执行的内部部分，但它是任务执行的起点。 Job Client 负责接受用户的程序代码，然后创建数据流，将数据流提交给 Job Manager 以便进一步执行。 执行完成后，Job Client 将结果返回给用户

* Job Manager：主进程（也称为作业管理器）协调和管理程序的执行。 它的主要职责包括安排任务，管理 checkpoint ，故障恢复等。机器集群中至少要有一个 master，master 负责调度 task，协调 checkpoints 和容灾，高可用设置的话可以有多个 master，但要保证一个是 leader, 其他是 standby; Job Manager 包含3个重要组件，分别是Dispatcher、ResourceManager、JobMaster；通常情况下，一个Job Manager包含一个Dispatcher、一个ResourceManager和多个JobMaster。

  

  

* Task Manager：从 Job Manager 处接收需要部署的 Task。Task Manager 是在 JVM 中的一个或多个线程中执行任务的工作节点。 任务执行的并行性由每个 Task Manager 上可用的任务槽（Slot 个数）决定。 每个任务代表分配给任务槽的一组资源。 例如，如果 Task Manager 有四个插槽，那么它将为每个插槽分配 25％ 的内存。 可以在任务槽中运行一个或多个线程。 同一插槽中的线程共享相同的 JVM。

  

![04f1c073a80bdfc126b7a51d1796e20b.png](en-resource://database/676:1)

  

  

### Task Slots and Resources

  

一个TaskManager有一个slot意味着每一个任务运行在一个独立的JVM进程中。有多个slot意味着多个任务共享一个JVM进程，共享JVM进程的任务之间共享TCP连接和心跳信息，同时共享数据集和数据结构，从而节省了每个任务的开销。

  

Having one slot per TaskManager means each task group runs in a separate JVM (which can be started in a separate container, for example). Having multiple slots means more subtasks share the same JVM. Tasks in the same JVM share TCP connections (via multiplexing) and heartbeat messages. They may also share data sets and data structures, thus reducing the per-task overhead.

![379dc88b724a206ba6ce0e176e93a780.png](en-resource://database/528:1)

  

By default, Flink allows subtasks to share slots even if they are subtasks of different tasks, so long as they are from the same job. The result is that one slot may hold an entire pipeline of the job. Allowing this slot sharing has two main benefits:

  

* A Flink cluster needs exactly as many task slots as the highest parallelism used in the job. No need to calculate how many tasks (with varying parallelism) a program contains in total.

  

* It is easier to get better resource utilization. Without slot sharing, the non-intensive source/map() subtasks would block as many resources as the resource intensive window subtasks. With slot sharing, increasing the base parallelism in our example from two to six yields full utilization of the slotted resources, while making sure that the heavy subtasks are fairly distributed among the TaskManagers.

  

![00a6d568113c2bc265b38d9290c71595.png](en-resource://database/520:1)

  

### Datastream

  

![9d29bd86a688127a509936452418c148.png](en-resource://database/536:2)

  

### Function

  

窗口应用函数，我们有三种最基本的操作窗口内的事件的选项:

* 像批量处理，ProcessWindowFunction 会缓存 Iterable 和窗口内容，供接下来全量计算；

* 或者像流处理，每一次有事件被分配到窗口时，都会调用 ReduceFunction 或者 AggregateFunction 来增量计算；

* 或者结合两者，通过 ReduceFunction 或者 AggregateFunction 预聚合的增量计算结果，在触发窗口时， 提供给 ProcessWindowFunction 做全量计算。

  

### JobManager数据结构

![3a6c355198efa4c7fe55cea613e681c2.png](en-resource://database/696:1)

![50ebaf6fd24eeebdaa2b43d2d1a9772c.png](en-resource://database/698:1)

  

  

  

## 源码研读

  

mvn clean install -Dmaven.test.skip=true -Pscala-2.12 -Dscala-2.12 -Dmaven.javadoc.skip=true -Dcheckstyle.skip=true

  

```java

SocketWindowWordCount.java

  

// get the execution environment

final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

  

// get input data by connecting to the socket

DataStream<String> text = env.socketTextStream(hostname, port, "\n");

  

// parse the data, group it, window it, and aggregate the counts

DataStream<WordWithCount> windowCounts = text

  

.flatMap(new FlatMapFunction<String, WordWithCount>() {

@Override

public void flatMap(String value, Collector<WordWithCount> out) {

for (String word : value.split("\\s")) {

out.collect(new WordWithCount(word, 1L));

}

}

})

  

.keyBy("word")

.timeWindow(Time.seconds(5))

  

.reduce(new ReduceFunction<WordWithCount>() {

@Override

public WordWithCount reduce(WordWithCount a, WordWithCount b) {

return new WordWithCount(a.word, a.count + b.count);

}

});

  

// print the results with a single thread, rather than in parallel

windowCounts.print().setParallelism(1);

  

env.execute("Socket Window WordCount");

```

SocketTextStreamFunction:

![0d96692fccabd615d092b8bac49598ff.png](en-resource://database/523:1)

  

StreamSource:

![37564da81aa807ae527889814d83885b.png](en-resource://database/527:1)

  

DataStream -> DataStreamSource:

![44640f4615c442141a15c2f10de76132.png](en-resource://database/531:1)

  

  

### run

```shell

./bin/flink run -c org.apache.flink.streaming.scala.examples.socket.SocketWindowWordCount app/flink-streaming-1.0.0-all.jar --port 9000

```

  

### Blog

http://wuchong.me

http://www.54tianzhisheng.cn/

  

### Checkpoint

![cc0fc8da13b58487b81d7ac56592fd61.png](en-resource://database/540:1)

  

https://www.jianshu.com/p/8d6569361999

  

CheckPoint执行过程

1. JobManager端的 CheckPointCoordinator向 所有SourceTask发送CheckPointTrigger，Source Task会在数据流中安插CheckPoint barrier

2. 当task收到所有的barrier后，向自己的下游继续传递barrier，然后自身执行快照，并将自己的状态异步写入到持久化存储中

* 增量CheckPoint只是把最新的一部分更新写入到 外部存储

* 为了下游尽快做CheckPoint，所以会先发送barrier到下游，自身再同步进行快照

3. 当task完成备份后，会将备份数据的地址（state handle）通知给JobManager的CheckPointCoordinator

* 如果CheckPoint的持续时长超过 了CheckPoint设定的超时时间，CheckPointCoordinator 还没有收集完所有的 State Handle，CheckPointCoordinator就会认为本次CheckPoint失败，会把这次CheckPoint产生的所有 状态数据全部删除

4. 最后 CheckPoint Coordinator 会把整个 StateHandle 封装成 completed CheckPoint Meta，写入到hdfs

  

  

| Checkpoint | Savepoint |

| :---------------- | :---------------------------|

| 由 Flink 的 JobManager 定时自动触发并管理 | 由用户手动触发并管理 |

| 主要用于任务发生故障时，为任务提供给自动恢复机制 | 主要用户升级 Flink 版本、修改任务的逻辑代码、调整算子的并行度，且必须手动恢复 |

| Flink 任务停止后，Checkpoint 的状态快照信息默认被清除 | 一旦触发 Savepoint，状态信息就被持久化到外部存储，除非用户手动删除 |

| Checkpoint 设计目标：轻量级且尽可能快地恢复任务 | Savepoint 的生成和恢复成本会更高一些，Savepoint 更多地关注代码的可移植性和兼容任务的更改操作 |

  

``` java

StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

env.enableCheckpointing(1000);

env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);

env.getCheckpointConfig().setMinPauseBetweenCheckpoints(500);

env.getCheckpointConfig().setCheckpointTimeout(60000);

env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);

env.getCheckpointConfig().enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);

```

![0c030dfe82e539a067d0f39d1c242a50.png](en-resource://database/522:1)

  

Exactly Once时必须barrier对齐

![63dc5312db133e5ba06e2ae1f6f0fbc1.png](en-resource://database/534:1)

  

什么是barrier不对齐？

上述图中，当还有其他输入流的barrier还没有到达时，会把已到达的barrier之后的数据1、2、3搁置在缓冲区，等待其他流的barrier到达后才能处理barrier不对齐就是指当还有其他流的barrier还没到达时，为了不影响性能，也不用理会，直接处理barrier之后的数据。等到所有流的barrier的都到达后，就可以对该Operator做CheckPoint了为什么要进行barrier对齐？不对齐到底行不行？答：Exactly Once时必须barrier对齐，如果barrier不对齐就变成了At Least Once后面的部分主要证明这句话CheckPoint的目的就是为了保存快照，如果不对齐，那么在chk-100快照之前，已经处理了一些chk-100 对应的offset之后的数据，当程序从chk-100恢复任务时，chk-100对应的offset之后的数据还会被处理一次，所以就出现了重复消费。如果听不懂没关系，后面有案例让您懂

  

``` shell

// 设置 checkpoint全局设置保存点

state.checkpoints.dir: hdfs:///checkpoints/

// 设置checkpoint 默认保留 数量

state.checkpoints.num-retained: 20

  

//从checkpoint恢复

bin/flink run -s hdfs://namenode:9000/flink/checkpoints/467e17d2cc343e6c56255d222bae3421/chk-56/_metadata flink-job.jar

```

  

savepoint的使用

```shell

flink savepoint jobId [targetDirectory] [-yid yarnAppId]【针对on yarn模式需要指定-yid参数】

  

flink cancel -s [targetDirectory] jobId [-yid yarnAppId]【针对on yarn模式需要指定-yid参数】

  

flink  savepoint 2f0511c8fe430acb81a968aff0cbb5b9 -d hdfs:/user/root/flink -yid application_1560242904558_0258 或者flink  savepoint 2f0511c8fe430acb81a968aff0cbb5b9 -yid application_1560242904558_0258 这里-d是指定位置

 flink cancel  -s hdfs:/user/root/flink 2f0511c8fe430acb81a968aff0cbb5b9  -yid application_1560242904558_0258 或者flink cancel  -s 2f0511c8fe430acb81a968aff0cbb5b9  -yid application_1560242904558_0258 这里-s后面可以跟路径也可以不跟路径，取决于state.savepoints.dir有没有配置

  

flink run -s savepointPath [runArgs]

```

  

### 失败恢复重试策略

  

1.Fixed Delay Restart Strategy(

固定间隔延迟重启策略)

  

Flink默认支持的失败恢复重启策略；表现形式为：给定延迟间隔，给定重试次数，且两次连续重试尝试之间必须保证等待配置的固定时间，在重试了给定次数之后依然失败，那么将最终失败。

  

``` shell

restart-strategy: fixed-delay

restart-strategy.fixed-delay.attempts: 3

restart-strategy.fixed-delay.delay: 10 s

  

env.setRestartStrategy(RestartStrategies.fixedDelayRestart( 3,// 尝试重启的次数

Time.of(10, TimeUnit.SECONDS) // 间隔 ));

```

2.Failure Rate Restart Strategy(失败率)

失败率计算公式失败率 = 失败次数/时间区间

  

失败率重启策略在Job失败后会重启，但是超过失败率后，Job会最终被认定失败。在两个连续的重启尝试之间，重启策略会等待一个固定的时间

  

下面配置是5分钟内若失败了3次则认为该job失败，重试间隔为10s

```shell

restart-strategy: failure-rate

restart-strategy.failure-rate.max-failures-per-interval: 3

restart-strategy.failure-rate.failure-rate-interval: 5 min

restart-strategy.failure-rate.delay: 10 s

env.setRestartStrategy(RestartStrategies.failureRateRestart( 3,//一个时间段内的最大失败次数

Time.of(5, TimeUnit.MINUTES), // 衡量失败次数的是时间段 Time.of(10, TimeUnit.SECONDS) // 间隔 ));

```

3.无重启策略

```shell

restart-strategy: none

  

ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment(); env.setRestartStrategy(RestartStrategies.noRestart());

```

  

### Late element

![a28f92826023a7373be8db8cf5d2adff.png](en-resource://database/537:1)

  

### run command

```shell

bin/start-cluster.sh

bin/flink run -d examples/streaming/TopSpeedWindowing.jar

bin/flink list -m 127.0.0.1:8081

bin/flink stop -m 127.0.0.1:8081 d67420e52bd051fae2fddbaa79e046bb

bin/flink cancel -m 127.0.0.1:8081 5e20cb6b0f357591171dfcca2eea09de

bin/flink cancel -m 127.0.0.1:8081 -s /tmp/savepoint 29da945b99dea6547c3fbafd57ed8759

bin/flink savepoint -m 127.0.0.1:8081 ec53edcfaeb96b2a5dadbfbe5ff62bbb /tmp/savepoint

  

#从指定的Savepiont启动

bin/flink run -d -s /tmp/savepoint/savepoint-f049ff-24ec0d3e0dc7

bin/flink run -m yarn-cluster ./examples/batch/WordCount.jar

```

  

### Datastream基本转换

![9d29bd86a688127a509936452418c148.png](en-resource://database/536:2)

  

### 反压

![2018d18cfa54c57e91d6bfafdbfda3e9.png](en-resource://database/525:1)

![5bb19de75e211012d7506f6171957dd9.png](en-resource://database/533:1)

![84ba9eedf0fd286d8c84a5d28b954cf2.png](en-resource://database/535:1)

  

### WaterMark

  

https://www.jianshu.com/p/9db56f81fa2a

  

watermark是一种衡量Event Time进展的机制，它是数据本身的一个隐藏属性。通常基于Event Time的数据，自身都包含一个timestamp.watermark是用于处理乱序事件的，而正确的处理乱序事件，通常用watermark机制结合window来实现。

  

  

AssignerWithPeriodicWatermarks

AssignerWithPunctuatedWatermarks

  

  

## Flink优秀博文收集

  

一文搞懂 Flink 网络流控与反压机制

https://www.jianshu.com/p/2779e73abcb8

  

https://mp.weixin.qq.com/s/6kqUEkJdsZgMIknqSPlzbQ

  

一文搞懂Flink内部的Exactly Once和At Least Once

https://www.jianshu.com/p/8d6569361999

  

https://mp.weixin.qq.com/s/1wlXeW_TAXtU1kV_uOZa4Q

  

### Data Sources

  

Data SourcesData Source的核心组件包括三个，分别是Split、SourceReader、SplitEnumerator。

* Split：Split是数据源的一部分，例如：文件的一部分或者一条日志。Splits是Source并行读取数据和分发作业的基础。

* SourceReader：SourceReader向SplitEnumerator请求Splits并处理请求到的Splits。SourceReader位于TaskManager中，这意味着它是并行运行的，同时，它可以产生并行的事件流/记录流。

* SplitEnumerator：SplitEnumerator负责管理Splits并且将他们发送给SourceReader。它是运行在JobManager上的单个实例。负责维护正在进行的分片的备份日志并将这些分片均衡的分发到SourceReader中。

![6250c80582c41047aa35854df9a851b5.png](en-resource://database/666:1)

  

  

## Flink 部署

### Flink作业的3种部署模式

#### Session模式

Session模式预分配资源，提前根据指定的资源参数初始化一个Flink集群，常驻在YARN系统中，拥有固定数量的JobManager和TaskManager（注意JobManager只有一个）。提交到这个集群的作业可以直接运行，免去每次分配资源的overhead。

特点：1. 集群和作业的生命周期不同，Session模式下，Flink集群的资源不会因为Flink作业的上下线而释放，Flink集群和提交到集群中的Flink作业的生命周期是相互独立的。

2. 不同Flink作业间的资源不隔离。

3. 作业部署速度快。

  

应用场景：1. 常驻核心高优Flink作业。2. 即席查询场景。

  

#### Per-Job

Per-Job模式仅支持YARN作为资源提供框架，在Per-Job模式下，对于用户提交的每一个Flink作业，都会从资源提供框架中为其申请并启动独立的Flink集群资源，集群中会包含一个JobManager以及这个作业所需要的所有TaskManager，JobManager和TaskManager只供这个作业使用。

  

特点：1. 集群和作业的生命周期相同。 2. 不同Flink作业间的资源完全隔离。 3. Flink作业部署速度较慢。4. 资源抢占。

废弃，用Application模式代替

  

#### Application

Applicaiotn同Per-Job模式最大的不同就是，Application模式中JobManager负责解析Flink作业并下载依赖。

  

  

### Local

  

bin/start-cluster.sh

ip:8081

  

### standalone

  

masters：master.hadooop.ljs:8081    

slaves： worker1.hadoop.ljs worker2.hadoop.ljs

  

### Flink On Yarn

  

yarn session模式：

bin/yarn-session.sh -n 2 -s 2 -jm 768 -tm 768

-n : TaskManager的数量，相当于executor的数量   

-s : 每个JobManager的core的数量，executor-cores。建议将slot的数量设置每台机器的处理器数量   

-tm : 每个TaskManager的内存大小，executor-memory   

-jm : JobManager的内存大小，driver-memory     

-s : 每个taskmanager的slot槽位数  默认是1

  

提交任务：

  

 /data/app/flink-1.10.0/bin/flink run   -m yarn-cluster -yid application_1585709615515_0004   -ys 2    /data/app/flink-1.10.0/examples/batch/WordCount.jar

-m :  指定运行模式 这里是yarn-cluster集群模式    

-yid : 就是你启动的yarn-session在yarn上的任务ID,重启yarn这个ID会变化；    

-ys:  每个taskmanager上的槽位数slot

  

直接在yarn上运行任务：

  

/data/app/flink-1.10.0/bin/flink run -ytm 1024 -ys 2 /data/app/flink-1.10.0/examples/batch/WordCount.jar

  

## Flink SQL

  

### FlinkSQL原理

![22bf4acd3953af055d959e9d560ae3b6.png](en-resource://database/526:1)

  

### Flink Stream SQL

  

#### 打包编译

```shell

mvn clean package -Dmaven.test.skip

```

  

#### 可运行的目录结构:

``` text

|

|-----bin

| |--- submit.sh 任务启动脚本

|-----lib: launcher包存储路径，是任务提交的入口

| |--- sql.launcher.jar

|-----plugins: 插件包存储路径(mvn 打包之后会自动将jar移动到该目录下)

| |--- core.jar

| |--- xxxsource

| |--- xxxsink

| |--- xxxside

```

  

#### 运行

  

``` shell

export HADOOP_USER_NAME=hdfs

  

sh /root/flinkStreamSQL/bin/submit.sh -sql /root/flinkStreamSQL/job/kafka2kafka.sql -name flinkStreamSQL-test -localSqlPluginPath /root/flinkStreamSQL/sqlplugins/ -remoteSqlPluginPath /root/flinkStreamSQL/sqlplugins/ -mode yarnPer -flinkconf /opt/flink/conf/ -yarnconf /etc/hadoop/conf.cloudera.yarn -flinkJarPath /opt/flink/lib/ -pluginLoadMode shipfile -confProp \{\"time.characteristic\":\"EventTime\",\"logLevel\":\"debug\"}

```

  

![dcdb351a35d1ee49b2de8d622f02f0d0.png](en-resource://database/549:1)

  

## Table & SQL

* Each identifier consists of 3 parts: catalog name, database name and object name

  

```java

  

tableEnvironment

.connect(...)

.withFormat(...)

.withSchema(...)

.inAppendMode()

.createTemporaryTable("MyTable")

```

  

SELECT user,TUMBLE_END(cTime,INTERVAL '1' HOURS) AS endT,

COUNT (url) AS cnt FROM clicks GROUP BY

user,TUMBLE(cTime,INTERVAL '1' HOURS)

  

### Time Attributes

  

```java

  

CREATE TABLE user_actions (

user_name STRING,

data STRING,

user_action_time AS PROCTIME() -- declare an additional field as a processing time attribute) WITH (

...);

  

SELECT TUMBLE_START(user_action_time, INTERVAL '10' MINUTE), COUNT(DISTINCT user_name)FROM user_actionsGROUP BY TUMBLE(user_action_time, INTERVAL '10' MINUTE);

  

  

DataStream<Tuple2<String, String>> stream = ...;

  

// declare an additional logical field as a processing time attributeTable table = tEnv.fromDataStream(stream, $("user_name"), $("data"), $("user_action_time").proctime());

  

WindowedTable windowedTable = table.window(

Tumble.over(lit(10).minutes())

.on($("user_action_time"))

.as("userActionWindow"));

TableSource DefinedProctimeAttribute 

```

  

## Flink优化

### Expression Reduce

表达式(Expression)优化：比如1+2+t1.value会变成3+t1.value

  

### PushDown Optimization

  

下推优化是指在保持关系代数语义不变的前提下将 SQL 语句中的变换操作尽可能下推到靠近数据源的位置以获得更优的性能，常见的下推优化有谓词下推（Predicate Pushdown），投影下推（Projection Pushdown，有时也译作列裁剪）等。

![a6a1163c5a7e516656b0d64f328efdb7.png](en-resource://database/547:1)

![4665c73517b1943c5326dd0e44b9179d.png](en-resource://database/546:1)

  

### BinaryRow

  

### MiniBatch

  

table.exec.mini-batch.enabled:true

table.exec.mini-batch.allow-latency:5s

table.exec.mini-batch.size:5000

  

### Local-Global Agg

数据倾斜

![ec306554cd079213d29eb1a7fd966f00.png](en-resource://database/542:1)

  

```java

  

// instantiate table environment

val tEnv: TableEnvironment = ...

  

// access flink configuration

val configuration = tEnv.getConfig().getConfiguration()

// set low-level key-value options

// local-global aggregation depends on mini-batch is enabled

configuration.setString("table.exec.mini-batch.enabled", "true")

configuration.setString("table.exec.mini-batch.allow-latency", "5 s")

configuration.setString("table.exec.mini-batch.size", "5000")

// enable two-phase, i.e. local-global aggregation

configuration.setString("table.optimizer.agg-phase-strategy", "TWO_PHASE")

```

  

## Flink参数调优

### Flink参数配置。

* jobmanger.rpc.address jm的地址。

* jobmanager.rpc.port jm的端口号。

* jobmanager.heap.mb jm的堆内存大小。不建议配的太大，1-2G足够。

* taskmanager.heap.mb tm的堆内存大小。大小视任务量而定。需要存储任务的中间值，网络缓存，用户数据等。taskmanager.numberOfTaskSlots slot数量。在yarn模式使用的时候会受到yarn.scheduler.maximum-allocation-vcores值的影响。此处指定的slot数量如果超过yarn的maximum-allocation-vcores，flink启动会报错。在yarn模式，flink启动的task manager个数可以参照如下计算公式：num_of_tm = ceil(parallelism / slot)  即并行度除以slot个数，结果向上取整。

* parallelsm.default 任务默认并行度，如果任务未指定并行度，将采用此设置。

* web.port Flink web ui的端口号。

* jobmanager.archive.fs.dir 将已完成的任务归档存储的目录。

* history.web.port 基于web的history server的端口号。

* historyserver.archive.fs.dir history server的归档目录。该配置必须包含jobmanager.archive.fs.dir配置的目录，以便history server能够读取到已完成的任务信息。

* historyserver.archive.fs.refresh-interval 刷新存档作业目录时间间隔state.backend 存储和检查点的后台存储。可选值为rocksdb filesystem hdfs。

* state.backend.fs.checkpointdir 检查点数据文件和元数据的默认目录。

* state.checkpoints.dir 保存检查点目录。

* state.savepoints.dir save point的目录。

* state.checkpoints.num-retained 保留最近检查点的数量。

* state.backend.incremental 增量存储。

* akka.ask.timeout Job Manager和Task Manager通信连接的超时时间。如果网络拥挤经常出现超时错误，可以增大该配置值。

* akka.watch.heartbeat.interval 心跳发送间隔，用来检测task manager的状态。

* akka.watch.heartbeat.pause 如果超过该时间仍未收到task manager的心跳，该task manager 会被认为已挂掉。

* taskmanager.network.memory.max 网络缓冲区最大内存大小。

* taskmanager.network.memory.min 网络缓冲区最小内存大小。

* taskmanager.network.memory.fraction 网络缓冲区使用的内存占据总JVM内存的比例。如果配置了taskmanager.network.memory.max和taskmanager.network.memory.min，本配置项会被覆盖。

* fs.hdfs.hadoopconf hadoop配置文件路径（已被废弃，建议使用HADOOP_CONF_DIR环境变量）

* yarn.application-attempts job失败尝试次数，主要是指job manager的重启尝试次数。该值不应该超过yarn-site.xml中的yarn.resourcemanager.am.max-attemps的值。

### Flink HA(Job Manager)的配置

* high-availability: zookeeper 使用zookeeper负责HA实现high-availability.

* zookeeper.path.root: /flink flink信息在zookeeper存储节点的名称

* high-availability.zookeeper.quorum: zk1,zk2,zk3 zookeeper集群节点的地址和端口

* high-availability.storageDir: hdfs://nameservice/flink/ha/ job manager元数据在文件系统储存的位置，zookeeper仅保存了指向该目录的指针。

### Flink metrics 监控相关配置

* metrics.reporters: prommetrics.

* reporter.prom.class: org.apache.flink.metrics.prometheus.PrometheusReportermetrics.

* reporter.prom.port: 9250-9260

### Kafka相关调优配置

* linger.ms/batch.size 这两个配置项配合使用，可以在吞吐量和延迟中得到最佳的平衡点。batch.size是kafka producer发送数据的批量大小，当数据量达到batch size的时候，会将这批数据发送出去，避免了数据一条一条的发送，频繁建立和断开网络连接。但是如果数据量比较小，导致迟迟不能达到batch.size，为了保证延迟不会过大，kafka不能无限等待数据量达到batch.size的时候才发送。为了解决这个问题，引入了linger.ms配置项。当数据在缓存中的时间超过linger.ms时，无论缓存中数据是否达到批量大小，都会被强制发送出去。

* ack 数据源是否需要kafka得到确认。all表示需要收到所有ISR节点的确认信息，1表示只需要收到kafka leader的确认信息，0表示不需要任何确认信息。该配置项需要对数据精准性和延迟吞吐量做出权衡。Kafka topic分区数和Flink并行度的关系Flink kafka source的并行度需要和kafka topic的分区数一致。最大化利用kafka多分区topic的并行读取能力。

### Yarn相关调优配置

* yarn.scheduler.maximum-allocation-vcores

* yarn.scheduler.minimum-allocation-vcores Flink单个task manager的slot数量必须介于这两个值之间

* yarn.scheduler.maximum-allocation-mb

* yarn.scheduler.minimum-allocation-mb Flink的job manager 和task manager内存不得超过container最大分配内存大小。

* yarn.nodemanager.resource.cpu-vcores yarn的虚拟CPU内核数，建议设置为物理CPU核心数的2-3倍，如果设置过少，会导致CPU资源无法被充分利用，跑任务的时候CPU占用率不高。

  

  

## 反压

### 反压采样

Task 是否反压是基于输出 Buffer 的可用性判断的，如果一个用于数据输出的 Buffer 都没有了，则表明 Task 被反压了。

  

默认情况下，JobManager 会触发 100 次采样，每次间隔 50ms 来确定反压。 你在 Web 界面看到的比率表示在获得的样本中有多少表明 Task 正在被反压，例如: 0.01 表示 100 个样本中只有 1 个反压了。

  

* OK: 0 <= 比例 <= 0.10

* LOW: 0.10 < 比例 <= 0.5

* HIGH: 0.5 < 比例 <= 1

为了不因为采样导致 TaskManager 负载过重，Web 界面仅在每 60 秒后重新采样。

  

![1544606dab8cfdc4c2baeda77f552f76.png](en-resource://database/548:1)

  

### 参数配置

  

你可以使用以下键来配置 JobManager 的样本数：

* web.backpressure.refresh-interval: 有效的反压结果被废弃并重新进行采样的时间 (默认: 60000, 1 min)。

* web.backpressure.num-samples: 用于确定反压采样的样本数 (默认: 100)。

* web.backpressure.delay-between-samples: 用于确定反压采样的间隔时间 (默认: 50, 50 ms)。

  

## Flink CDC

  

https://blog.csdn.net/zhangjun5965

![7d37365dccce62819b31c09d91e12d32.png](en-resource://database/664:1)

  

  

https://www.136.la/jingpin/show-206585.html

  

## 源码

### yarn-per-job

https://zhangboyi.blog.csdn.net/article/details/115744316

  

org.apache.flink.runtime.entrypoint.StandaloneSessionClusterEntrypoint

  

org.apache.flink.runtime.taskexecutor.TaskManagerRunner

  

flink run提交程序入口：

org.apache.flink.client.cli.CliFrontend

  

Yarn-Per-Job

![0a29a3345ebd12cded7da4f6134b0825.png](en-resource://database/691:1)

  

CliFrontend

  

1. 参数解析

2. 封装CommandLine：三个，依次添加

3. 配置的封装

4. 执行用户代码： execute()

5. 生成StreamGraph

6. Executor：生成JobGraph

7. 集群描述器：上传jar包、配置， 封装提交给yarn的命令

8. yarnclient提交应用

  

YarnJobClusterEntryPoint ：AM执行的入口类

  

1. Dispatcher的创建和启动

2. ResourceManager的创建、启动：里面有一个 slotmanager（真正管理资源的、向yarn申请资源）

3. Dispatcher启动JobMaster：生成ExecutionGraph（里面有一个slotpool，真正去发送请求的）

4. slotpool向slotmanager申请资源， slotmanager向yarn申请资源（启动新节点）

  

YarnTaskExecutorRunner：Yarn模式下的TaskManager的入口类

  

1. 启动 TaskExecutor

2. 向ResourceManager注册slot

3. ResourceManager分配slot

4. TaskExecutor接收到分配的指令，提供offset给JobMaster（slotpool）

5. JobMaster提交任务给TaskExecutor去执行

  

  

### Source架构

JobManager->SourceCoordinator->Source.createEnumerator() (

SplitEnumerator)

  

TaskManager->SourceOperator->Source.createReader() (SourceReader)

  

#### Source接口

Source接口中定义的方法用来创建读取数据时必须的组件，其中比较关键的有下面两个

* createEnumerator方法：运行在JobManager端的SourceCoordinator调用此方法获取到SplitEnumerator对要读取的数据进行分片

* createReader方法：运行在TaskManager端的SourceOperator通过此方法获取到SourceReader读取数据

  

SplitEnumerator

SplitEnumerator主要用来处理分片相关的信息主要的方法有下面几个

* start方法：对要读取的数据进行分片

* handleSplitRequest方法：当SourceReader需要获取分片读取数据时会通过RPC请求发送RequestSplitEvent请求分片，最终处理这些请求的方法就是handleSplitRequest方法

* handleSourceEvent方法：处理来自SourceReader的自定义事件的方法，开发时可以通过自定义SourceEvent在SplitEnumerator和SourceReader之间传递信息进行通信

  

  

#### SourceReader

SourceReader用来读取指定分片内的数据，主要的方法有下面几个

* start方法：启动Reader向SplitEnumerator请求数据分片

* pollNext方法：获取SourceReader生产的数据发送到下游算子

* addSplits方法：当SourceReader通过RPC请求发送RequestSplitEvent请求分片的事件被SplitEnumerator处理后SplitEnumerator会为SourceReader分配分片并返回AddSplitEvent，最终会调用addSplits方法将分片信息添加到SourceReader中

* handleSourceEvents方法：处理来自SplitEnumerator的自定义事件的方法

  

  

#### SourceCoordinator

SourceCoordinator运行在JobManager中使用event loop线程模型和Flink runtime交互，主要负责SplitEnumerator的创建、启动和状态管理，同时也能接受来自SourceOperator的事件进行处理

  

  

#### SourceOperator

SourceOperator运行在TaskManager中，主要负责SourceReader的创建、初始化和状态管理，同时还会处理来自SourceCoordinator的自定义事件，除了之外SourceOperator还负责和Flink框架的交互，Flink框架会定时获取SourceOperator的状态并将数据输出到下游算子

  

## Window

### Keyed Windows

stream

.keyBy(...) <- 仅 keyed 窗口需要

.window(...) <- 必填项："assigner"

[.trigger(...)] <- 可选项："trigger" (省略则使用默认 trigger)

[.evictor(...)] <- 可选项："evictor" (省略则不使用 evictor)

[.allowedLateness(...)] <- 可选项："lateness" (省略则为 0)

[.sideOutputLateData(...)] <- 可选项："output tag" (省略则不对迟到数据使用 side output)

.reduce/aggregate/apply() <- 必填项："function"

[.getSideOutput(...)] <- 可选项："output tag"

  

### Non-Keyed Windows

stream

.windowAll(...) <- 必填项："assigner"

[.trigger(...)] <- 可选项："trigger" (else default trigger)

[.evictor(...)] <- 可选项："evictor" (else no evictor)

[.allowedLateness(...)] <- 可选项："lateness" (else zero)

[.sideOutputLateData(...)] <- 可选项："output tag" (else no side output for late data)

.reduce/aggregate/apply() <- 必填项："function"

[.getSideOutput(...)] <- 可选项："output tag"