## Dataflow Programming Model
### Levels of Abstraction
![[levels_of_abstraction.svg]]
### Even Processing Time
![[event_processing_time.svg]]

## Flink架构
![[processes.svg]]  

* Program Code：我们编写的 Flink 应用程序代码
* Job Client：Job Client 不是 Flink 程序执行的内部部分，但它是任务执行的起点。 Job Client 负责接受用户的程序代码，然后创建数据流，将数据流提交给 Job Manager 以便进一步执行。 执行完成后，Job Client 将结果返回给用户
* Job Manager：主进程（也称为作业管理器）协调和管理程序的执行。 它的主要职责包括安排任务，管理 checkpoint ，故障恢复等。机器集群中至少要有一个 master，master 负责调度 task，协调 checkpoints 和容灾，高可用设置的话可以有多个 master，但要保证一个是 leader, 其他是 standby; Job Manager 包含3个重要组件，分别是Dispatcher、ResourceManager、JobMaster；通常情况下，一个Job Manager包含一个Dispatcher、一个ResourceManager和多个JobMaster。
* Task Manager：从 Job Manager 处接收需要部署的 Task。Task Manager 是在 JVM 中的一个或多个线程中执行任务的工作节点。 任务执行的并行性由每个 Task Manager 上可用的任务槽（Slot 个数）决定。 每个任务代表分配给任务槽的一组资源。 例如，如果 Task Manager 有四个插槽，那么它将为每个插槽分配 25％ 的内存。 可以在任务槽中运行一个或多个线程。 同一插槽中的线程共享相同的 JVM。
### Tasks和算子链
对于分布式执行，Flink 将算子的 subtasks _链接_成 _tasks_。每个 task 由一个线程执行。将算子链接成 task 是个有用的优化：它减少线程间切换、缓冲的开销，并且减少延迟的同时增加整体吞吐量。链行为是可以配置的。

下图中样例数据流用 5 个 subtask 执行，因此有 5 个并行线程。
![[tasks_chains.svg]]

### Watermarks in Parallel Streams
![[parallel_streams_watermarks.svg]]
如果算子有多个输入流，选择输入流最小的那个作为watermark。

### Task Slots和资源
每个 worker（TaskManager）都是一个 _JVM 进程_，可以在单独的线程中执行一个或多个 subtask。为了控制一个 TaskManager 中接受多少个 task，就有了所谓的 **task slots**（至少一个）。

每个 _task slot_ 代表 TaskManager 中资源的固定子集。例如，具有 3 个 slot 的 TaskManager，会将其托管内存 1/3 用于每个 slot。分配资源意味着 subtask 不会与其他作业的 subtask 竞争托管内存，而是具有一定数量的保留托管内存。注意此处没有 CPU 隔离；当前 slot 仅分离 task 的托管内存。

通过调整 task slot 的数量，用户可以定义 subtask 如何互相隔离。每个 TaskManager 有一个 slot，这意味着每个 task 组都在单独的 JVM 中运行（例如，可以在单独的容器中启动）。具有多个 slot 意味着更多 subtask 共享同一 JVM。同一 JVM 中的 task 共享 TCP 连接（通过多路复用）和心跳信息。它们还可以共享数据集和数据结构，从而减少了每个 task 的开销。
![[tasks_slots.svg]]

默认情况下，Flink 允许 subtask 共享 slot，即便它们是不同的 task 的 subtask，只要是来自于同一作业即可。结果就是一个 slot 可以持有整个作业管道。允许 _slot 共享_有两个主要优点：

- Flink 集群所需的 task slot 和作业中使用的最大并行度恰好一样。无需计算程序总共包含多少个 task（具有不同并行度）。
- 容易获得更好的资源利用。如果没有 slot 共享，非密集 subtask（_source/map()_）将阻塞和密集型 subtask（_window_） 一样多的资源。通过 slot 共享，我们示例中的基本并行度从 2 增加到 6，可以充分利用分配的资源，同时确保繁重的 subtask 在 TaskManager 之间公平分配。
![[slot_sharing.svg]]
### Flink应用程序执行
#### Session模式
Session模式预分配资源，提前根据指定的资源参数初始化一个Flink集群，常驻在YARN系统中，拥有固定数量的JobManager和TaskManager（注意JobManager只有一个）。提交到这个集群的作业可以直接运行，免去每次分配资源的overhead。

特点：
1. 集群和作业的生命周期不同，Session模式下，Flink集群的资源不会因为Flink作业的上下线而释放，Flink集群和提交到集群中的Flink作业的生命周期是相互独立的。
2. 不同Flink作业间的资源不隔离。
3. 作业部署速度快。

应用场景：
1. 常驻核心高优Flink作业。
2. 2. 即席查询场景。
#### Per-Job
Per-Job模式仅支持YARN作为资源提供框架，在Per-Job模式下，对于用户提交的每一个Flink作业，都会从资源提供框架中为其申请并启动独立的Flink集群资源，集群中会包含一个JobManager以及这个作业所需要的所有TaskManager，JobManager和TaskManager只供这个作业使用。

特点：
1. 集群和作业的生命周期相同。 
2. 不同Flink作业间的资源完全隔离。 
3. Flink作业部署速度较慢。
4. 资源抢占。
废弃，用Application模式代替
#### Application
Applicaiotn同Per-Job模式最大的不同就是，Application模式中JobManager负责解析Flink作业并下载依赖。
#### Local
./bin/start-cluster.sh
ip:8081
#### standalone
masters：master.hadooop.ljs:8081    
slaves： worker1.hadoop.ljs worker2.hadoop.ljs

### Checkpoint
![[checkpoints.svg]]

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
| Checkpoint 设计目标：轻量级且尽可能快地恢复任务 | Savepoint 的生成和恢复成本会更高一些，Savepoint 更多地关注代码的可移植性和兼容任务的更改操作 |[[]]

``` java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
//设置Checkpoint的执行时间间隔，单位为ms，当前配置代表每1min执行一次Checkpoint
env.enableCheckpointing(60000);
//设置快照持久化存储地址，当前配置代表将快照存储到HDFS的hdfs:///my/checkpoint/dir目录中
env.getCheckpointConfig().setCheckpointStorage("hdfs:///my/checkpoint/dir");
//设置Checkpoint的一致性语义，当前配置代表语义为精确一次
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
//设置Checkpoint的超时时间，单位为ms，如果执行超时，则认为执行Checkpoint失败。
env.getCheckpointConfig().setCheckpointTimeout(60000);
//连续两次Checkpoint执行的最小间隔时间，单位为ms。
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(5000);
//设置作业同时执行Checkpoint的数目，当前配置表示作业同一时间只允许执行一个Checkpoint
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);
env.getCheckpointConfig().enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
```

Exactly Once时必须barrier对齐

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

## Flink优秀博文收集
一文搞懂 Flink 网络流控与反压机制
https://www.jianshu.com/p/2779e73abcb8
https://mp.weixin.qq.com/s/6kqUEkJdsZgMIknqSPlzbQ
一文搞懂Flink内部的Exactly Once和At Least Once
https://www.jianshu.com/p/8d6569361999
https://mp.weixin.qq.com/s/1wlXeW_TAXtU1kV_uOZa4Q

## Flink SQL

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
* 
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

## Source架构

JobManager->SourceCoordinator->Source.createEnumerator() (SplitEnumerator)
TaskManager->SourceOperator->Source.createReader() (SourceReader)

### Source接口
Source接口中定义的方法用来创建读取数据时必须的组件，其中比较关键的有下面两个
* createEnumerator方法：运行在JobManager端的SourceCoordinator调用此方法获取到SplitEnumerator对要读取的数据进行分片
* createReader方法：运行在TaskManager端的SourceOperator通过此方法获取到SourceReader读取数据
* 
SplitEnumerator
SplitEnumerator主要用来处理分片相关的信息主要的方法有下面几个
* start方法：对要读取的数据进行分片
* handleSplitRequest方法：当SourceReader需要获取分片读取数据时会通过RPC请求发送RequestSplitEvent请求分片，最终处理这些请求的方法就是handleSplitRequest方法
* handleSourceEvent方法：处理来自SourceReader的自定义事件的方法，开发时可以通过自定义SourceEvent在SplitEnumerator和SourceReader之间传递信息进行通信

### SourceReader
SourceReader用来读取指定分片内的数据，主要的方法有下面几个
* start方法：启动Reader向SplitEnumerator请求数据分片
* pollNext方法：获取SourceReader生产的数据发送到下游算子
* addSplits方法：当SourceReader通过RPC请求发送RequestSplitEvent请求分片的事件被SplitEnumerator处理后SplitEnumerator会为SourceReader分配分片并返回AddSplitEvent，最终会调用addSplits方法将分片信息添加到SourceReader中
* handleSourceEvents方法：处理来自SplitEnumerator的自定义事件的方法

### SourceCoordinator
SourceCoordinator运行在JobManager中使用event loop线程模型和Flink runtime交互，主要负责SplitEnumerator的创建、启动和状态管理，同时也能接受来自SourceOperator的事件进行处理

### SourceOperator
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

## 源码研读