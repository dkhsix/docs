# 1 [Spark](http://spark.apache.org/)特性

- 版本号 x.y.z  3.1.1
  - x：Major version(change APIs)	
  - y：Minor version(add APIs/features)
  - z: Patch version(bug fixes)
  - 如何选择：能选最新的尽可能选最新的，尽量选择z最大的

## 1.1 优点

* 1）speed
  * both batch and streaming data（批，流一体：Spark、Flink）=> 快，从哪些角度？运行角度（memory、dag、thread）、开发角度（Python、Scala、SQL）
* 2）易用性
	* high-level operators
* 3）通用性Generality
	* 	stack  生态
	![](./pictures/Spark/模块.png)
* 4）Runs Everywhere
  * It can access diverse data sources.
  * YARN/Local/StandLone => Spark应用程序不需要改的, master来指定运行在哪个模式下


## 1.2 Spark与hadoop对比
### 模块对比

| use case          | other        | Spark                   |
| ----------------- | ------------ | ----------------------- |
| Batch processing  | MR           | RDD、DataFrame、DataSet |
| SQL querying      | Hive         | Spark SQL               |
| Stream processing | Kafka        | Spark Streaming、SSS    |
| ML                | Mahout       | Spark ML lib            |
| real time lookups | NoSQL(Hbase) | DataSourceAPI           |

### 计算方式对比

| Hadoop                                    | Spark                                                        |
| ----------------------------------------- | ------------------------------------------------------------ |
| Distributed Storage + Distributed Compute | Distributed Compute Only                                     |
| MR framework                              | Generalized computation（Stack）                             |
| usually Data on disk(HDFS)                | on disk/memory                                               |
| Not ideal for iterative work              | great at iterative workloads(ML ...)                         |
| batch process                             | Up to 2x - 10x faster forvdata on disk ; up to 100x faster for data in memeory |
|                                           | Compact code Java Python Scala supported                     |

### 处理方式

- MapReduce
  - map==>磁盘   reduce==>HDFS
  - 一个作业由多个MR作业所构成 A ==> B ==> C ==> D
  - Task: JVM

![](./pictures/Spark/比较方式.PNG)



## 1.3 Spark运行原理

1. <u>Each application gets its own executor processes</u>, which stay up for the duration of the whole application and run tasks in multiple threads. This has the benefit of **isolating applications from each other**, on both the **scheduling** side (each driver schedules its own tasks) and **executor** side (tasks from different applications run in different JVMs). However, it also means that **data cannot be shared across different Spark applications** (instances of SparkContext) without writing it to an external storage system.
2. <u>Spark is agnostic to the underlying cluster manager</u>（不感知底层cm）. As long as it can acquire executor processes, and these communicate with each other, it is relatively easy to run it even on a cluster manager that also supports other applications (e.g. Mesos/YARN/Kubernetes).
3. The driver program must listen for and accept incoming connections from its executors throughout its lifetime (e.g., see [spark.driver.port in the network config section](https://spark.apache.org/docs/latest/configuration.html#networking)). As such, the driver program must be network addressable from the worker nodes.
4. Because the driver schedules tasks on the cluster, it should be run close to the worker nodes, preferably on the same local area network. If you’d like to send requests to the cluster remotely, it’s better to open an RPC to the driver and have it submit operations from nearby than to run a driver far away from the worker nodes.

![](pictures/Spark/cluster-overview.png)

| Term                | Meaning                                                      |
| :------------------ | :----------------------------------------------------------- |
| **Application**     | User program built on Spark. Consists of a *driver program* and *executors* on the cluster.                                         => **Application = 1 Driver + N Executors** |
| Application jar     | A jar containing the user's Spark application. In some cases users will want to create an "uber jar" containing their application along with its dependencies. The user's jar should never include Hadoop or Spark libraries, however, these will be added at runtime. |
| **Driver program**  | The process running the **main()** function of the application and creating the **SparkContext** |
| **Cluster manager** | An external service(外部服务) for acquiring resources on the cluster (e.g. standalone manager, Mesos, YARN, Kubernetes) |
| Deploy mode         | Distinguishes where the driver process runs. In "**cluster**" mode, the framework launches the driver inside of the cluster. In "**client**" mode, the submitter launches the driver outside of the cluster.（决定Driver跑在哪） |
| Worker node         | Any node that can run application code in the cluster(Yarn的话就在Node manager上) |
| **Executor**        | A **process** launched for an application on a worker node, that **runs tasks** and **keeps data in memory or disk** storage across them. **Each application has its own executors**. |
| Task                | A unit of work that will be sent to one executor             |
| Job                 | **A parallel computation consisting of multiple tasks** that gets spawned in response to a Spark action (e.g. `save`, `collect`); you'll see this term used in the driver's logs. |
| Stage               | Each job gets divided into **smaller sets of tasks** (小的task的集合)called *stages* that depend on each other (similar to the map and reduce stages in MapReduce); you'll see this term used in the driver's logs |

- Deploy mode
  - client      Driver是运行本地的
  - cluster    Driver是运行在NODE
  - 提交机器可以在集群外部，但是必须可以访问到 HADOOP_CONF_DIR
- Job：action算子 触发  Job
- Stage：一个job可能会被拆分成多个stage

```shell
spark-shell --master yarn
jps
# 启动进程
- SparkSubmit
- YarnCoarseGrainedExecutorBackend
- YarnCoarseGrainedExecutorBackend
- ExecutorLauncher
```

### [内存优化](https://spark.apache.org/docs/latest/tuning.html#memory-tuning)

- Spark-shell

```shell
# driver
--driver-memory MEM         Memory for driver (e.g. 1000M, 2G) (Default: 1024M).
# executor
--executor-memory MEM       Memory per executor (e.g. 1000M, 2G) (Default: 1G)
--executor-cores NUM        Number of cores used by each executor. (Default: 1 in YARN and K8S modes, or 	   												  		all available cores on the worker in standalone mode).
```



#### Memory Management Overview

| Property Name                  | Default | Meaning                                                      | Since Version |
| :----------------------------- | :------ | :----------------------------------------------------------- | :------------ |
| `spark.memory.fraction`        | 0.6     | Fraction of (heap space - 300MB) used for execution and storage. The lower this is, the more frequently spills and cached data eviction occur. The purpose of this config is to set aside memory for internal metadata, user data structures, and imprecise size estimation in the case of sparse, unusually large records. Leaving this at the default value is recommended. For more detail, including important information about correctly tuning JVM garbage collection when increasing this value, see [this description](https://spark.apache.org/docs/latest/tuning.html#memory-management-overview). | 1.6.0         |
| `spark.memory.storageFraction` | 0.5     | Amount of storage memory immune to eviction, expressed as a fraction of the size of the region set aside by `spark.memory.fraction`. The higher this is, the less working memory may be available to execution and tasks may spill to disk more often. Leaving this at the default value is recommended. For more detail, see [this description](https://spark.apache.org/docs/latest/tuning.html#memory-management-overview). | 1.6.0         |
| `spark.memory.offHeap.enabled` | false   | If true, Spark will attempt to use off-heap memory for certain operations. If off-heap memory use is enabled, then `spark.memory.offHeap.size` must be positive. | 1.6.0         |
| `spark.memory.offHeap.size`    | 0       | The absolute amount of memory which can be used for off-heap allocation, in bytes unless otherwise specified. This setting has no impact on heap memory usage, so if your executors' total memory consumption must fit within some hard limit then be sure to shrink your JVM heap size accordingly. This must be set to a positive value when `spark.memory.offHeap.enabled=true`. | 1.6.0         |

Memory usage in Spark largely falls under one of two categories: **execution** and **storage**. 

- Execution memory refers to that used for computation in **shuffles, joins, sorts and aggregations**, 
- while storage memory refers to that used for **caching and propagating internal data across the cluster**. 
- In Spark, execution and storage **share a unified region** (M). <u>When no execution memory is used, storage can acquire all the available memory and vice versa. Execution may evict storage if necessary</u>, but only until total storage memory usage falls under a certain threshold (R).
-  In other words, `R` describes a subregion within `M` where cached blocks are never evicted. Storage may not evict execution due to complexities in implementation.

- `spark.memory.fraction` expresses the size of `M` as a fraction of the (JVM heap space - 300MiB) (default 0.6). The rest of the space (40%) is reserved for user data structures, internal metadata in Spark, and safeguarding against OOM errors in the case of sparse and unusually large records.
- `spark.memory.storageFraction` expresses the size of `R` as a fraction of `M` (default 0.5). `R` is the storage space within `M` where cached blocks immune to being evicted by execution.

The value of `spark.memory.fraction` should be set in order to fit this amount of heap space comfortably within the JVM’s old or “tenured” generation. See the discussion of advanced GC tuning below for details.

```scala
/** 
 1.6版本 有静态内存管理（难度大）
 作业cache用的少，但是由于是静态管理，有很多内存空间被白白浪费
 spark.storage.memoryFraction 调小
 spark.shuffle.memoryFraction 调大
 
 统一内存管理
 executor和Storage可以相互借用
 executor可以剔除storage内存
 */
private[spark] abstract class MemoryManager(
    conf: SparkConf,
    numCores: Int,
    onHeapStorageMemory: Long,
    onHeapExecutionMemory: Long) {
  val memoryManager: MemoryManager = UnifiedMemoryManager(conf, numUsableCores)
}

object UnifiedMemoryManager {
  // Set aside a fixed amount of memory for non-storage, non-execution purposes.
  // This serves a function similar to `spark.memory.fraction`, but guarantees that we reserve
  // sufficient memory for the system even for small heaps. E.g. if we have a 1GB JVM, then
  // the memory used for execution and storage will be (1024 - 300) * 0.6 = 434MB by default.
  private val RESERVED_SYSTEM_MEMORY_BYTES = 300 * 1024 * 1024
  
  private def getMaxMemory(conf: SparkConf): Long = {
    val systemMemory =  conf.get("spark.testing.memory", Runtime.getRuntime.maxMemory)
    val reservedMemory = RESERVED_SYSTEM_MEMORY_BYTES = 300 * 1024 * 1024 // 300M
    val usableMemory = systemMemory - reservedMemory                      // 700M
    val memoryFraction = conf.get("spark.memory.fraction", 0.6)           // 0.6
    (usableMemory * memoryFraction).toLong                                // 700 * 0.6 = 420M
  }
  
  maxHeapMemory = maxMemory                                               // 420M
  onHeapStorageRegionSize = 
  	(maxMemory * conf.get("spark.memory.storageFraction", 0.5)).toLong    // 0.5*420 = 210M
}
```

### Shuffle

[文章](https://tech.meituan.com/2016/05/12/spark-tuning-pro.html)

#### 历史

- 0.8：Hash Based Shuffle

- 0.8.1: File Consolidation机制
- 0.9: ExternalAppendOnlyMap
- 1.1：Sort Based Shuffle （default：hash）
- 1.2：default：Sort
- 1.4：Tungsten-Sorted
- 1.6：Tungsten并入Sorted
- 2.0：Hash退出历史

```scala
// SparkEnv.scala
// Let the user specify short names for shuffle managers
val shortShuffleMgrNames = Map(
  "sort" -> classOf[org.apache.spark.shuffle.sort.SortShuffleManager].getName,
  "tungsten-sort" -> classOf[org.apache.spark.shuffle.sort.SortShuffleManager].getName)
val shuffleMgrName = conf.get(config.SHUFFLE_MANAGER)
val shuffleMgrClass =
	shortShuffleMgrNames.getOrElse(shuffleMgrName.toLowerCase(Locale.ROOT), shuffleMgrName)
val shuffleManager = instantiateClass[ShuffleManager](shuffleMgrClass)

val memoryManager: MemoryManager = UnifiedMemoryManager(conf, numUsableCores)
```

#### Shuffle文件命名规则

```scala
/**
shuffle_0_0_1
shuffle_shuffle-id_mapid_reduceid
org.apache.spark.shuffle.api.ShuffleExecutorComponents

用spark-shell测试 版本1.5.2
设置 spark.local.dir /xx/x/x
*/
ShuffleMapOutputWriter createMapOutputWriter(
  int shuffleId,
  long mapTaskId,
  int numPartitions) throws IOException;
```



#### HashShuffleManager

##### 未优化

###### shuffle write

- shuffle write阶段，主要就是在一个stage结束计算之后，为了下一个stage可以执行shuffle类的算子（比如reduceByKey），而将每个task处理的数据按key进行“分类”。
  - 所谓“分类”，就是对相同的key执行hash算法，从而将相同key都写入同一个磁盘文件中，而每一个磁盘文件都只属于下游stage的一个task。
  - 在将数据写入磁盘之前，会先将数据写入内存缓冲中，当内存缓冲填满之后，才会溢写到磁盘文件中去。

- 每一个上游的task为每一个下游的task准备文件
  - 比如下一个stage总共有100个task，那么当前stage的每个task都要创建100份磁盘文件。如果当前stage有50个task，总共有10个Executor，每个Executor执行5个Task，那么每个Executor上总共就要创建500个磁盘文件，所有Executor上会创建5000个磁盘文件

###### shuffle read

- shuffle read，通常就是一个stage刚开始时要做的事情。
  - 此时该stage的每一个task就需要将上一个stage的计算结果中的所有相同key，从各个节点上通过网络都拉取到自己所在的节点上，然后进行key的聚合或连接等操作。
  - 由于shuffle write的过程中，task给下游stage的每个task都创建了一个磁盘文件，因此shuffle read的过程中，每个task只要从上游stage的所有task所在节点上，拉取属于自己的那一个磁盘文件即可。

- shuffle read的拉取过程是一边拉取一边进行聚合的。
  - 每个shuffle read task都会有一个自己的buffer缓冲，每次都只能拉取与buffer缓冲相同大小的数据，然后通过内存中的一个Map进行聚合等操作。
  - 聚合完一批数据后，再拉取下一批数据，并放到buffer缓冲中进行聚合操作。以此类推，直到最后将所有数据到拉取完，并得到最终的结果。

<img src="pictures/Spark/shuffle未优化.png" style="zoom:50%;" />

##### [优化](https://issues.apache.org/jira/browse/SPARK-751)

- 这里说的优化，是指我们可以设置一个参数，spark.shuffle.consolidateFiles。该参数默认值为false，将其设置为true即可开启优化机制。通常来说，如果我们使用HashShuffleManager，那么都建议开启这个选项。

- 开启consolidate机制之后，在shuffle write过程中，task就不是为下游stage的每个task创建一个磁盘文件了。此时会出现shuffleFileGroup的概念，每个shuffleFileGroup会对应一批磁盘文件，磁盘文件的数量与下游stage的task数量是相同的。一个Executor上有多少个CPU core，就可以并行执行多少个task。而第一批并行执行的每个task都会创建一个shuffleFileGroup，并将数据写入对应的磁盘文件内。
- 当Executor的CPU core执行完一批task，接着执行下一批task时，下一批task就会复用之前已有的shuffleFileGroup，包括其中的磁盘文件。也就是说，此时task会将数据写入已有的磁盘文件中，而不会写入新的磁盘文件中。因此，consolidate机制允许不同的task复用同一批磁盘文件，这样就可以有效将多个task的磁盘文件进行一定程度上的合并，从而大幅度减少磁盘文件的数量，进而提升shuffle write的性能。
- 假设第二个stage有100个task，第一个stage有50个task，总共还是有10个Executor，每个Executor执行5个task。那么原本使用未经优化的HashShuffleManager时，每个Executor会产生500个磁盘文件，所有Executor会产生5000个磁盘文件的。但是此时经过优化之后，每个Executor创建的磁盘文件的数量的计算公式为：**CPU core的数量 * 下一个stage的task数量**。也就是说，每个Executor此时只会创建100个磁盘文件，所有Executor只会创建1000个磁盘文件。

![](pictures/Spark/shuffle优化.png)



#### SortShuffleManager

SortShuffleManager的运行机制主要分成两种，一种是普通运行机制，另一种是bypass运行机制。**当shuffle read task的数量小于等于spark.shuffle.sort.bypassMergeThreshold参数的值时（默认为200），就会启用bypass机制**。

##### 普通运行机制

- 在该模式下，数据会先写入一个内存数据结构中，此时根据不同的shuffle算子，可能选用不同的数据结构。
  - 如果是reduceByKey这种聚合类的shuffle算子，那么会选用Map数据结构，一边通过Map进行聚合，一边写入内存；
  - 如果是join这种普通的shuffle算子，那么会选用Array数据结构，直接写入内存。接着，每写一条数据进入内存数据结构之后，就会判断一下，是否达到了某个临界阈值。如果达到临界阈值的话，那么就会尝试将内存数据结构中的数据溢写到磁盘，然后清空内存数据结构。
- 在溢写到磁盘文件之前，会先根据key对内存数据结构中已有的数据进行排序。排序过后，会分批将数据写入磁盘文件。
  - **默认的batch数量是10000条**，也就是说，排序好的数据，会以每批1万条数据的形式分批写入磁盘文件。
  - 写入磁盘文件是通过Java的BufferedOutputStream实现的。BufferedOutputStream是Java的缓冲输出流，首先会将数据缓冲在内存中，当内存缓冲满溢之后再一次写入磁盘文件中，这样可以减少磁盘IO次数，提升性能。

- 一个task将所有数据写入内存数据结构的过程中，会发生多次磁盘溢写操作，**也就会产生多个临时文件。最后会将之前所有的临时磁盘文件都进行合并，这就是merge过程**，此时会将之前所有临时磁盘文件中的数据读取出来，然后依次写入最终的磁盘文件之中。
  - 此外，由于一个task就只对应一个磁盘文件，也就意味着该task为下游stage的task准备的数据都在这一个文件中，**因此还会单独写一份索引文件，其中标识了下游各个task的数据在文件中的start offset与end offset(偏移量)**。

- SortShuffleManager由于有一个磁盘文件merge的过程，因此大大减少了文件数量。比如第一个stage有50个task，总共有10个Executor，每个Executor执行5个task，而第二个stage有100个task。由于每个task最终只有一个磁盘文件，因此此时每个Executor上只有5个磁盘文件，所有Executor只有50个磁盘文件。

![](pictures/Spark/sortshuffle1.png)



##### bypass运行机制

- bypass运行机制的触发条件如下： 
  -  **shuffle map task数量小于spark.shuffle.sort.bypassMergeThreshold参数的值**。
  - **不是聚合类的shuffle算子**（比如reduceByKey）。

- 此时task会为每个下游task都创建一个临时磁盘文件，并将数据按key进行hash然后根据key的hash值，将key写入对应的磁盘文件之中。当然，写入磁盘文件时也是先写入内存缓冲，缓冲写满之后再溢写到磁盘文件的。最后，同样会将所有临时磁盘文件都合并成一个磁盘文件，并创建一个单独的索引文件。

- 该过程的磁盘写机制其实跟未经优化的HashShuffleManager是一模一样的，因为都要创建数量惊人的磁盘文件，只是在最后会做一个磁盘文件的合并而已。因此少量的最终磁盘文件，也让该机制相对未经优化的HashShuffleManager来说，shuffle read的性能会更好。

- 而该机制与普通SortShuffleManager运行机制的不同在于：第一，磁盘写机制不同；第二，不会进行排序。也就是说，启用该机制的最大好处在于，shuffle write过程中，不需要进行数据的排序操作，也就节省掉了这部分的性能开销。

![](pictures/Spark/bypass.png)

#### [shuffle相关参数调优](https://spark.apache.org/docs/latest/configuration.html#shuffle-behavior)

以下是Shffule过程中的一些主要参数，这里详细讲解了各个参数的功能、默认值以及基于实践经验给出的调优建议。(**以官网为准，不同版本有变化**)

##### spark.shuffle.file.buffer

- 默认值：32k
- 参数说明：该参数用于设置shuffle write task的BufferedOutputStream的buffer缓冲大小。将数据写到磁盘文件之前，会先写入buffer缓冲中，待缓冲写满之后，才会溢写到磁盘。
- 调优建议：如果作业可用的内存资源较为充足的话，可以适当增加这个参数的大小（比如64k），从而减少shuffle write过程中溢写磁盘文件的次数，也就可以减少磁盘IO次数，进而提升性能。在实践中发现，合理调节该参数，性能会有1%~5%的提升。

##### spark.reducer.maxSizeInFlight

- 默认值：48m
- 参数说明：该参数用于设置shuffle read task的buffer缓冲大小，而这个buffer缓冲决定了每次能够拉取多少数据。
- 调优建议：如果作业可用的内存资源较为充足的话，可以适当增加这个参数的大小（比如96m），从而减少拉取数据的次数，也就可以减少网络传输的次数，进而提升性能。在实践中发现，合理调节该参数，性能会有1%~5%的提升。

##### spark.shuffle.io.maxRetries

- 默认值：3
- 参数说明：shuffle read task从shuffle write task所在节点拉取属于自己的数据时，如果因为网络异常导致拉取失败，是会自动进行重试的。该参数就代表了可以重试的最大次数。如果在指定次数之内拉取还是没有成功，就可能会导致作业执行失败。
- 调优建议：对于那些包含了特别耗时的shuffle操作的作业，建议增加重试最大次数（比如60次），以避免由于JVM的full gc或者网络不稳定等因素导致的数据拉取失败。在实践中发现，对于针对超大数据量（数十亿~上百亿）的shuffle过程，调节该参数可以大幅度提升稳定性。

##### spark.shuffle.io.retryWait

- 默认值：5s
- 参数说明：具体解释同上，该参数代表了每次重试拉取数据的等待间隔，默认是5s。
- 调优建议：建议加大间隔时长（比如60s），以增加shuffle操作的稳定性。

##### spark.shuffle.memoryFraction

- 默认值：0.2
- 参数说明：该参数代表了Executor内存中，分配给shuffle read task进行聚合操作的内存比例，默认是20%。
- 调优建议：在资源参数调优中讲解过这个参数。如果内存充足，而且很少使用持久化操作，建议调高这个比例，给shuffle read的聚合操作更多内存，以避免由于内存不足导致聚合过程中频繁读写磁盘。在实践中发现，合理调节该参数可以将性能提升10%左右。

##### spark.shuffle.manager

- 默认值：sort
- 参数说明：该参数用于设置ShuffleManager的类型。Spark 1.5以后，有三个可选项：hash、sort和tungsten-sort。HashShuffleManager是Spark 1.2以前的默认选项，但是Spark 1.2以及之后的版本默认都是SortShuffleManager了。tungsten-sort与sort类似，但是使用了tungsten计划中的堆外内存管理机制，内存使用效率更高。
- 调优建议：由于SortShuffleManager默认会对数据进行排序，因此如果你的业务逻辑中需要该排序机制的话，则使用默认的SortShuffleManager就可以；而如果你的业务逻辑不需要对数据进行排序，那么建议参考后面的几个参数调优，通过bypass机制或优化的HashShuffleManager来避免排序操作，同时提供较好的磁盘读写性能。这里要注意的是，tungsten-sort要慎用，因为之前发现了一些相应的bug。

##### spark.shuffle.sort.bypassMergeThreshold

- 默认值：200
- 参数说明：当ShuffleManager为SortShuffleManager时，如果shuffle read task的数量小于这个阈值（默认是200），则shuffle write过程中不会进行排序操作，而是直接按照未经优化的HashShuffleManager的方式去写数据，但是最后会将每个task产生的所有临时磁盘文件都合并成一个文件，并会创建单独的索引文件。
- 调优建议：当你使用SortShuffleManager时，如果的确不需要排序操作，那么建议将这个参数调大一些，大于shuffle read task的数量。那么此时就会自动启用bypass机制，map-side就不会进行排序了，减少了排序的性能开销。但是这种方式下，依然会产生大量的磁盘文件，因此shuffle write性能有待提高。

##### spark.shuffle.consolidateFiles

- 默认值：false
- 参数说明：如果使用HashShuffleManager，该参数有效。如果设置为true，那么就会开启consolidate机制，会大幅度合并shuffle write的输出文件，对于shuffle read task数量特别多的情况下，这种方法可以极大地减少磁盘IO开销，提升性能。
- 调优建议：如果的确不需要SortShuffleManager的排序机制，那么除了使用bypass机制，还可以尝试将spark.shffle.manager参数手动指定为hash，使用HashShuffleManager，同时开启consolidate机制。在实践中尝试过，发现其性能比开启了bypass机制的SortShuffleManager要高出10%~30%。





## 1.4 [Spark监控](https://spark.apache.org/docs/latest/monitoring.html)

- SparkListener类

```java
// 发送邮件
import javax.mail;
import jedis;  // Redis
```



### 1.4.1 web UI

Every SparkContext launches a [Web UI](https://spark.apache.org/docs/latest/web-ui.html), by default on port 4040, that displays useful information about the application. This includes:

- A list of scheduler stages and tasks
- A summary of RDD sizes and memory usage
- Environmental information.
- Information about the running executors

You can access this interface by simply opening `http://<driver-node>:4040` in a web browser. If multiple SparkContexts are running on the same host, they will bind to successive ports beginning with 4040 (4041, 4042, etc).

Note that this information is only available for the duration of the application by default. To view the web UI after the fact, set `spark.eventLog.enabled` to true before starting the application. This configures Spark to log Spark events that encode the information displayed in the UI to persisted storage.

#### history server

```scala
org.apache.spark.deploy.history.HistoryServer  // 启动类
```

##### Viewing After the Fact

It is still possible to construct the UI of an application through Spark’s history server, provided that the application’s event logs exist. You can **start the history server by executing**:

```shell
./sbin/start-history-server.sh  # 启动服务 配置如下
```

This creates a web interface at `http://<server-url>:18080` by default, listing incomplete and completed applications and attempts.

When using the file-system provider class (see `spark.history.provider` below), the base logging directory must be supplied in the **`spark.history.fs.logDirectory`** (**哪里读日日志**)configuration option, and should contain sub-directories that each represents an application’s event logs.

The spark jobs themselves must be configured to log events, and to log them to the same shared, writable directory. For example, if the server was configured with a log directory of `hdfs://namenode/shared/spark-logs`, then the client-side options would be:

```properties
spark.eventLog.enabled true
spark.eventLog.dir hdfs://namenode/shared/spark-logs  # 日志写在哪里
```



##### 配置history server

- Environment Variables

| Environment Variable     | Meaning                                                      |
| :----------------------- | :----------------------------------------------------------- |
| `SPARK_DAEMON_MEMORY`    | Memory to allocate to the history server (default: 1g).      |
| `SPARK_DAEMON_JAVA_OPTS` | JVM options for the history server (default: none).          |
| `SPARK_DAEMON_CLASSPATH` | Classpath for the history server (default: none).            |
| `SPARK_PUBLIC_DNS`       | The public address for the history server. If this is not set, links to application history may use the internal address of the server, resulting in broken links (default: none). |
| **`SPARK_HISTORY_OPTS`** | `spark.history.*` configuration options for the history server (default: none). |

- [Spark History Server Configuration Options](https://spark.apache.org/docs/latest/monitoring.html#spark-history-server-configuration-options)

- conf/spark-env.sh

```properties
SPARK_HISTORY_OPTS="-Dspark.history.fs.logDirectory=hdfs://hadoop000:8020/sparkEventLog -Dspark.history.ui.port=18000"
```

- conf/spark-defaults.conf

```properties
spark.eventLog.enabled true
spark.eventLog.dir     hdfs://hadoop000:8020/sparkEventLog
```

##### Note

1. The history server displays both completed and incomplete Spark jobs. If an application makes multiple attempts after failures, the failed attempts will be displayed, as well as any ongoing incomplete attempt or the final successful attempt.
2. Incomplete applications are only updated intermittently. The time between updates is defined by the interval between checks for changed files (`spark.history.fs.update.interval`). On larger clusters, the update interval may be set to large values. The way to view a running application is actually to view its own web UI.
3. Applications which exited without registering themselves as completed will be listed as incomplete —even though they are no longer running. This can happen if an application crashes.
4. One way to signal the completion of a Spark job is to stop the Spark Context explicitly (`sc.stop()`), or in Python using the `with SparkContext() as sc:` construct to handle the Spark Context setup and tear down.

#### [REST API](https://spark.apache.org/docs/latest/monitoring.html#rest-api)

In addition to viewing the metrics in the UI, they are also available as JSON. This gives developers an easy way to create new visualizations and monitoring tools for Spark. The JSON is available for both running applications, and in the history server. The endpoints are mounted at `/api/v1`. For example, for the history server, they would typically be accessible at `http://<server-url>:18080/api/v1`, and for a running application, at `http://localhost:4040/api/v1`.

In the API, an application is referenced by its application ID, `[app-id]`. When running on YARN, each application may have multiple attempts, but there are attempt IDs only for applications in cluster mode, not applications in client mode. Applications in YARN cluster mode can be identified by their `[attempt-id]`. In the API listed below, when running in YARN cluster mode, `[app-id]` will actually be `[base-app-id]/[attempt-id]`, where `[base-app-id]` is the YARN application ID.

### 1.4.2 metrics

org.apache.spark.scheduler.SparkListener

```scala
val sparkConf: SparkConf = new SparkConf()
  .setMaster("local[1]")
  .setAppName("testMonitor")
  .set("spark.extraListeners", "org.apache.spark.metrics.DataSparkListener")  // 加载自定义监听器
val sc = new SparkContext(sparkConf)

sc.parallelize(List(1,2,3,4,5)).foreach(println)

sc.stop()
```

- Metrics监听器

```scala
import org.apache.spark.SparkConf
import org.apache.spark.executor.TaskMetrics
import org.apache.spark.internal.Logging
import org.apache.spark.scheduler.{SparkListener, SparkListenerTaskEnd, SparkListenerTaskStart}

import scala.collection.mutable

class DataSparkListener(conf: SparkConf) extends SparkListener with Logging {

    override def onTaskStart(taskStart: SparkListenerTaskStart): Unit = {
        log.warn("=================start======================")
    }

    override def onTaskEnd(taskEnd: SparkListenerTaskEnd): Unit = {
        val appName = conf.get("spark.app.name")
        log.warn("appName: "+appName)

        val metrics: TaskMetrics = taskEnd.taskMetrics
        val taskMetricsMap = mutable.HashMap(
            "executorDeserializeTime" -> metrics.executorDeserializeTime,
            "executorDeserializeCpuTime" -> metrics.executorDeserializeCpuTime,
            "resultSize" -> metrics.resultSize,
        )

        log.warn("metrics: "+taskMetricsMap)
    }
}
```

## 1.5 [Spark优化](https://spark.apache.org/docs/latest/tuning.html)

[Spark性能优化指南——基础篇](https://tech.meituan.com/2016/04/29/spark-tuning-basic.html)

[Spark性能优化指南—高级篇](https://tech.meituan.com/2016/05/12/spark-tuning-pro.html)

### Data Locality

Data locality can have a major impact on the performance of Spark jobs. If data and the code that operates on it are together then computation tends to be fast. But if code and data are separated, one must move to the other. Typically it is faster to ship serialized code from place to place than a chunk of data because code size is much smaller than data. Spark builds its scheduling around this general principle of data locality.

Data locality is how close data is to the code processing it. There are several levels of locality based on the data’s current location. In order from closest to farthest:

- **`PROCESS_LOCAL`** **data is in the same JVM as the running code**. This is the best locality possible（数据与code在同一进程，太理想化）
- **`NODE_LOCAL`** **data is on the same node**. Examples might be in HDFS on the same node, or in another executor on the same node. This is a little slower than `PROCESS_LOCAL` because the data has to travel between processes(性能较佳)
- **`NO_PREF`** data is accessed equally quickly from anywhere and **has no locality preference**
- **`RACK_LOCAL`** **data is on the same rack(机架) of servers**. Data is on a different server on the same rack so needs to be sent over the network, typically through a single switch
- **`ANY`** data is elsewhere on the network and not in the same rack

Spark prefers to schedule all tasks at the best locality level, but this is not always possible. In situations where there is no unprocessed data on any idle executor, Spark switches to lower locality levels. There are two options:

- a) wait until a busy CPU frees up to start a task on data on the same server
- b) immediately start a new task in a farther away place that requires moving data there.

What Spark typically does is wait a bit in the hopes that a busy CPU frees up. Once that timeout expires, it starts moving the data from far away to the free CPU. The wait timeout for fallback between each level can be configured individually or all together in one parameter; see the `spark.locality` parameters on the [configuration page](https://spark.apache.org/docs/latest/configuration.html#scheduling) for details. You should increase these settings if your tasks are long and see poor locality, but the default usually works well.



# 2 环境部署

## 配置

- hive-site.xml软链接（Spark可以访问Hive元数据）

```shell
ln -s ~/app/apache-hive-2.3.8-bin/conf/hive-site.xml ~/app/spark-3.1.1-bin-hadoop2.7/conf/
```

- MySQL jar包(**不要污染原生环境**)

```shell
spark-shell --master local[2] \
--jars ~/software/software/mysql-connector-java-5.1.27-bin.jar \
--driver-class-path ~/software/software/mysql-connector-java-5.1.27-bin.jar
--verbose
```



全局配置信息：spark-default.conf

每次执行添加配置：--conf  推荐（不受全局配置影响）

## [Spark运行模式](https://spark.apache.org/docs/latest/submitting-applications.html)

- local : 本地运行，在开发代码的时候，我们使用该模式进行测试是非常方便的
- standalone : Hadoop部署多个节点的，同理Spark可以部署多个节点  用的不多
- Mesos
- K8S : 2.3版本才正式稍微稳定   是未来比较好的一个方向

> 补充：运行模式和代码没有任何关系，同一份代码可以不做修改运行在不同的运行模式下

### local

```shell
# 运行
# local[2] 并行度为2，不写为1
./spark-submit \
--class  com.imooc.bigdata.chapter02.SparkWordCountAppV2 \
--master local \
--name SparkWordCountAppV2 \
/home/hadoop/lib/sparksql-train-1.0.jar \
hdfs://hadoop000:8020/pk/wc.data hdfs://hadoop000:8020/pk/out
```



```shell
spark-submit \
--master local[4] \
--class xxxx.xx.x \
--conf k=v \
var1 var2
```



### yarn

要将Spark应用程序运行在YARN上，一定要配置HADOOP_CONF_DIR或者YARN_CONF_DIR指向$HADOOP_HOME/etc/conf

启动yarn的server服务

#### client模式

AM：负责申请资源

Driver：调度（可以看到全部日志信息）

```shell
 spark-submit --class org.apache.spark.examples.SparkPi \
 --master yarn \
 ~/app/spark-3.1.1-bin-hadoop2.7/examples/jars/spark-examples*.jar 2
```

#### cluster模式

Driver运行在AM

```shell
 spark-submit --class org.apache.spark.examples.SparkPi \
 --master yarn \
 --deploy-mode cluster \
 ~/app/spark-3.1.1-bin-hadoop2.7/examples/jars/spark-examples*.jar 2
```



### Standalone

* 多个机器，那么你每个机器都需要部署spark(不推荐)

* 相关配置：

```shell
# $SPARK_HOME/conf/slaves
hadoop000

# $SPARK_HOME/conf/spark-env.sh
export SPARK_MASTER_HOST=hadoop000
export JAVA_HOME=/xxxxx
```

- 启动Spark集群 

```shell
$SPARK_HOME/sbin/start-all.sh
# check Master  Worker
jps
```

- 提交作业


```shell
./spark-submit \
--class  com.imooc.bigdata.chapter02.SparkWordCountAppV2 \
--master spark://hadoop000:7077 \
--name SparkWordCountAppV2 \
/home/hadoop/lib/sparksql-train-1.0.jar \
hdfs://hadoop000:8020/pk/wc.data hdfs://hadoop000:8020/pk/out2

# 案例2
spark-submit --master local[2] \
--name SQLApp \
--class com.apache.spark.habse.app.BasicApp \
--jars ~/software/mysql-connector-java-5.1.27-bin.jar \
--driver-class-path ~/software/mysql-connector-java-5.1.27-bin.jar \
/home/hadoop/lib/sparkhbase-1.0-SNAPSHOT.jar
```

> 不管什么运行模式，代码不用改变，只需要在spark-submit脚本提交时, 通过--master xxx 来设置你的运行模式即可



## Maven开发

[编译](http://spark.apache.org/docs/latest/building-spark.html)
[submit](/home/hadoop/app/spark-2.4.3-bin-2.6.0-cdh5.15.1/bin)

- maven3.6.3

```shell
mvn archetype:generate -DarchetypeGroupId=net.alchim31.maven \
-DarchetypeArtifactId=scala-archetype-simple \
-DremoteRepositories=http://scala-tools.org/repo-releases \
-DarchetypeVersion=1.5 \
-DgroupId=com.imooc.bigdata \
-DartifactId=sparksql-train \
-Dversion=1.0
```

* pom.xml
```xml
<properties>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
    <encoding>UTF-8</encoding>
    <scala.tools.version>2.12</scala.tools.version>
    <scala.version>2.12.10</scala.version>
    <spark.version>3.1.1</spark.version>
    <hadoop.version>3.2.2</hadoop.version>
</properties>	

<!--添加CDH的仓库-->
<repositories>
    <repository>
        <id>cloudera</id>
        <url>https://repository.cloudera.com/artifactory/cloudera-repos</url>
    </repository>
</repositories>

<dependency>
  <groupId>org.apache.spark</groupId>
  <artifactId>spark-core_${scala.tools.version}</artifactId>
  <version>${spark.version}</version>
</dependency>
<!--Spark SQL依赖-->
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-sql_${scala.tools.version}</artifactId>
    <version>${spark.version}</version>
</dependency>
<!-- Hadoop相关依赖-->
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-client</artifactId>
    <version>${hadoop.version}</version>
</dependency>
```

- log4j.properties

```properties
log4j.rootLogger=info, stdout
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.target=System.out
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern="%d{yyyy-MM-dd HH:mm:ss} %p [%c:%L] - %m%n
```



### 运行案例

#### 基本运行

```scala
import org.apache.spark.rdd.RDD
import org.apache.spark.{SparkConf, SparkContext}

val conf: SparkConf = new SparkConf().setAppName("SparkWCApp").setMaster("local")
val sc = new SparkContext(conf)

// 将集合元素转为RDD，再进行后续操作
// 方式 1 parallelize
val rdd: RDD[Int] = sc.parallelize(List(1, 2, 3, 4, 5, 6))
rdd.collect()

// 方式 2 makeRDD
sc.makeRDD(List(1,2,3,4,5,6))

sc.stop()
```

#### 文件读取

```scala
val conf: SparkConf = new SparkConf().setAppName("SparkWCApp").setMaster("local")
val sc = new SparkContext(conf)

// 读本地文件
val rdd = sc.textFile("data/wc.data")
rdd.foreach(println)

// 读hdfs文件
sc.hadoopConfiguration.set("dfs.client.use.datanode.hostname", "true")
sc.hadoopConfiguration.set("dfs.replication", "1")
sc.hadoopConfiguration.set("fs.defaultFS", "hdfs://hadoop000:8020")
sc.textFile("hdfs://hadoop000:8020/test.txt").foreach(println)
```



# 3 SparkCore

## 3.1 [RDD](http://people.csail.mit.edu/matei/papers/2012/nsdi_spark.pdf)

Resilient Distributed Dataset 

### 3.1.1 基本特性

- <font color=red>弹性</font>     分布式计算可以容错(节点挂了、数据丢了....)
- <font color=red>分布式</font>   可以运行在多个节点上   并行 
- <font color=red>数据集</font>   集合、文件、Hive Table、HBase Table、RDBMS Table

```txt
RDD ==> 集合就是一个数据集

Represents an immutable,             不可变   RDDA map => RDDB
partitioned collection of elements   集合里面的元素是可以分区/切割  vs HDFS file==>block
that can be operated on in parallel. 并行计算
```

PairRDDFunctions： K->V对

#### 弹性（体现 课程24）

- pipeline
- 如果挂掉，就从老子里面重新跑
  - RDDB挂掉，从A走；RDDC挂掉，从B走
- 血缘关系dependencies(容错)
- checkpoint（容错）数据落地disk



### 3.1.2 RDD的5大特性

- 1）A list of **partitions**

  ```java
  file ==> block
  partition/block  ==> 并行
  
  protected def getPartitions: Array[Partition] // hadooprdd
  ```

- 2）A **function** for computing each split/partition

  ```java
  map  filter
  RDD map ==> RDD里面的每个分区/分片 都作用上map方法
  InputSplit ==> MapTask  
  input ==> N个InputSplit分别交给对应的MapTask去执行
  
  def compute(split: Partition, context: TaskContext): Iterator[T]  // 返回迭代器
  ```

- 3）A list of **dependencies** on other RDDs

  ```java
  RDDA => RDDB ==> RDDC 
  RDDA: 没依赖： 集合数据、测试数据、文本数据
  
  protected def getDependencies: Seq[Dependency[ _ ]] = deps  // 获取依赖
  ```

- 4）Optionally, a **Partitioner** for key-value RDDs (e.g. to say that the RDD is hash-partitioned)

  - key-value类型才有的分区器

  ```scala
  val partitioner: Option[Partitioner] = None
  ```

- 5）Optionally, a list of **preferred locations** to compute each split on (e.g. block locations foran HDFS file)

  - preferred locations  最佳位置
  - 移动计算优于移动数据

  ```java
  protected def getPreferredLocations(split: Partition): Seq[String] = Nil
  ```

  <img src="/Users/andy/Library/Containers/com.tencent.xinWeChat/Data/Library/Application Support/com.tencent.xinWeChat/2.0b4.0.9/d3b9fdb963ba7bb6e480774e95509ff5/Message/MessageTemp/9e20f478899dc29eb19741386f9343c8/Image/511615814017_.pic_hd.jpg" alt="511615814017_.pic_hd" style="zoom:80%;" />



- 案例：一个Partition ==> 一个Task
  - RDD = 5   ==>    5个task就是并行走的
  - Partition是一个逻辑上的概念   想到InputSplit



### 3.1.3 [闭包](https://spark.apache.org/docs/latest/rdd-programming-guide.html#understanding-closures-)

在算子的作用域范围外声明，却在算子作用域内操作和执行

```scala
// 生命周期不一致
var counter = 0  // driver周期
val rdd = sc.parallelize(1 to 5 , 2)
rdd.foreach(x => { // executor周期 count在executor里修改完，driver端获取不到
  counter += x
  println(".........." + counter)
})

println("Counter value: " + counter)
```

- driver端需要序列化到executor端再反序列化,如果connection放在外面，无法进行序列化，报错
  - 解决方案1：对不能序列化的class实现序列化接口
  - 解决方案2：不能序列化，放在executor里面

```scala
sc.parallelize(List(1 to 10 :_*), 2)
.foreachPartition(partition => {
  val connection = MySQLUtils.getConnection()    // Driver must
  println("~~~~~~~~" + connection)

  val sql = "insert into student2(name) values(?)"
  val pstmt = connection.prepareStatement(sql)

  partition.foreach(x => {
    val name = "若泽"+x
    pstmt.setString(1, name)
    pstmt.addBatch()
  })

  pstmt.executeBatch()
  MySQLUtils.closeConnection(connection)
})
```

### 3.1.4 血缘关系(Lineage)

- 血缘关系通过getDependencies获取
- `NarrowDependency` 不需要 shuffle 操作，并且可以用于流式操作（pipeline）。`ShuffleDependency` 则需要进行 shuffle 操作，有 shuffle 的地方需要划分不同的 stage(有shuffle一定是宽依赖) [文章](https://blog.csdn.net/Colton_Null/article/details/112299969?spm=1001.2014.3001.5501)

#### 窄依赖

- 一个父RDD的partition至多被子RDD的某个partition使用一次
- pipeline方式
- 尽可能的使用窄依赖的，尽可能不带shuffle

#### 宽依赖

- 一个父RDD的partition会被子RDD的partition使用多次
- 会有shuffle操作
- UI：至少得有2个stage

![561617021044_.pic](pictures/Spark/血缘关系.jpg)





## 3.2 [算子](https://spark.apache.org/docs/latest/rdd-programming-guide.html#rdd-operations)

- transformations
  - lazy 懒惰
  - 不会真正执行
- actions
  - 执行的
- cache

### 3.2.1 transformations

```scala
val sparkConf: SparkConf = new SparkConf().setMaster("local[2]").setAppName("test")
val sc = new SparkContext(sparkConf)

// 具体的算子

sc.stop()
```

#### 1 map/mapPartitions

- map/mapPartitions区别
  - **mapPartitions** => Return a new RDD by applying a function to **each partition** of this RDD.
  - **map** => Return a new RDD by applying a function to **all elements** of this RDD.

```scala
// map不改变分区数
// 1) 闭包
// 2) MapPartitionsRDD
sc.makeRDD(List(1,2,3,4)).map(x => x * 2).collect()

// 源码 new MapPartitionsRDD[U, T](this, (_, _, iter) => iter.map(cleanF))
// 自定义 MapPartitionsRDD
val rdd  = sc.parallelize(1 to 10)
new MapPartitionsRDD[Int, Int](rdd,
                               (_, _, iterator) => {iterator.map(x => x * 2)}
                              ).foreach(println)
```

- mapPartitions

```scala
sc.parallelize(List(1,2,3,4,5)).mapPartitions(partition=>{
  println("---这是一个分区---")
  partition.map(_*2)
}).foreach(println)
```

#### 2 mapPartitionsWithIndex

```scala
sc.makeRDD(List(1,2,3,4,5,6,7,8,9,10),2)
  .mapPartitionsWithIndex((index, partition) => {
    println("---这是一个分区---")
    partition.map(x => {s"分区:$index, 元素:$x"})
  }).foreach(println)
```

#### 3 flatMap

```scala
// Array(a, a, a, b, b, b)
sc.parallelize(List("a,a,a", "b,b,b")).flatMap(_.split(",")).collect()
// Array(2, 4, 6, 8, 10, 12)
sc.parallelize(List(List(1,2,3),List(4,5,6))).flatMap(_.map(_*2)).collect()
// 一进多出 x => (x, x的平方, x的立方)  Array((1,1,1), (2,4,8), (3,9,27))
sc.parallelize(List(1,2,3)).flatMap(x=>List((x,x*x,x*x*x))).collect()
```

#### 4 filter

```scala
sc.parallelize(List(1,2,3,4,5)).filter(_>3).collect()
sc.parallelize(List(1,2,3,4,5)).filter(_>3).filter(_ % 2 == 0).collect()
sc.parallelize(List(1,2,3,4,5)).filter(x =>{x>3 && x % 2 == 0}).collect()
```

#### 5 zip/zipWithIndex

对接Hbase外部数据源有用

```scala
val names = sc.makeRDD(List("rz", "pk", "xx"))
val ages = sc.makeRDD(List(12, 35, 55))
names.zip(ages).collect()                 // Array((rz,12), (pk,35), (xx,55))
names.zip(ages).zipWithIndex().collect()  // Array(((rz,12),0), ((pk,35),1), ((xx,55),2))
```

#### 6 mapValues/flatMapValues(k-v)

- mapValues

```scala
// Array((a,3), (b,22))
sc.parallelize(List(("a",2),("b",21))).mapValues(_+1).collect()  // 针对value操作，不处理key
```

- flatMapValues

```scala
// Array((a,1), (a,2), (a,3), (b,4), (b,5), (b,6))
sc.parallelize(List(("a","1,2,3"),("b","4,5,6"))).flatMapValues(_.split(",")).collect()
```

#### 7 keys/values(k-v)

```scala
sc.parallelize(List(("a",2),("b",21))).keys.collect()    // keys   Array(a, b)

sc.parallelize(List(("a",2),("b",21))).values.collect()  // values Array(2, 21)
```

#### 8 keyBy(k-v)

```scala
sc.parallelize(List("abc", "ab", "s")).keyBy(_.length).collect()  // Array((3,abc), (2,ab), (1,s))
```

#### 9 reduceBy/groupByKey(k-v)

- 两者底层都用到**combineByKeyWithClassTag**(<font color=blue>查看18</font>)

- shuffle
  - **groupByKey** <font color=red>mapSideCombine = false 不会做本地预聚合</font> 所有数据都走shuffle
  - **reduceByKey** <font color=red>mapSideCombine = true</font> 聚合过后数据走shuffle
  - reduceByKey 性能更好

##### reduceByKey

```scala
/**
* wc:
* 1) 拆分
* 2) (word,1)
* 3) (word,Iterable(1,1,1,...))
* 4) 汇总
* def reduceByKey(func: (V, V) => V)
* 拆分成2个stage: 遇到shuffle就会拆stage
* 底层走combiner  mapSideCombine: Boolean = true 做本地预聚合
* Array((b,2), (a,3), (c,1))
*/
sc.parallelize(List("a,a,a", "b,b", "c")).flatMap(_.split(",")).map((_,1)).reduceByKey(_+_).collect()
```

##### groupByKey

```scala
/**
* groupByKey(): RDD[(K, Iterable[V])]
* Array((b,CompactBuffer(1, 1)), (a,CompactBuffer(1, 1, 1)), (c,CompactBuffer(1)))
* mapSideCombine: Boolean = false  不会做本地预聚合
*/
sc.parallelize(List("a,a,a", "b,b", "c")).flatMap(_.split(",")).map((_,1))
  .groupByKey().mapValues(_.size).collect()
```

> MR的 combiner？本地预聚合?

#### 10 groupBy

map + groupByKey

```scala
// groupBy底层时使用groupByKey, 但是groupByKey是针对KV的，而我们现在的groupBy是针对value
// 所以需要多一个操作: 通过map把单value转成KV的
sc.parallelize(List("a","a","a", "b","b", "c")).groupBy(x => x).mapValues(_.size).collect()
```

```scala
sc.parallelize(1 to 10).groupBy(x => {
  if (x % 2 == 0) "even" else "odd"
}).collect()

// map + groupByKey
sc.parallelize(1 to 10).map(x => {
  (if (x % 2 == 0) "even" else "odd", x)
}).groupByKey.collect()
```

#### 11 sortBy/sortByKey

```scala
sc.parallelize(List(("a", 10), ("b", 20), ("c", 15))).sortBy(-_._2).collect()
```

```scala
// k-v 对key排序
sc.parallelize(List(("a", 10), ("b", 20), ("c", 15))).sortByKey().collect()
```

#### 12 distinct

```scala
// 去重
sc.parallelize(List(1,1,2,3,4,5,6)).distinct().collect()

/**
* 去重原理：
* map：
*    out(key, nullWritable)
* reduce:
*    input(key, Iterable(nullWritable, nullWritable, ....))
*    output: 只需要key就行
*/
sc.parallelize(List(1,1,2,3,4,5,6)).map(x=>(x, null)).reduceByKey((x,_)=>x).map(_._1).collect()
```

#### 13 union/intersection/subtract/cartesian

```scala
val aa = sc.parallelize(List(1, 2, 3, 4, 5))
val bb = sc.parallelize(List(4, 5, 6, 7, 8))
aa.union(bb).collect()         // 并集
aa.intersection(bb).collect()  // 交集
aa.subtract(bb).collect()      // 差集
sc.parallelize(List("AC米兰", "曼联")).cartesian(sc.parallelize(List("胜", "平", "负"))).collect()  // 笛卡尔积
```

#### 14 join

**join**/**leftOuterJoin**/**rightOuterJoin**/**fullOuterJoin**

```scala
val left = sc.parallelize(List(("rz", "北京"), ("jj", "上海"), ("xx1", "深圳")))
val right = sc.parallelize(List(("rz", "10"), ("jj", "20"), ("xx2", "NO")))
// Array((jj,(上海,20)), (rz,(北京,10)))
val res: Array[(String, (String, String))] = left.join(right).collect()
// Array((jj,(上海,Some(20))), (rz,(北京,Some(10))), (xx1,(深圳,None)))
val res2: Array[(String, (String, Option[String]))] = left.leftOuterJoin(right).collect()
// Array((jj,(Some(上海),20)), (xx2,(None,NO)), (rz,(Some(北京),10)))
val res3: Array[(String, (Option[String], String))] = left.rightOuterJoin(right).collect()
// Array((jj,(Some(上海),Some(20))), (xx2,(None,Some(NO))), (rz,(Some(北京),Some(10))), (xx1,(Some(深圳),None)))
val res4: Array[(String, (Option[String], Option[String]))] = left.fullOuterJoin(right).collect()
```

> 底层 cogroup ?

#### 15 coalesce/repartition

- coalesce
  - 默认是调小分区数，如果想要调大分区数，设为true
  - 当指定的分区数 > 原来的, 分区数不变

```scala
// coalesce
sc.parallelize(List(1 to 100 : _*), 5).coalesce(3).collect()
sc.parallelize(List(1 to 100 : _*), 5).coalesce(10,true).collect()
// repartition
sc.parallelize(List(1 to 100 : _*), 5).repartition(6).collect()
```

#### 16 aggregateByKey(高级 pairRDD)

```scala
/**
   * Aggregate the values of each key, using given combine functions and a neutral "zero value".
   * This function can return a different result type, U, than the type of the values in this RDD,
   * V. Thus, we need one operation for merging a V into a U and one operation for merging two U's,
   * as in scala.TraversableOnce. The former operation is used for merging values within a
   * partition, and the latter is used for merging values between partitions. To avoid memory
   * allocation, both of these functions are allowed to modify and return their first argument
   * instead of creating a new U.
   
   * zeroValue 只参与局部计算，不参与全局计算
   */
def aggregateByKey[U: ClassTag](zeroValue: U, partitioner: Partitioner)(seqOp: (U, V) => U,
                                                                        combOp: (U, U) => U): RDD[(K, U)] = self.withScope {}
```

- 案例

```scala
// 根据key：求 max ，最后 sum(max)
val rdd = sc.parallelize(List(("a", 3), ("a", 2), ("b", 6), ("c", 7), ("c", 8)), 2)
// 观察数据分区
rdd.mapPartitionsWithIndex((index, partition)=>{
  partition.map(x=>s"$index, $x")
}).collect()

rdd.aggregateByKey(0)(math.max(_,_), _+_).collect()
```

#### 17 foldByKey(高级)

```scala
/**
   * Merge the values for each key using an associative function and a neutral "zero value" which
   * may be added to the result an arbitrary number of times, and must not change the result
   * (e.g., Nil for list concatenation, 0 for addition, or 1 for multiplication.).
   */
def foldByKey(
  zeroValue: V,
  partitioner: Partitioner)(func: (V, V) => V): RDD[(K, V)] = self.withScope {}
```

- 案例

```scala
// 分区内和分区间一致 简写
sc.parallelize(List((1,2), (3,4), (5,6)), 3).foldByKey(0)(_+_).collect()  // 求和
```

#### 18 combineByKey(高级)

```scala
/**
   * Generic function to combine the elements for each key using a custom set of aggregation
   * functions. This method is here for backward compatibility. It does not provide combiner
   * classtag information to the shuffle.
   *
   * @see `combineByKeyWithClassTag`
   */
def combineByKey[C](
  createCombiner: V => C,             // 确定聚合值的类型 初始值 累加值
  mergeValue: (C, V) => C,            // 分区内聚合
  mergeCombiners: (C, C) => C,        // 全局聚合
  partitioner: Partitioner,
  mapSideCombine: Boolean = true,     // 局部聚合
  serializer: Serializer = null): RDD[(K, C)] = self.withScope {}
```

- 案例

```scala
val rdd = sc.parallelize(List((1,2), (3,4), (5,6)), 3)
rdd.combineByKey(x=>x, (x:Int, y:Int)=>x+y, (a:Int, b:Int)=>a+b).collect()  // 

// 求平均值 = key的总和 / key出现的次数
val rdd = sc.parallelize(List(("a", 3), ("a", 2), ("b", 6), ("c", 7), ("c", 8)), 2)
rdd.combineByKey(
  (_,1),
  (acc:(Int, Int), v) => (acc._1+v, acc._2+1),            // 分区内（总和，次数）
  (x:(Int, Int), y:(Int, Int)) => (x._1+y._1, x._2+y._2)  // 聚合（总和，次数）
).map{ case (key, value) => (key, value._1/value._2.toDouble)}.collect()
```



#### sample(了解)

```scala
sc.parallelize(List(1,2,3,4,5)).sample(false, 0.2).collect()
```

#### glom(了解)

```scala
// Return an RDD created by coalescing all elements within each partition into an array
sc.parallelize(1 to 10, 3).glom().collect()
```



### 3.2.2 action

一般一个action触发一个job，但也有例外

```scala
sc.makeRDD(List(4,3,5,7,2,8,10,9,6,1),6).take(2)  // 触发2个job 因为10个数分成6组，会有一组拿不到2个数

sc.makeRDD(List(4,3,5,7,2,8,10,9,6,1),6)
.mapPartitionsWithIndex((index, partition) => {
  println("-----这是一个分区-----")
  partition.map(x => s"分区是$index,元素是$x")
}).foreach(println)
```

- debug

```scala
sc.parallelize(List(1,2,3,4,5)).map((_,1)).reduceByKey(_+_).toDebugString
```



#### 1 collect

```scala
/**
* Return an array that contains all of the elements in this RDD.
*
* @note This method should only be used if the resulting array is expected to be small, as
* all the data is loaded into the driver's memory.
*/
sc.parallelize(List(1,2,3,4,5)).map(_*2).collect()
```

#### 2 collectAsMap(k-v)

```scala
sc.parallelize(List(1,2,3,4,5)).map(_*2).zipWithIndex().collectAsMap()
```

#### 3 foreach

```scala
sc.parallelize(List(1,2,3,4,5)).map(_*2).foreach(println)  // 类似map
sc.parallelize(List(1,2,3,4,5),3).foreach(println)
```

#### 4 foreachPartition(k-v)

只要遇到写东西到外部存储：foreachPartition

```scala
val rdd = sc.makeRDD(List(1, 2, 3, 4, 5), 3) // partition ==> task
rdd.foreachPartition(partition => {
  println("----------------------")
  for(ele <- partition) {
    println(ele)
  }
})
```

#### 5 take/takeOrdered

```scala
val rdd = sc.makeRDD(List(1, 2, 3, 4, 5))
rdd.take(2)
sc.makeRDD(List(1,20,30,4)).takeOrdered(2)(Ordering.by(x => -x))  // 降序
```

#### 6 reduce/fold

```scala
sc.parallelize(List(1,2,3,4,5)).reduce(_+_)
sc.parallelize(List(1,2,3,4,5)).fold(0)(_+_)  // 0 初始值
```

#### 7 aggregate(高级)

```scala
 /**
   * Aggregate the elements of each partition, and then the results for all the partitions, using
   * given combine functions and a neutral "zero value". This function can return a different result
   * type, U, than the type of this RDD, T. Thus, we need one operation for merging a T into an U
   * and one operation for merging two U's, as in scala.TraversableOnce. Both of these functions are
   * allowed to modify and return their first argument instead of creating a new U to avoid memory
   * allocation.
   *
   * @param zeroValue the initial value for the accumulated result of each partition for the
   *                  `seqOp` operator, and also the initial value for the combine results from
   *                  different partitions for the `combOp` operator - this will typically be the
   *                  neutral element (e.g. `Nil` for list concatenation or `0` for summation)
   * @param seqOp an operator used to accumulate results within a partition
   * @param combOp an associative operator used to combine results from different partitions
   */
  def aggregate[U: ClassTag](zeroValue: U)(seqOp: (U, T) => U, combOp: (U, U) => U): U = withScope {}
```

- 单分区

```scala
def fun1(x:Int, y:Int) = x * y
def fun2(x:Int, y:Int) = x + y

sc.makeRDD(1 to 5, 1).aggregate(3)(fun1, fun2)  // 363

sc.parallelize(List(1,2,3,4,5)).aggregate(0)(_+_, _+_)  // 0 初始值, 第一个_+_分区内操作, 第二个_+_分区间操作
```

- 多分区

```scala
// 观察分区数据
sc.makeRDD(1 to 10, 3).mapPartitionsWithIndex((index, partitions) =>{
  partitions.map(x => s"$index ==> $x")
}).collect()

sc.makeRDD(1 to 10, 3).aggregate(2)(fun1, fun2)  // 10334
```



#### 8 countByKey

```scala
sc.parallelize(List("ruoze","ruoze","ruoze","pk","pk","xingxing")).map((_,1)).countByKey()
```

#### 9 saveAsTextFile

```scala
sc.parallelize(List("ruoze","pk","xingxing")).saveAsTextFile("a.txt", classOf[GzipCodec])
```

#### first/top(了解)



#### 案例

##### 统计

```scala
val rdd = sc.makeRDD(List(1, 2, 3, 4, 5))
rdd.max()
rdd.min()
rdd.count()
rdd.sum()
rdd.mean()

rdd.stdev()
rdd.sampleStdev()
rdd.variance()
rdd.sampleVariance()

rdd.reduce(math.max(_,_))
rdd.reduce((x,y) => if(x>y)x else y)
rdd.reduce(math.min(_,_))
```

##### 读文件

```scala
val lines = sc.textFile("file:///home/hadoop/data/ruozeinput.txt")
val words = lines.flatMap(_.split("\t"))
val pairs = words.map((_,1))
val reduce = pairs.reduceByKey(_+_)
reduce.collect
```

<img src="/Users/andy/Library/Containers/com.tencent.xinWeChat/Data/Library/Application Support/com.tencent.xinWeChat/2.0b4.0.9/d3b9fdb963ba7bb6e480774e95509ff5/Message/MessageTemp/9e20f478899dc29eb19741386f9343c8/Image/571617021045_.pic.jpg" alt="571617021045_.pic" style="zoom:80%;" />

##### 分区内求最大值，分区间求和

```scala
def fun2(x:Int, y:Int) = x + y
def fun3(x:Int, y:List[Int]) = x.max(y.max)

sc.parallelize(List(List(1,2), List(3,4), List(5,6)), 3).aggregate(0)(fun3, fun2)
```

##### 最大值 最小值

```scala
def main(args: Array[String]): Unit = {
  val sparkConf = new SparkConf().setMaster("local").setAppName(this.getClass.getSimpleName)
  val sc = new SparkContext(sparkConf)

  val data = sc.parallelize(List(
    ("A", "A1"),
    ("A", "A2"),
    ("A", "A3"),
    ("B", "B1"),
    ("B", "B2"),
    ("C", "C1"),
  ))

  data.groupByKey().mapValues(_.toList.sorted).map(x => {
    (x._1, firstValueAndLastValue(x._2))
  }).foreach(println)
  
  sc.stop()
}


/**
* @return (key, firstValue, LastValue)
*/
def firstValueAndLastValue(items:List[String]) = {
  for (i <- 0 until (items.length)) yield (items(i), items.head, items(i))
}
```



### 3.2.3 [缓存Cache](https://spark.apache.org/docs/latest/rdd-programming-guide.html#rdd-persistence)	

- 缓存操作是lazy，需要action算子触发
- When you persist an RDD, each node stores any partitions of it that it computes in memory and reuses them in other actions on that dataset (or datasets derived from it).
- You can mark an RDD to be persisted using the `persist()` or `cache()` methods on it. The first time it is computed in an action, it will be kept in memory on the nodes. Spark’s cache is fault-tolerant – if any partition of an RDD is lost, it will automatically be recomputed using the transformations that originally created it.
- **Note:** *In Python, stored objects will always be serialized with the [Pickle](https://docs.python.org/3/library/pickle.html) library, so it does not matter whether you choose a serialized level. The available storage levels in Python include `MEMORY_ONLY`, `MEMORY_ONLY_2`, `MEMORY_AND_DISK`, `MEMORY_AND_DISK_2`, `DISK_ONLY`, `DISK_ONLY_2`, and `DISK_ONLY_3`.*
- Spark also automatically persists some intermediate data in shuffle operations (e.g. `reduceByKey`), even without users calling `persist`. This is done to avoid recomputing the entire input if a node fails during the shuffle. We still recommend users call `persist` on the resulting RDD if they plan to reuse it.

| Storage Level                          | Meaning                                                      |
| :------------------------------------- | :----------------------------------------------------------- |
| MEMORY_ONLY                            | Store RDD as deserialized Java objects in the JVM. If the RDD does not fit in memory, some partitions will not be cached and will be recomputed on the fly each time they're needed. This is the default level. |
| MEMORY_AND_DISK                        | Store RDD as deserialized Java objects in the JVM. If the RDD does not fit in memory, store the partitions that don't fit on disk, and read them from there when they're needed. |
| MEMORY_ONLY_SER (Java and Scala)       | Store RDD as *serialized* Java objects (one byte array per partition). This is generally more space-efficient than deserialized objects, especially when using a [fast serializer](https://spark.apache.org/docs/latest/tuning.html), but more CPU-intensive to read. |
| MEMORY_AND_DISK_SER (Java and Scala)   | Similar to MEMORY_ONLY_SER, but spill partitions that don't fit in memory to disk instead of recomputing them on the fly each time they're needed. |
| DISK_ONLY                              | Store the RDD partitions only on disk.                       |
| MEMORY_ONLY_2, MEMORY_AND_DISK_2, etc. | Same as the levels above, but replicate each partition on two cluster nodes. |
| OFF_HEAP (experimental)                | Similar to MEMORY_ONLY_SER, but store the data in [off-heap memory](https://spark.apache.org/docs/latest/configuration.html#memory-management). This requires off-heap memory to be enabled. |

#### cache 与 persist

- cache()方法表示：使用非序列化的方式将RDD中的数据全部尝试持久化到内存中(cache就是persist)
- persist()方法表示：手动选择持久化级别，并使用指定的方式进行持久化

```scala
// 如果要对一个RDD进行持久化，只要对这个RDD调用cache()和persist()即可。

// 正确的做法。
// cache()方法表示：使用非序列化的方式将RDD中的数据全部尝试持久化到内存中。
// 此时再对rdd1执行两次算子操作时，只有在第一次执行map算子时，才会将这个rdd1从源头处计算一次。
// 第二次执行reduce算子时，就会直接从内存中提取数据进行计算，不会重复计算一个rdd。
val rdd1 = sc.textFile("hdfs://192.168.0.1:9000/hello.txt").cache()
rdd1.map(...)
rdd1.reduce(...)

// persist()方法表示：手动选择持久化级别，并使用指定的方式进行持久化。
// 比如说，StorageLevel.MEMORY_AND_DISK_SER表示，内存充足时优先持久化到内存中，内存不充足时持久化到磁盘文件中。
// 而且其中的_SER后缀表示，使用序列化的方式来保存RDD数据，此时RDD中的每个partition都会序列化成一个大的字节数组，然后再持久化到内存或磁盘中。
// 序列化的方式可以减少持久化的数据对内存/磁盘的占用量，进而避免内存被持久化数据占用过多，从而发生频繁GC。
val rdd1 = sc.textFile("hdfs://192.168.0.1:9000/hello.txt").persist(StorageLevel.MEMORY_AND_DISK_SER)
rdd1.map(...)
rdd1.reduce(...)


rdd.cache()
rdd.persist()
rdd.unpersist() // 清除缓存 eager模式
```

#### Which Storage Level to Choose?

Spark’s storage levels are meant to provide different trade-offs between memory usage and CPU efficiency. We recommend going through the following process to select one:

##### 如何选择一种最合适的持久化策略

- 默认情况下，性能最高的当然是**MEMORY_ONLY**，但前提是你的内存必须足够足够大，可以绰绰有余地存放下整个RDD的所有数据。因为不进行序列化与反序列化操作，就避免了这部分的性能开销；对这个RDD的后续算子操作，都是基于纯内存中的数据的操作，不需要从磁盘文件中读取数据，性能也很高；而且不需要复制一份数据副本，并远程传送到其他节点上。但是这里必须要注意的是，在实际的生产环境中，恐怕能够直接用这种策略的场景还是有限的，如果RDD中数据比较多时（比如几十亿），直接用这种持久化级别，会导致JVM的OOM内存溢出异常。
- 如果使用MEMORY_ONLY级别时发生了内存溢出，那么建议尝试使用**MEMORY_ONLY_SER**级别。该级别会将RDD数据序列化后再保存在内存中，此时每个partition仅仅是一个字节数组而已，大大减少了对象数量，并降低了内存占用。这种级别比MEMORY_ONLY多出来的性能开销，主要就是序列化与反序列化的开销。但是后续算子可以基于纯内存进行操作，因此性能总体还是比较高的。此外，可能发生的问题同上，如果RDD中的数据量过多的话，还是可能会导致OOM内存溢出的异常。
- 如果纯内存的级别都无法使用，那么建议使用**MEMORY_AND_DISK_SER**策略，而不是MEMORY_AND_DISK策略。因为既然到了这一步，就说明RDD的数据量很大，内存无法完全放下。序列化后的数据比较少，可以节省内存和磁盘的空间开销。同时该策略会优先尽量尝试将数据缓存在内存中，内存缓存不下才会写入磁盘。
- 通常不建议使用DISK_ONLY和后缀为_2的级别：因为完全基于磁盘文件进行数据的读写，会导致性能急剧降低，有时还不如重新计算一次所有RDD。后缀为_2的级别，必须将所有数据都复制一份副本，并发送到其他节点上，数据复制以及网络传输会导致较大的性能开销，除非是要求作业的高可用性，否则不建议使用。

```scala
import org.apache.spark.storage.StorageLevel
import org.apache.spark.{SparkConf, SparkContext}

import scala.collection.mutable.ArrayBuffer
import scala.util.Random

/**
 * spark-submit .....  --conf spark.serializer=org.apache.spark.serializer.KryoSerializer
 *
 * $SPARK_HOME/conf/spark-defaults.conf
 * spark.serializer org.apache.spark.serializer.KryoSerializer
 **/
object CacheApp {
  def main(args: Array[String]): Unit = {
    val sparkConf = new SparkConf()
//      .setMaster("local")
//      .setAppName(this.getClass.getCanonicalName)
//      .set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
      .registerKryoClasses(Array(classOf[Info]))

    val sc = new SparkContext(sparkConf)

    val infos = new ArrayBuffer[Info]()
    val names = Array[String]("班长","25号技师","29号技师")
    val genders = Array[String]("女女女", "未知", "男男男")
    val addresses = Array[String]("上海", "山东", "广州")
    1.to(1000000).map(x => {
      val name = names(Random.nextInt(3))
      val age = Random.nextInt(100)
      val gender = genders(Random.nextInt(3))
      val address = addresses(Random.nextInt(3))
      infos .+= (Info(name, age, gender, address))
    })

    val rdd = sc.parallelize(infos)
    rdd.persist(StorageLevel.MEMORY_ONLY_SER)
    println(rdd.count())

    /**
     * 34.3 MiB  rdd.cache()
     * 25.4 MiB  rdd.persist(StorageLevel.MEMORY_ONLY_SER)  Java序列化
     *
     * 69.3 MiB  KryoSerializer rdd.persist(StorageLevel.MEMORY_ONLY_SER) 未注册
     * 31.2 MiB  KryoSerializer rdd.persist(StorageLevel.MEMORY_ONLY_SER) 注册
     */

    Thread.sleep(Int.MaxValue)
    sc.stop()
  }

  case class Info(name:String, age:Int, gender:String, address:String)
}
```



## 3.3 共享变量

Normally, when a function passed to a Spark operation (such as `map` or `reduce`) is executed on a remote cluster node, it works on separate copies of all the variables used in the function. These variables are copied to each machine, and no updates to the variables on the remote machine are propagated back to the driver program. Supporting general, read-write shared variables across tasks would be inefficient. However, **Spark does provide two limited types of *shared variables* for two common usage patterns: broadcast variables and accumulators**.

### 1 Broadcast Variables

Broadcast variables allow the programmer to **keep a read-only variable cached on each machine rather than shipping a copy of it with tasks**. They can be used, for example, to give every node a copy of a large input dataset in an efficient manner. Spark also attempts to distribute broadcast variables using efficient broadcast algorithms to reduce communication cost.

Spark actions are executed through a set of stages, separated by distributed “shuffle” operations. Spark automatically broadcasts the common data needed by tasks within each stage. The data broadcasted this way is cached in serialized form and deserialized before running each task. **This means that explicitly creating broadcast variables is only useful when tasks across multiple stages need the same data or when caching the data in deserialized form is important**.

```scala
/**
* 用1 Broadcast进行join可以减少shuffle，但是只能数据量少的情况
 * 身份证号  名字
 * 110 zs
 * 222 ls
 *
 * 身份证号 学校名字  学号
 * 110 school1  1
 * .........
 *
 * 身份证号 名字 学校名字
 * 110   zs     school1
 */
object BroadcastApp {
def main(args: Array[String]): Unit = {
    val sparkConf = new SparkConf().setMaster("local").setAppName(this.getClass.getCanonicalName)
    val sc = new SparkContext(sparkConf)

    val peopleInfo = sc.parallelize(Array(("110", "zs"), ("222", "ls"))).collectAsMap()
    val peopleBc = sc.broadcast(peopleInfo)

    val peopleDetail = sc.parallelize(Array(
      ("110","school1", 1),
      ("111","school2", 2)
    )).map(x => (x._1, x))

    peopleDetail.mapPartitions(x => {
      val broadcastPeople = peopleBc.value  // 取出广播变量的值

      for((key,value) <- x if broadcastPeople.contains(key))
        yield (key, broadcastPeople.getOrElse(key, ""), value._2)
    }).foreach(println)

    Thread.sleep(Int.MaxValue)
    sc.stop()
  }
}
```



### 2 Accumulators

Accumulators are variables that are only “added” to through an associative and commutative operation and can therefore be efficiently supported in parallel. They can be used to implement counters (as in MapReduce) or sums. Spark natively supports accumulators of numeric types, and programmers can add support for new types.

#### longAccumulator

```scala
val ac = sc.longAccumulator("cnts")
sc.makeRDD(1 to 10, 2).map(x=>{
  if (x % 2 == 0) ac.add(1L)
}).collect()
```

- 自定义累加器

```scala
import org.apache.spark.util.AccumulatorV2

class dataIntAccmulator extends AccumulatorV2[Int, Int]{
  private var _count = 0

  override def value: Int = _count
  override def isZero: Boolean = _count == 0

  // 把当前的累加复制给一个新的累加器
  override def copy(): AccumulatorV2[Int, Int] = {
    val newAcc = new dataIntAccmulator
    newAcc._count = this._count
    newAcc
  }

  override def reset(): Unit = {
    _count = 0
  }

  // 分区内
  override def add(v: Int): Unit =  {
    _count += 1
  }

  // 分区间
  override def merge(other: AccumulatorV2[Int, Int]): Unit = other match {
    case o: dataIntAccmulator =>
      _count += o._count
    case _ =>
      throw new UnsupportedOperationException(
        s"Cannot merge ${this.getClass.getName} with ${other.getClass.getName}")
  }
}
```

```scala
// 执行代码
val sparkConf = new SparkConf().setMaster("local").setAppName(this.getClass.getCanonicalName)
val sc = new SparkContext(sparkConf)

val rdd = sc.parallelize(List(1, 2, 3, 4, 5, 6, 7))
val accmulator = new dataIntAccmulator // new
sc.register(accmulator, "计数器") // 注册我们自己开发的计数器; 没有命名，UI不显示

rdd.map(x => {
  accmulator.add(1)
  x
}).foreach(println)

println(accmulator.value)

sc.stop()
```



#### collectionAccumulator

```scala
case class User(name:String, phone:String)

// 尾号三位相同的
val users = Array(
  User("PK", "13723456789"),
  User("班长", "13812340000"),
  User("一姐", "13712341111"),
  User("字节", "13813812800")
)

val rdd = sc.parallelize(users)

val 尾号三位相同的 = sc.collectionAccumulator[User]("尾号三位相同的")

rdd.foreach(user => {
  val phone = user.phone.reverse
  if(phone(0) == phone(1) && phone(0) == phone(2)) {
    尾号三位相同的.add(user)
  }
})
```

- 自定义累加器-多个变量（sum, count, avg）

```scala
class dataAggAccumulator extends AccumulatorV2[Double, Map[String, Any]]{
  private var map = Map[String, Any]()

  override def isZero: Boolean = map.isEmpty

  override def copy(): AccumulatorV2[Double, Map[String, Any]] = {
    println("-------copy--------")
    val newAcc = new dataAggAccumulator
    newAcc.map = map
    newAcc
  }

  override def reset(): Unit = {
    println("-------reset--------")
    Map[String, Any]()
  }

  override def add(v: Double): Unit = {
    println("-------add--------")
    map += "sum" -> (map.getOrElse("sum",0D).asInstanceOf[Double] + v)
    map += "count" -> (map.getOrElse("count",0L).asInstanceOf[Long] + 1L)
  }

  override def merge(other: AccumulatorV2[Double, Map[String, Any]]): Unit = {
    println("-------merge--------")
    other match {
      case o: dataAggAccumulator =>
        map += "sum" -> (map.getOrElse("sum",0D).asInstanceOf[Double] + o.map.getOrElse("sum",0D).asInstanceOf[Double])
        map += "count" -> (map.getOrElse("count",0L).asInstanceOf[Long] + o.map.getOrElse("count",0L).asInstanceOf[Long])

      case _ =>
        throw new UnsupportedOperationException(
          s"Cannot merge ${this.getClass.getName} with ${other.getClass.getName}")
    }
  }

  override def value: Map[String, Any] = {
    map += "avg" -> (map.getOrElse("sum", 0D).asInstanceOf[Double] / map.getOrElse("count", 0L).asInstanceOf[Long])
    map
  }
}
```

```scala
// 执行代码
val sparkConf = new SparkConf().setMaster("local").setAppName(this.getClass.getCanonicalName)
val sc = new SparkContext(sparkConf)

val rdd = sc.parallelize(1 to 10, 3)
val accmulator = new dataAggAccumulator
sc.register(accmulator, "你很棒")
rdd.foreach(x => accmulator.add(x))

println(accmulator.value)

sc.stop()
```

## 3.4 案例

rdd间不能相互嵌套

collect慎用

### 1 排序

####  case class 与 class方法

```scala
// case class 与 class 的区别
class Products(val name:String, val price:Double, val amount:Int) extends Ordered[Products] with Serializable {
  override def compare(that: Products): Int = {this.amount - that.amount}
}

case class Product2(name:String, price:Double, amount:Int) extends Ordered[Product2] {
  override def compare(that: Product2): Int = this.amount - that.amount
  override def toString: String = s"$name,$price,$amount"
}
```

- main程序

```scala
def main(args: Array[String]): Unit = {
  val sparkConf = new SparkConf().setMaster("local[2]").setAppName(this.getClass.getSimpleName)
  val sc = new SparkContext(sparkConf)

  val products = sc.parallelize(List("aaa 20 10", "bbb 30 1000", "ccc 5 2000", "ddd 25 300", "eee 10 500"),1)
  products.map(x=>{
    val splits = x.split(" ")
    val name = splits(0)
    val price = splits(1).toDouble
    val amount = splits(2).toInt
    Product2(name,price,amount)
  }).sortBy(x=>x).foreach(println)

  sc.stop()
}
```

#### 隐式转换方法

```scala
def main(args: Array[String]): Unit = {
  val sparkConf = new SparkConf().setMaster("local[2]").setAppName(this.getClass.getSimpleName)
  val sc = new SparkContext(sparkConf)

  val products = sc.parallelize(List("aaa 20 10", "bbb 30 1000", "ccc 5 2000", "ddd 25 300", "eee 10 500"),1)
  products.map(x=>{
    val splits = x.split(" ")
    val name = splits(0)
    val price = splits(1).toDouble
    val amount = splits(2).toInt
    new Product2(name,price,amount)
  }).sortBy(x=>x).foreach(println)

  implicit def product2Ordered(product2: Product2):Ordered[Product2] = new Ordered[Product2] {
    override def compare(that: Product2): Int = product2.amount - that.amount
  }

  sc.stop()
}


class Product2(val name:String, val price:Double, val amount:Int) extends Serializable {
  override def toString: String = s"$name,$price,$amount"
}
```

```scala
class Product2(val name:String, val price:Double, val amount:Int) extends Serializable {
  override def toString: String = s"$name,$price,$amount"
}

implicit object Product2Ordering extends Ordering[Product2] {
  override def compare(x: Product2, y: Product2): Int = x.amount-y.amount
}

products.map(x=>{
  val splits = x.split(" ")
  val name = splits(0)
  val price = splits(1).toDouble
  val amount = splits(2).toInt
  new Product2(name,price,amount)
}).sortBy(x=>x).foreach(println)
```

```scala
class Product2(val name:String, val price:Double, val amount:Int) extends Serializable {
  override def toString: String = s"$name,$price,$amount"
}

implicit val Product2Ordering: Ordering[Product2] = new Ordering[Product2]{
  override def compare(x: Product2, y: Product2): Int = x.amount-y.amount
}

products.map(x=>{
  val splits = x.split(" ")
  val name = splits(0)
  val price = splits(1).toDouble
  val amount = splits(2).toInt
  new Product2(name,price,amount)
}).sortBy(x=>x).foreach(println)
```

```scala
val products = sc.parallelize(List("aaa 20 10", "bbb 30 1000", "ccc 5 2000", "ddd 20 300", "eee 10 500"),1)

// （终极方法）价格降序 数量升序
implicit val ord = Ordering[(Double, Int)].on[(String, Double, Int)](x => (-x._2,x._3))

products.map(x=>{
  val splits = x.split(" ")
  val name = splits(0)
  val price = splits(1).toDouble
  val amount = splits(2).toInt
  (name,price,amount)
}).sortBy(x=>x).foreach(println)
```



# 4 SparkSQL

## 4.1 概述

### SQL on Hadoop 框架

| 框架                 | 内容                                                         |
| -------------------- | ------------------------------------------------------------ |
| Apache Hive          | SQL转换成一系列可以在Hadoop上运行的MapReduce（最稳定）/Tez/Spark作业 <br>SQL到底底层是运行在哪种分布式引擎之上的，是可以通过一个参数来设置</br><br>多语言Apache Thrift驱动（可以Python）</br><br>自定义的UDF函数：按照标准接口实现，打包，加载到Hive中元数据（函数有限，须有开发）</br> |
| Cloudera Impala      | 使用了自己的执行守护进程集合，一般情况下这些进程是需要与Hadoop DN安装在一个节点上（网络开销大）<br>Hive支持, 与Hive能够共享元数据, 性能方面是Hive要快速一些，基于内存<br/>命令行、代码 |
| Spark SQL            | 优点：快  与Hive能够兼容<br/> 缺点：执行计划优化完全依赖于Hive  进程 vs 线程<br/> 使用：需要独立维护一个打了补丁的Hive源码分支<br>**Hive on Spark** => 切换Hive的执行引擎即可，底层添加了Spark执行引擎的支持(不推荐,**属于Hive**)<br>**Spark SQL** 对接Hive(**属于Spark**) |
| Presto（京东，美团） | 交互式查询引擎<br>共享元数据信息<br>提供了一系列的连接器，Hive Cassandra...<br>Drill => 支持多种后端存储，然后直接进行各种后端数据的处理 HDFS、Hive、Spark SQL |
| Phoenix              | HBase的数据，是要基于API进行查询<br/> Phoenix使用SQL来查询HBase中的数据<br/> 主要点：如果想查询的快的话，还是取决于ROWKEY的设计 |




### [Spark SQL概述](http://spark.apache.org/sql/)
Spark SQL is Apache Spark's module for working with structured data.

Spark SQL is not about SQL
Spark SQL is about more than SQL

误区一：Spark SQL就是一个SQL处理框架

   * 1）集成性：在Spark编程中无缝对接多种复杂的SQL

* 2）统一的数据访问方式：以类似的方式访问多种不同的数据源，而且可以进行相关操作

  ```scala
  spark.read.format("json").load(path)
  spark.read.format("text").load(path)
  spark.read.format("parquet").load(path)
  spark.read.format("jdbc").load(path)
  spark.read.format("json").option("...","...").load(path)
  ```
- 3）兼容性
  - Spark SQL应用并不局限于SQL
  - 还支持Hive（可以访问metaStore）、JSON、Parquet文件的直接读取以及操作
  - 标准的数据连接：提供标准的JDBC/ODBC连接方式

### SparkSession、SQLContext、HiveContext

- SQLContext是1.x通往SparkSQL的入口
- HiveContext是1.x通往hive入口，HiveContext具有SQLContext的所有功能
- SparkSession是（1.6版本）在Spark 2.0中引入的，它使开发人员可以轻松地使用它，这样我们就不用担心不同的上下文，并简化了对不同上下文的访问。通过访问SparkSession，我们可以自动访问SparkContext（enableHiveSupport）



### Spark SQL框架
* 1 Frontend
    * Hive AST   : SQL语句（字符串）==> 抽象语法树
    * Spark Program : DF/DS API
    * Streaming SQL
* 2 Catalyst
    * Unresolved LogicPlan
      
        ```sql
        select empno, ename from emp
        ```
        
    * Schema Catalog => 和MetaStore 进行逻辑关联
      
    * LogicPlan

    * Optimized LogicPlan（内部优化）

        ```sql
        -- 将我们的SQL作用上很多内置的Rule，使得我们拿到的逻辑执行计划是比较好的
        select * from (select ... from xxx limit 10) limit 5;
        ```

    * Physical Plan



![](pictures/Spark/Catalyst.png)

- Analysis: Transforms an Unresolved Logical Plan to a Resolved Logical Plan
- Unresolved => Resolved: Use Catalog to find where datasets and columns are coming from and types of columns
- Logical Optimization: Transforms a Resolved Logical Plan to an Optimized Logical Plan
- Physical Planning: Transforms a Optimized Logical Plan to a Physical Plan



## 4.2 脚本启动流程分析

### spark-shell启动流程
每个Spark应用程序（spark-shell）在不同目录下启动，其实在该目录下是有metastore_db
* 他们是单独的
* 如果你想spark-shell共享我们的元数据的话，肯定要指定元数据信息==> 后续讲Spark SQL整合Hive的时候讲解

- REPL: Read-Eval-Print Loop  读取-求值-输出(提供给用户即时交互一个命令窗口)

```shell
case $变量名 in
模式1)
  command1
;;
模式2)
  command2
;;
*)
  default
;;
esac

# 知识点
$@  # 传所有外部参数
```




### spark-sql启动流程
* spark-sql底层调用的也是spark-submit
* spark-submit底层调用的是  spark-class
* 因为spark-sql它就是一个Spark应用程序，和spark-shell一样
* 对于你想启动一个Spark应用程序，肯定要借助于spark-submit这脚本进行提交
* spark-sql调用的类是org.apache.spark.sql.hive.thriftserver.SparkSQLCLIDriver
* spark-shell调用的类是org.apache.spark.repl.Main



## 4.3 SparkSQL  API编程

### SparkSession
Spark Core: SparkContext

SparkSQL：2.x之后统一的SparkSession；1.x里面Spark SQL的编程的入口点：SQLContext, HiveContext

### Datasets and DataFrames

A Dataset is a distributed collection of data. **Dataset is a new interface added in Spark 1.6**（之前是SchemaRDD） that provides the benefits of RDDs (strong typing, ability to use powerful lambda functions) with the benefits of Spark SQL’s optimized execution engine. A Dataset can be [constructed](https://spark.apache.org/docs/3.1.1/sql-getting-started.html#creating-datasets) from JVM objects and then manipulated using functional transformations (`map`, `flatMap`, `filter`, etc.). The Dataset API is available in [Scala](https://spark.apache.org/docs/3.1.1/api/scala/org/apache/spark/sql/Dataset.html) and [Java](https://spark.apache.org/docs/3.1.1/api/java/index.html?org/apache/spark/sql/Dataset.html). Python does not have the support for the Dataset API. But due to Python’s dynamic nature, many of the benefits of the Dataset API are already available (i.e. you can access the field of a row by name naturally `row.columnName`). The case for R is similar.

**A DataFrame is a *Dataset* organized into named columns(<font color=red>以列（列名、列类型、列值）的形式构成的分布式数据集</font>). It is conceptually equivalent to a table in a relational database or a data frame in R/Python(关系型数据库一张表), but with richer optimizations under the hood**. DataFrames can be constructed from a wide array of [sources](https://spark.apache.org/docs/3.1.1/sql-data-sources.html) such as: structured data files, tables in Hive, external databases, or existing RDDs. The DataFrame API is available in Scala, Java, [Python](https://spark.apache.org/docs/3.1.1/api/python/pyspark.sql.html#pyspark.sql.DataFrame), and [R](https://spark.apache.org/docs/3.1.1/api/R/index.html). In Scala and Java, a DataFrame is represented by a Dataset of `Row`s. **In [the Scala API](https://spark.apache.org/docs/3.1.1/api/scala/org/apache/spark/sql/Dataset.html), `DataFrame` is simply a type alias of `Dataset[Row]`**. While, in [Java API](https://spark.apache.org/docs/3.1.1/api/java/index.html?org/apache/spark/sql/Dataset.html), users need to use `Dataset<Row>` to represent a `DataFrame`.

> DataFrame = Dataset[Row]
>
> DataFrame 弱类型 untype类型
>
> Dataset   强类型  typed类型



#### DataFrame vs Dataset

```scala
val peopleDF: DataFrame = spark.read.json("data/a.json")
val peopleDS: Dataset[Person] = peopleDF.as[Person]

// 本质区别
peopleDF.select("nme").show()    // 是在运行期报错
peopleDS.map(x => x.nme).show()  // 编译期报错
```



#### DataFrame API
```scala
val spark = SparkSession.builder().appName("DataFrameAPIApp").master("local").getOrCreate()
import spark.implicits._

val df = spark.read.json("data/a.json")

df.printSchema()  // 查看DF内部结构：列名，列的数据类型，是否为空。。
df.show()  // 展示内部数据

// 只要name列
df.select("name").show()
df.select('name).show()
df.select(df("name")).show()
df.select($"name").show()  // 需要隐式转换

spark.stop()
```
```scala
// 过滤
df.filter($"age" > 12).show()
df.filter("age > 12").show()

df.filter(df.col("age") > 15).withColumnRenamed("name", "name2").show(10)
```
```scala
// 聚合
df.groupBy("age").count().show()
```
```scala
// 用SQL方式操作
people.createOrReplaceTempView("people")
spark.sql("select * from people where age > 15").show()
```
```scala
// 取前几条
data.show(10,false)
data.head(3).foreach(println)
data.first()
data.take(5)

// 筛选 排序
student.filter("name ='' or name = 'NULL'").show()
student.orderBy('name, 'id.desc).show()
student.sort()

student.select('id, 'name, 'phone.as("telephone")).show()
```

```scala
// join
val s1 = spark.sparkContext.textFile("data/student.data").map(_.split("\\|")).map(x => Student(x(0).trim, x(1).trim, x(2).trim, x(3).trim)).toDF()
val s2 = spark.sparkContext.textFile("data/student.data").map(_.split("\\|")).map(x => Student(x(0).trim, x(1).trim, x(2).trim, x(3).trim)).toDF()

s1.join(s2, s1("id") === s2("id"), "").show()
```

```scala
import org.apache.spark.sql.functions._
// desc是内置函数
df.select("age", "name").filter(df.col("name") === "b").orderBy(desc("age")).show()
```
````scala
// sql 与 api 2种写法
spark.sql(
  """
        |select
        |*
        |from
        |(
        |select
        |t.*, row_number() over(partition by platform order by cnt desc) as r
        |from
        |(
        |select
        |platform,province, count(1) cnt
        |from
        |access
        |group by
        |platform,province
        |) t
        |) a where a.r <= 3
        |""".stripMargin) //.show()

df.groupBy("platform","province")
.agg(count("*").as("cnt"))
.select('platform,'province, 'cnt, row_number().over(Window.partitionBy("platform").orderBy('cnt.desc))
.as("r")
).filter("r <= 3").show()
````

#### Dataset API

```scala
case class Person(name: String, age: Long)

val spark = SparkSession.builder().master("local").appName("DataFrameAPIApp").getOrCreate()
import spark.implicits._

// 类型转换
val ds: Dataset[Person] = Seq(Person("PK", 30)).toDS()  // 强类型，知道是Person
val res: Dataset[Int] = Seq(1, 2, 3).toDS()

ds.show()
res.map(x=>x+1).collect().foreach(println)

val people: DataFrame = spark.read.json("data/a.json")
val peopleDS: Dataset[Person] = people.as[Person]  // 类型转换
peopleDS.show(false)

spark.stop()
```
#### RDD与DataFrame互转
* 1）uses reflection to infer the schema of an RDD that contains specific types of objects
* 2）creating Datasets is through a programmatic interface that allows you to construct a schema and then apply it to an existing RDD

对于字段比较少的场景，个人倾向于使用第一种
对于字段比较多的场景，个人倾向于使用第二种，自己灵活定制

* 第一种
```scala
/**
* 第一种方式: 反射
* 1）定义 case class
* 2）RDD map， map中每一行数据转成case calss
*/
case class People(name: String, age: Int)

import spark.implicits._
val peopleRDD: RDD[String] = spark.sparkContext.textFile("data/people.txt")

// RDD  => DF
val peopleDF: DataFrame = peopleRDD.map(_.split(",")).map(x => People(x(0), x(1).trim.toInt)) //RDD
.toDF()

peopleDF.createOrReplaceTempView("people")
val queryDF: DataFrame = spark.sql("select name from people limit 1") // DF
queryDF.show()

queryDF.map(x => "name: " + x(0)).show()
queryDF.map(x => "name: " + x.getAs[String]("name")).show() // 通过字段来取
```
* 第二种
	* 编程方式实现的三步曲
		* 1）Create an RDD of Rows from the original RDD;
		* 2）Create the schema represented by a StructType matching the structure of Rows in the RDD created in Step 1.
		* 3）Apply the schema to the RDD of Rows via createDataFrame method provided by SparkSession.
```scala
import org.apache.spark.rdd.RDD
import org.apache.spark.sql.types.{IntegerType, StringType, StructField, StructType}

/**
 * 第二种方式：自定义Schema
 */
val peopleRDD: RDD[String] = spark.sparkContext.textFile("data/people.txt")

// step 1
val peopleRowRDD: RDD[Row] = peopleRDD.map(_.split(",")).map(x => Row(x(0), x(1).trim.toInt))

// step 2
val struct = StructType(
                        StructField("name", StringType, true) ::
                        StructField("age", IntegerType, false) :: Nil
                      )

//step 3
val peopleDF: DataFrame = spark.createDataFrame(peopleRowRDD, struct)

peopleDF.show()
```



## 4.4 UDF

```scala
val spark = SparkSession.builder().master("local").getOrCreate()
import  spark.implicits._
val df = spark.sparkContext.textFile("data/udf.txt")
  .map(_.split("\t"))
  .map(x => FootballTeam(x(0).trim, x(1).trim))
  .toDF

df.createOrReplaceTempView("teams")

/**
 * 1) 定义UDF并注册
 * 2) 使用
 */
val teamsLenUDF = spark.udf.register("teams_len", (input: String) => {
  input.split(",").length
})

spark.sql(
  """
    |
    |select
    |name, teams, teams_len(teams) as len
    |from
    |teams
    |
    |""".stripMargin).show(false)

df.select($"name", $"teams", teamsLenUDF($"teams").as("length")).show(false)
```



```scala
import org.apache.spark.sql.functions._

val df = ds.toDF("province", "city", "district")
df.createOrReplaceTempView("tmp")

val func = (split:String, p1:String, p2:String, p3:String) => {
  p1 + split + p2 + split + p3
}

spark.udf.register("rz_concat_ws", func)

df.select(
  expr("rz_concat_ws('|', province,city,district)")
).show(false)
```

### UDAF

```scala
import org.apache.spark.sql.functions._

val spark = SparkSession.builder().master("local").getOrCreate()
import  spark.implicits._

val rows = new util.ArrayList[Row]()
rows.add(Row("PK", 30, "M"))
rows.add(Row("一姐",18, "F"))
rows.add(Row("班长",80, "M"))

val schema = StructType(
  List(
    StructField("name", StringType, false),
    StructField("age", IntegerType, false),
    StructField("sex", StringType, false)
  )
)

val df = spark.createDataFrame(rows, schema)
df.createOrReplaceTempView("users")

spark.udf.register("avg_udaf", udaf(AvgUDAF))
spark.udf.register("to_double", (column:Any) => column.toString.toDouble)

spark.sql(
  """
    |select
    |sex, avg_udaf(to_double(age)) as avg_age
    |from
    |users
    |group by sex
    |""".stripMargin).show(false)
```

- AvgUDAF

```scala
import org.apache.spark.sql.{Encoder, Encoders}
import org.apache.spark.sql.expressions.Aggregator

object AvgUDAF extends Aggregator[Double, (Double,Long), Double]{
  override def zero: (Double, Long) = (0.0D, 0L)

  override def reduce(b: (Double, Long), a: Double): (Double, Long) = (b._1+a, b._2+1)

  override def merge(b1: (Double, Long), b2: (Double, Long)): (Double, Long) = (b1._1+b2._1, b1._2+b2._2)

  override def finish(reduction: (Double, Long)): Double = reduction._1 / reduction._2

  override def bufferEncoder: Encoder[(Double, Long)] = Encoders.product

  override def outputEncoder: Encoder[Double] = Encoders.scalaDouble
}
```



# 5 Spark Streaming

![](./pictures/Spark/streaming-arch.png)

## [基本概念](https://spark.apache.org/docs/latest/streaming-programming-guide.html#overview)

- 微批处理	
- 接入数据之后，按照指定的间隔进行拆分成batch
- Spark是批处理为主的，实时处理/流处理是批处理的特例
- Flink是实时处理为主的，批处理是实时处理/流处理的特例

![](./pictures/Spark/streaming-flow.png)

- 特性
  - Input DStream is associated with a Receiver (except file stream, discussed later in this section) 
  - continuous stream
  - input data streams
  - applying high-level operations on other DStreams
  - a DStream is represented as a sequence of RDDs.	

| 比较 |                               |                            |
| ---- | ----------------------------- | -------------------------- |
| SS   | discretized stream or DStream | StreamingContext           |
| CORE | RDD                           | SparkContext  SparkSession |
| SQL  | DF/DS                         | SparkSession               |

- pending...
  	job0 为什么一直都在？job0就是接收器
- 至少要2个线程

## Output Operations on DStreams

- Output operations allow DStream’s data to be pushed out to external systems like a database or a file systems. 
- Since the output operations actually allow the transformed data to be consumed by external systems, they trigger the actual execution of all the DStream transformations (similar to actions for RDDs). 

| Output Operation                            | Meaning                                                      |
| :------------------------------------------ | :----------------------------------------------------------- |
| **print**()                                 | Prints the first ten elements of every batch of data in a DStream on the driver node running the streaming application. This is useful for development and debugging. **Python API** This is called **pprint()** in the Python API. |
| **saveAsTextFiles**(*prefix*, [*suffix*])   | Save this DStream's contents as text files. The file name at each batch interval is generated based on *prefix* and *suffix*: *"prefix-TIME_IN_MS[.suffix]"*. |
| **saveAsObjectFiles**(*prefix*, [*suffix*]) | Save this DStream's contents as `SequenceFiles` of serialized Java objects. The file name at each batch interval is generated based on *prefix* and *suffix*: *"prefix-TIME_IN_MS[.suffix]"*. **Python API** This is not available in the Python API. |
| **saveAsHadoopFiles**(*prefix*, [*suffix*]) | Save this DStream's contents as Hadoop files. The file name at each batch interval is generated based on *prefix* and *suffix*: *"prefix-TIME_IN_MS[.suffix]"*. **Python API** This is not available in the Python API. |
| **foreachRDD**(*func*)                      | The most generic output operator that applies a function, *func*, to each RDD generated from the stream. This function should push the data in each RDD to an external system, such as saving the RDD to files, or writing it over the network to a database. Note that the function *func* is executed in the driver process running the streaming application, and will usually have RDD actions in it that will force the computation of the streaming RDDs. |

## Receiver Reliability

There can be two kinds of data sources based on their *reliability*. Sources (like Kafka) allow the transferred data to be acknowledged. If the system receiving data from these *reliable* sources acknowledges the received data correctly, it can be ensured that no data will be lost due to any kind of failure. This leads to two kinds of receivers:

1. *Reliable Receiver* - A *reliable receiver* correctly sends acknowledgment to a reliable source when the data has been received and stored in Spark with replication.
2. *Unreliable Receiver* - An *unreliable receiver* does *not* send acknowledgment to a source. This can be used for sources that do not support acknowledgment, or even for reliable sources when one does not want or need to go into the complexity of acknowledgment.

The details of how to write a reliable receiver are discussed in the [Custom Receiver Guide](https://spark.apache.org/docs/latest/streaming-custom-receivers.html).

### 代码

- CustomReceiver

```scala
import java.io.{BufferedReader, InputStreamReader}
import java.net.Socket
import java.nio.charset.StandardCharsets

import org.apache.spark.internal.Logging
import org.apache.spark.storage.StorageLevel
import org.apache.spark.streaming.receiver.Receiver

class CustomReceiver(host: String, port: Int) 
extends Receiver[String](StorageLevel.MEMORY_AND_DISK_2) with Logging {
    def onStart() {
    // Start the thread that receives data over a connection
    new Thread("Socket Receiver") {
      override def run() { receive() }
    }.start()
  }

  def onStop() {
    // There is nothing much to do as the thread calling receive()
    // is designed to stop by itself if isStopped() returns false
  }

    /** Create a socket connection and receive data until receiver is stopped */
  private def receive() {
    var socket: Socket = null
    var userInput: String = null
    try {
      // Connect to host:port
      socket = new Socket(host, port)

      // Until stopped or connection broken continue reading
      // JavaSE IO 大量的使用了装饰模式
      val reader = new BufferedReader(
        new InputStreamReader(socket.getInputStream(), StandardCharsets.UTF_8))
      userInput = reader.readLine()
      while(!isStopped && userInput != null) {
        store(userInput)
        userInput = reader.readLine()
      }
      reader.close()
      socket.close()

      // Restart in an attempt to connect again when server is active again
      restart("Trying to connect again")
    } catch {
      case e: java.net.ConnectException =>
        // restart if could not connect to server
        restart("Error connecting to " + host + ":" + port, e)
      case t: Throwable =>
        // restart if there is any other error
        restart("Error receiving data", t)
    }
  }
}
```

- main

```scala
val conf: SparkConf = new SparkConf().setAppName(this.getClass.getSimpleName).setMaster("local[2]")
val ssc = new StreamingContext(conf, Seconds(5))

val lines = ssc.receiverStream(new CustomReceiver("hadoop000", 9527))
val result = lines.flatMap(_.split(",")).map((_, 1))

result.foreachRDD(rdd => {
  rdd.foreach(println)
})

ssc.start()
ssc.awaitTermination()
```



## Kafka对接

### 历史版本

- [Spark Streaming 3.X](https://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html) is compatible with Kafka broker versions 0.10 or higher

- [Spark Streaming 2.X](https://spark.apache.org/docs/2.2.0/streaming-kafka-integration.html) is compatible with Kafka broker versions 0.8.2.1 or higher

|                            | [spark-streaming-kafka-0-8](https://spark.apache.org/docs/2.4.6/streaming-kafka-0-8-integration.html) | [spark-streaming-kafka-0-10](https://spark.apache.org/docs/2.4.6/streaming-kafka-0-10-integration.html) |
| :------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| Broker Version             | 0.8.2.1 or higher                                            | 0.10.0 or higher                                             |
| API Maturity               | Deprecated                                                   | Stable                                                       |
| Language Support           | Scala, Java, Python                                          | Scala, Java                                                  |
| Receiver DStream           | Yes（job常驻）                                               | No                                                           |
| Direct DStream             | Yes                                                          | Yes(根据offset取数据)                                        |
| SSL / TLS Support          | No                                                           | Yes                                                          |
| Offset Commit API          | No                                                           | Yes                                                          |
| Dynamic Topic Subscription | No                                                           | Yes                                                          |

### 接收Kafka数据方式

#### 1: Receiver-based Approach

This approach uses a Receiver to receive the data. The Receiver is implemented using the Kafka high-level consumer API. As with all receivers, **the data received from Kafka through a Receiver is stored in Spark executor**s, and then jobs launched by Spark Streaming processes the data.

However, under default configuration, this approach can lose data under failures (see [receiver reliability](https://spark.apache.org/docs/2.4.6/streaming-programming-guide.html#receiver-reliability). To ensure zero-data loss, you have to **additionally enable Write-Ahead Logs in Spark Streaming** (introduced in Spark 1.2). This synchronously （同步）saves all the received Kafka data into write-ahead logs on a distributed file system (e.g HDFS), so that all the data can be recovered on failure. See [Deploying section](https://spark.apache.org/docs/2.4.6/streaming-programming-guide.html#deploying-applications) in the streaming programming guide for more details on Write-Ahead Logs.



Points to remember

- **Topic partitions in Kafka do not correlate to partitions of RDDs generated in Spark Streaming**. So **increasing the number of topic-specific partitions** in the `KafkaUtils.createStream()` **only increases the number of threads** using which topics that are consumed within a single receiver. **It does not increase the parallelism of Spark in processing the data**. Refer to the main document for more information on that.
- Multiple Kafka input DStreams can be created with different groups and topics for parallel receiving of data using multiple receivers.
- If you have enabled **Write-Ahead Logs** with a replicated file system like HDFS, **the received data is already being replicated in the log**. Hence, the storage level in storage level for the input stream to `StorageLevel.MEMORY_AND_DISK_SER` (that is, use `KafkaUtils.createStream(..., StorageLevel.MEMORY_AND_DISK_SER)`).



#### 2: Direct Approach (No Receivers)

This new receiver-less “direct” approach has been introduced in Spark 1.3 to ensure stronger end-to-end guarantees. Instead of using receivers to receive data, this approach **periodically queries Kafka for the latest offsets in each topic+partition, and accordingly defines the offset ranges to process in each batch**. When the jobs to process the data are launched, Kafka’s simple consumer API is used to read the defined ranges of offsets from Kafka (similar to read files from a file system). Note that this **feature was introduced in Spark 1.3 for the Scala and Java API, in Spark 1.4 for the Python API.**

This approach has the following advantages over the receiver-based approach (i.e. Approach 1).

- ***Simplified Parallelism**:* No need to create multiple input Kafka streams and union them. With `directStream`, Spark Streaming will create as many RDD partitions as there are Kafka partitions to consume, which will all read data from Kafka in parallel. So there is a one-to-one mapping between Kafka and RDD partitions, which is easier to understand and tune.
- ***Efficiency**:* Achieving zero-data loss in the first approach required the data to be stored in a Write-Ahead Log, which further replicated the data. This is actually inefficient as the data effectively gets replicated twice - once by Kafka, and a second time by the Write-Ahead Log. This second approach eliminates the problem as there is no receiver, and hence no need for Write-Ahead Logs. **As long as you have sufficient Kafka retention, messages can be recovered from Kafka**.
- ***Exactly-once semantics**:* The first approach uses Kafka’s high-level API to **store consumed offsets in Zookeeper**. This is traditionally the way to consume data from Kafka. While this approach (in combination with-write-ahead logs) can ensure zero data loss (i.e. at-least once semantics), there is a small chance some records may get consumed twice under some failures. **This occurs because of inconsistencies between data reliably received by Spark Streaming and offsets tracked by Zookeeper**. Hence, **in this second approach, we use simple Kafka API that does not use Zookeeper**. Offsets are tracked by Spark Streaming within its checkpoints. This eliminates inconsistencies between Spark Streaming and Zookeeper/Kafka, and so each record is received by Spark Streaming effectively exactly once despite failures. In order to achieve exactly-once semantics for output of your results, your output operation that saves the data to an external data store must be either idempotent, or an atomic transaction that saves results and offsets (see [Semantics of output operations](https://spark.apache.org/docs/2.4.6/streaming-programming-guide.html#semantics-of-output-operations) in the main programming guide for further information).

### API策略

#### LocationStrategies

The new Kafka consumer API will pre-fetch messages into buffers. Therefore it is important for performance reasons that the Spark integration keep cached consumers on executors (rather than recreating them for each batch), and prefer to schedule partitions on the host locations that have the appropriate consumers.

In most cases, you should use `LocationStrategies.PreferConsistent` as shown above. This will distribute partitions evenly across available executors. If your executors are on the same hosts as your Kafka brokers, use `PreferBrokers`, which will prefer to schedule partitions on the Kafka leader for that partition. Finally, if you have a significant skew in load among partitions, use `PreferFixed`. This allows you to specify an explicit mapping of partitions to hosts (any unspecified partitions will use a consistent location).

**The cache for consumers has a default maximum size of 64.** If you expect to be handling more than (64 * number of executors) Kafka partitions, you can change this setting via `spark.streaming.kafka.consumer.cache.maxCapacity`.

If you would like to disable the caching for Kafka consumers, you can set `spark.streaming.kafka.consumer.cache.enabled` to `false`.

The cache is keyed by topicpartition and group.id, so use a **separate** `group.id` for each call to `createDirectStream`.

#### ConsumerStrategies

The new Kafka consumer API has a number of different ways to specify topics, some of which require considerable post-object-instantiation setup. `ConsumerStrategies` provides an abstraction that allows Spark to obtain properly configured consumers even after restart from checkpoint.

`ConsumerStrategies.Subscribe`, as shown above, allows you to subscribe to a fixed collection of topics. `SubscribePattern` allows you to use a regex to specify topics of interest. Note that unlike the 0.8 integration, using `Subscribe` or `SubscribePattern` should respond to adding partitions during a running stream. Finally, `Assign` allows you to specify a fixed collection of partitions. All three strategies have overloaded constructors that allow you to specify the starting offset for a particular partition.

If you have specific consumer setup needs that are not met by the options above, `ConsumerStrategy` is a public class that you can extend.

### 存储Offsets

Kafka delivery semantics in the case of failure depend on how and when offsets are stored. Spark output operations are [at-least-once](https://spark.apache.org/docs/2.4.6/streaming-programming-guide.html#semantics-of-output-operations). So if you want the equivalent of exactly-once semantics, you must either store offsets after an idempotent output, or store offsets in an atomic transaction alongside output. With this integration, you have 3 options, in order of increasing reliability (and code complexity), for how to store offsets.

#### Checkpoints

If you enable Spark [checkpointing](https://spark.apache.org/docs/2.4.6/streaming-programming-guide.html#checkpointing), offsets will be stored in the checkpoint. This is easy to enable, but there are drawbacks. Your output operation must be idempotent, since you will get repeated outputs; transactions are not an option. Furthermore, you cannot recover from a checkpoint if your application code has changed. For planned upgrades, you can mitigate this by running the new code at the same time as the old code (since outputs need to be idempotent anyway, they should not clash). But for unplanned failures that require code changes, you will lose data unless you have another way to identify known good starting offsets.

#### Kafka itself

Kafka has an offset commit API that stores offsets in a special Kafka topic. By default, the new consumer will periodically auto-commit offsets. This is almost certainly not what you want, because messages successfully polled by the consumer may not yet have resulted in a Spark output operation, resulting in undefined semantics. This is why the stream example above sets “enable.auto.commit” to false. However, you can commit offsets to Kafka after you know your output has been stored, using the `commitAsync` API. The benefit as compared to checkpoints is that Kafka is a durable store regardless of changes to your application code. However, Kafka is not transactional, so your outputs must still be idempotent（幂等性）.

#### Your own data store

For data stores that support transactions, saving offsets in the same transaction as the results can keep the two in sync, even in failure situations. If you’re careful about detecting repeated or skipped offset ranges, rolling back the transaction prevents duplicated or lost messages from affecting results. This gives the equivalent of exactly-once semantics. It is also possible to use this tactic even for outputs that result from aggregations, which are typically hard to make idempotent.

```scala
// The details depend on your data store, but the general idea looks like this

// begin from the the offsets committed to the database
val fromOffsets = selectOffsetsFromYourDatabase.map { resultSet =>
  new TopicPartition(resultSet.string("topic"), resultSet.int("partition")) -> resultSet.long("offset")
}.toMap

val stream = KafkaUtils.createDirectStream[String, String](
  streamingContext,
  PreferConsistent,
  Assign[String, String](fromOffsets.keys.toList, kafkaParams, fromOffsets)
)

stream.foreachRDD { rdd =>
  val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges

  val results = yourCalculation(rdd)

  // begin your transaction

  // update results
  // update offsets where the end of existing offsets matches the beginning of this batch of offsets
  // assert that offsets were updated correctly

  // end your transaction
}
```



##### 设计与代码（48-SS增强02-B）

###### 核心逻辑

- SS对接Kafka的数据

  - 针对offset，需要从某个地方获取已经消费过的offset

  ```scala
  val offset = new mutable.HashMap[TopicPartition,Long]()
  val stream = KafkaUtils.createDirectStream[String, String](
  	ssc,
    PreferConsistent,
    Subscribe[String, String](topics, kafkaParams, offset)
  )
  ```

- 业务逻辑处理（事务性保障）

  - 聚合类：数据量小，把聚合结果拉到Driver
  - 非聚合类：在executor中执行

- 保存结果，提交offset（保证exactly-once 需要结果和offset幂等）

  - 聚合类：Driver：data+offset进行存储
  - 非聚合类：executor：data+offset进行存储

- data参考选型：
  - MySQL/Redis:事务
  - HBase：行级事务，表中设计2个cf，一个存data，一个offset
  - 建议同一类型数据库，不然无法保证事务



> 如何保证SS+Kafka的exactly-once
>
> 1 input ： offset
>
> 2 transformation ： ok
>
> 3 output：幂等 或 事务

###### 聚合 

事务保障表设计

- topic
- group_id
- partition
- offset
- 联合主键（topic, group_id, partition）

###### 非聚合

- ETL逻辑：data=>Kafka=>SS=>NoSQL(HBase/ES/..)
- 在executor端完成
  - data落到NoSQL
  - offset也落到NoSQL（与data同一类型数据库）
  - 由于HBase保证行级别事务，hbase表：o表示数据cf，offset表示offset的cf
- 代码逻辑
  - executor端获取offset+data写入HBase（只要记录partition内的最后一条记录存offset）
  - 还需从HBase中获取已经有的offset

### 代码

- 1

```scala
import org.apache.kafka.common.serialization.StringDeserializer
import org.apache.spark.SparkConf
import org.apache.spark.streaming.kafka010.{CanCommitOffsets, HasOffsetRanges, KafkaUtils}
import org.apache.spark.streaming.kafka010.ConsumerStrategies.Subscribe
import org.apache.spark.streaming.kafka010.LocationStrategies.PreferConsistent
import org.apache.spark.streaming.{Seconds, StreamingContext}

object OffsetApp {
    def main(args: Array[String]): Unit = {
        val conf: SparkConf = new SparkConf().setAppName(this.getClass.getSimpleName).setMaster("local[2]")
          .set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")  // 应对序列化
        val ssc = new StreamingContext(conf, Seconds(5))

        val kafkaParams = Map[String, Object](
          "bootstrap.servers" -> "hadoop000:9092",
          "key.deserializer" -> classOf[StringDeserializer],
          "value.deserializer" -> classOf[StringDeserializer],
          "group.id" -> "testGroup",
          "auto.offset.reset" -> "earliest",
          "enable.auto.commit" -> (false: java.lang.Boolean)
        )

        val topics = Array("test01")
        // KafkaUtils.createDirectStream拿到的是一手Stream
        val stream = KafkaUtils.createDirectStream[String, String](
          ssc,
          PreferConsistent,
          Subscribe[String, String](topics, kafkaParams)
        )  // 不能 .map(_.value()) 这样不是一手的Stream

        stream.foreachRDD(rdd => {
            if (!rdd.isEmpty()) {
                // 获取offset 这个RDD必须是KafkaRDD
                // Driver端执行
                val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
                offsetRanges.foreach(x => {
                    println(s"${x.topic}, ${x.partition}, ${x.fromOffset}, ${x.untilOffset}")
                })

                // 业务逻辑
                // executor端执行
                rdd.flatMap(_.value().split(",")).map((_,1)).reduceByKey(_+_).foreach(println)

                // 提交offset
                // Driver端执行
                stream.asInstanceOf[CanCommitOffsets].commitAsync(offsetRanges) // 无法保证幂等性

            } else {
                println("该批次没有数据")
            }
        })

        ssc.start()
        ssc.awaitTermination()
    }
}
```

- 2

```scala
import org.apache.kafka.clients.consumer.ConsumerRecord
import org.apache.kafka.common.serialization.StringDeserializer
import org.apache.spark.SparkConf
import org.apache.spark.streaming.dstream.DStream
import org.apache.spark.streaming.kafka010.ConsumerStrategies.Subscribe
import org.apache.spark.streaming.kafka010.LocationStrategies.PreferConsistent
import org.apache.spark.streaming.kafka010.{CanCommitOffsets, HasOffsetRanges, KafkaUtils, OffsetRange}
import org.apache.spark.streaming.{Seconds, StreamingContext}

object OffsetApp2 {
    def main(args: Array[String]): Unit = {
        val conf: SparkConf = new SparkConf().setAppName(this.getClass.getSimpleName).setMaster("local[2]")
          .set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")  // 应对序列化
        val ssc = new StreamingContext(conf, Seconds(5))
        ssc.checkpoint("data/offset")

        val kafkaParams = Map[String, Object](
           "bootstrap.servers" -> "hadoop000:9092",
          "key.deserializer" -> classOf[StringDeserializer],
          "value.deserializer" -> classOf[StringDeserializer],
          "group.id" -> "testGroup2",
          "auto.offset.reset" -> "earliest",
          "enable.auto.commit" -> (false: java.lang.Boolean)
        )

        val topics = Array("test01")

        // KafkaUtils.createDirectStream拿到的是一手Stream
        val stream = KafkaUtils.createDirectStream[String, String](
          ssc,
          PreferConsistent,
          Subscribe[String, String](topics, kafkaParams)
        )

        // 通过transform 可以使用二手DStream
        var offsetRanges: Array[OffsetRange] = null
        val transformDstream = stream.transform(rdd => {
            offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
            offsetRanges.foreach(x => {
                    println(s"${x.topic}, ${x.partition}, ${x.fromOffset}, ${x.untilOffset}")
                })

            rdd
        })

        val result = transformDstream.flatMap(_.value().split(",")).map((_, 1)).updateStateByKey(updateFunction)

        result.foreachRDD(rdd => {
            if (!rdd.isEmpty()) {
                // 获取offset 这个RDD必须是KafkaRDD
                // Driver端执行

                // 业务逻辑
                // executor端执行
                rdd.foreach(println)

                // 提交offset
                // Driver端执行
                stream.asInstanceOf[CanCommitOffsets].commitAsync(offsetRanges) // 无法保证幂等性

            } else {
                println("该批次没有数据")
            }
        })


        ssc.start()
        ssc.awaitTermination()
    }

    def updateFunction(newValues: Seq[Int], preValues: Option[Int]) = {
        val newCount = newValues.sum
        val pre = preValues.getOrElse(0)
        Some(newCount + pre)
    }
}
```



#### 案例 有状态 外部存储

- redis

```scala
/**
 * 使用类似数据库连接池的思想构建redis的连接池
 */
public class JedisUtil {

    private static JedisPool pool;

    private JedisUtil(){}

    static {
        JedisPoolConfig config = new JedisPoolConfig();
        pool = new JedisPool(config, "hadoop000", 6379);
    }

    // 与redis建立连接相当于redis的连接池
    public static Jedis getJedis() {
        pool.getResource().select(3);
        return pool.getResource();
    }

    public static void returnJedis(Jedis jedis) {
        pool.returnResource(jedis);
        jedis.close();
    }

    public static void main(String[] args) {
        Jedis jedis = JedisUtil.getJedis();
        JedisUtil.returnJedis(jedis);
    }
}
```

- trait

```scala
import org.apache.kafka.common.TopicPartition
import org.apache.spark.streaming.kafka010.OffsetRange

import scala.collection.mutable

trait OffsetsManager {

    def obtainOffsets(topics:Array[String], groupId:String): mutable.HashMap[TopicPartition, Long]

    def storeOffsets(groupId:String, offsetsRanges:Array[OffsetRange])
}
```

- redis 管理应用

```scala
package com.apache.spark.habse.unique

import org.apache.kafka.common.TopicPartition
import org.apache.spark.streaming.kafka010.OffsetRange

import scala.collection.mutable

class RedisOffsetsManager extends OffsetsManager with Serializable {
        override def obtainOffsets(topics: Array[String], groupId: String): mutable.HashMap[TopicPartition, Long] = {
        //mutable可变的声明map为可变的默认map不可变
        val fromOffset = mutable.HashMap[TopicPartition, Long]()
        import scala.collection.JavaConversions._
        val jedis = JedisUtil.getJedis
        //遍历topics
        for(topic <- topics) {
            //以列表形式返回哈希表的域和域的值。
            // 若 key 不存在，返回空列表。

            val map = jedis.hgetAll(topic)
            for((field, value) <- map) {//field=group|partition
                //获取分区
                val partition = field.substring(field.indexOf("|") + 1).toInt
                //获取偏移量
                val offset = value.toLong
                //将topic和partition和偏移量放入分区
               fromOffset.put(new TopicPartition(topic, partition), offset)
            }
        }
        //将redis放回连接池
        JedisUtil.returnJedis(jedis)
        fromOffset
    }

    override def storeOffsets(groupId: String, offsetsRanges: Array[OffsetRange]) = {
        //获取redis连接池连接redis
        val jedis = JedisUtil.getJedis
        for (offsetRange <- offsetsRanges) {
            val topic = offsetRange.topic
            val partition = offsetRange.partition
            val offset = offsetRange.untilOffset
            val field = s"${groupId}|${partition}"
            jedis.hset(topic, field, offset.toString)
        }
        //将redis送回redis的连接池
        JedisUtil.returnJedis(jedis)
    }
}
```

- 主程序

```scala
package com.apache.spark.habse.unique

import java.util.Date

import com.alibaba.fastjson.JSON
import org.apache.commons.lang3.time.FastDateFormat
import org.apache.kafka.clients.consumer.ConsumerRecord
import org.apache.kafka.common.TopicPartition
import org.apache.kafka.common.serialization.StringDeserializer
import org.apache.spark.SparkConf
import org.apache.spark.streaming.dstream.InputDStream
import org.apache.spark.streaming.kafka010.ConsumerStrategies.Subscribe
import org.apache.spark.streaming.kafka010.{HasOffsetRanges, KafkaUtils, OffsetRange}
import org.apache.spark.streaming.kafka010.LocationStrategies.PreferConsistent
import org.apache.spark.streaming.{Seconds, StreamingContext}

import scala.collection.mutable
import scala.collection.mutable.ListBuffer
import scala.util.Random

/*
    kafka-0.10.1.X版本之前: auto.offset.reset 的值为smallest,和,largest.(offest保存在zk中)
    kafka-0.10.1.X版本之后: auto.offset.reset 的值更改为:earliest,latest,和none
        (offest保存在kafka的一个特殊的topic名为:__consumer_offsets里面)
    如果存在已经提交的offest时,不管设置为earliest 或者latest
    都会从已经提交的offest处开始消费如果不存在已经提交的offest时,earliest 表示从头开始消费,
    latest 表示从最新的数据消费,也就是新产生的数据.
    none topic各分区都存在已提交的offset时，从提交的offest处开始消费；只要有一个分区不存在已提交的offset，则抛出异常
 */
object OffsetApp {
    def main(args: Array[String]): Unit = {

        val provinces = List(
            "浙江省",
            "广东省",
            "江苏省",
            "河北省",
            "上海市",
        )

        val conf = new SparkConf()
          .setMaster("local[2]")
          .setAppName("test111")
          .set("spark.serializer", "org.apache.spark.serializer.KryoSerializer") // 如果出现序列化问题

        val ssc = new StreamingContext(conf, Seconds(5))
        ssc.checkpoint("Offset002")

        val groupId = "test01"
        val kafkaParams = Map[String, Object](
  "bootstrap.servers" -> "hadoop000:9092,hadoop000:9093",
          "key.deserializer" -> classOf[StringDeserializer],
          "value.deserializer" -> classOf[StringDeserializer],
          "group.id" -> groupId,
          "auto.offset.reset" -> "earliest",
          "enable.auto.commit" -> "false"
        )
        val topics = Array("rzdata10")

        val manager = new RedisOffsetsManager()
        val offsets: mutable.HashMap[TopicPartition, Long] = manager.obtainOffsets(topics, groupId)

        val stream = if (offsets != null && offsets.nonEmpty) {
            //有偏移量获取偏移量根据偏移量读数据
            KafkaUtils.createDirectStream(ssc, PreferConsistent, Subscribe[String, String](topics, kafkaParams, offsets))
        } else {
            // 第一次没有
            KafkaUtils.createDirectStream[String, String](ssc, PreferConsistent, Subscribe[String, String](topics, kafkaParams))
        }

        // 拿到本批次要的Kafka数据
        var offsetRanges: Array[OffsetRange] = null
        val transformStream = stream.transform(rdd => {
            offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
            rdd
        })

        val mapStream = transformStream.map(x=>{
            val json = JSON.parseObject(x.value())
            val uid = json.getString("uid")
            val time = json.getLong("time")
            val ip = json.getString("ip")
            val province  = provinces(Random.nextInt(provinces.length))

            val tmp = FastDateFormat.getInstance("yyyy-MM-dd HH").format(new Date()).split(" ")
            val day = tmp(0).replace("-", "")
            val hour = tmp(1)
            AccessInfo(uid, time, ip, province, day, hour)
        })

        val filterStream = mapStream.mapPartitions(partition => {
            val jedis = JedisUtil.getJedis
            val list = new ListBuffer[AccessInfo]
            for (ele <- partition) {
                val key = "daily_user_test: " + ele.day
                val isAdd = jedis.sadd(key, ele.uid)
                if (isAdd == 1){
                    list.append(ele)
                }
            }

            //将redis放回连接池
            JedisUtil.returnJedis(jedis)
            list.toIterator
        })

        // 把满足条件数据存储
        filterStream.foreachRDD(rdd => {
            rdd.foreachPartition(partition => {
                partition.map(x=>{

                })
            })

            manager.storeOffsets(groupId, offsetRanges)
        })

        ssc.start()
        ssc.awaitTermination()

    }
}

case class AccessInfo(uid:String, time:Long, ip:String, province:String, day:String, hour:String)
```

- 造数据

```scala
import java.util.Properties
import com.alibaba.fastjson.JSONObject
import org.apache.kafka.clients.producer.{KafkaProducer, ProducerRecord}
import scala.util.Random

object MockData {
    def main(args: Array[String]): Unit = {

         val properties = new Properties()
        properties.setProperty("bootstrap.servers", "hadoop000:9092,hadoop000:9093")
        properties.setProperty("group.id", "test01")
        properties.setProperty("auto.offset.reset", "latest")
        properties.setProperty("enable.auto.commit", "false")
        properties.setProperty("acks", "1")
        properties.setProperty("key.serializer", "org.apache.kafka.common.serialization.StringSerializer")
        properties.setProperty("value.serializer", "org.apache.kafka.common.serialization.StringSerializer")
        val producer = new KafkaProducer[String, String](properties)

        val ip = List(
            "223.33.11.111",
            "224.33.11.111",
            "225.33.11.111",
            "226.33.11.111",
            "227.33.11.111",
            "229.33.11.111",
            "223.34.11.111",
            "223.36.11.111",
            "223.37.112.111",
            "223.33.118.111",
            "223.33.113.111",
        )

        val channels = List(
            "ping",
            "baidu",
            "google",
            "alibaba",
            "apple",
        )
        for (i <- 1 to 100) {
            val uid = Random.nextInt(30) + 1
            val time = System.currentTimeMillis()
            Thread.sleep(100)

            val log = new JSONObject()
            log.put("id", i)
            log.put("uid", uid)
            log.put("ip", ip(Random.nextInt(ip.length)))
            log.put("channel", channels(Random.nextInt(channels.length)))
            log.put("time", time)

            //println(log)
            val part = i % 2 // 取分区号
            val value = new ProducerRecord[String, String]("rzdata10", part, "", log.toJSONString)
            producer.send(value)
        }
    }
}
```



## 案例

### Points to remember

- Once a context has been started, no new streaming computations can be set up or added to it.
- Once a context has been stopped, it cannot be restarted.
- Only one StreamingContext can be active in a JVM at the same time.
- stop() on StreamingContext also stops the SparkContext. To stop only the StreamingContext, set the optional parameter of `stop()` called `stopSparkContext` to false.
- A SparkContext can be re-used to create multiple StreamingContexts, as long as the previous StreamingContext is stopped (without stopping the SparkContext) before the next StreamingContext is created.

```xml
<dependency>
  <groupId>org.apache.spark</groupId>
  <artifactId>spark-streaming_${scala.tools.version}</artifactId>
</dependency>
```

- log4j.properties

```properties
log4j.rootLogger = error, stdout
log4j.appender.stdout = org.apache.log4j.ConsoleAppender
log4j.appender.stdout.target = System.out
log4j.appender.stdout.layout = org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern = "%d{yyyy-MM-dd HH:mm:ss} %p [%c:%L] - %m%n
```

- 例子1

```scala
import org.apache.spark.SparkConf
import org.apache.spark.streaming.dstream.{DStream, ReceiverInputDStream}
import org.apache.spark.streaming.{Seconds, StreamingContext}

val sparkConf = new SparkConf().setAppName(this.getClass.getCanonicalName).setMaster("local[2]")
val ssc = new StreamingContext(sparkConf, Seconds(5))

/**
* 1 对接数据源
* ReceiverInputDStream extends InputDStream extends DStream
*/
val lines: ReceiverInputDStream[String] = ssc.socketTextStream("hadoop000", 9527)

/**
* 2 业务逻辑处理
*/
val result = lines.flatMap(_.split(",")).countByValue()

/**
* 3 输出  dstream:output   rdd:action
*/
result.print()

ssc.start()
// result....
ssc.awaitTermination()
```

- 例子-整合SparkSQL

```scala
val sparkConf = new SparkConf().setAppName(this.getClass.getCanonicalName).setMaster("local[2]")
val ssc = new StreamingContext(sparkConf, Seconds(5))

val lines = ssc.socketTextStream("hadoop000", 9527)
val words = lines.flatMap(_.split(","))
words.foreachRDD(rdd => {
  // Get the singleton instance of SparkSession
  // 不要new 是一体的
  val spark = SparkSession.builder.config(rdd.sparkContext.getConf).getOrCreate()
  import spark.implicits._

  // Convert RDD[String] to DataFrame
  val wordsDataFrame = rdd.toDF("word")

  // Create a temporary view
  wordsDataFrame.createOrReplaceTempView("words")

  // Do word count on DataFrame using SQL and print it
  val wordCountsDataFrame = spark.sql("select word, count(*) as total from words group by word")
  wordCountsDataFrame.show()
})

ssc.start()
ssc.awaitTermination()
```

- 例子-DStream整合RDD

```scala
import org.apache.spark.SparkConf
import org.apache.spark.streaming.dstream.{DStream, ReceiverInputDStream}
import org.apache.spark.streaming.{Seconds, StreamingContext}

import scala.collection.mutable.ListBuffer

object TransformApp {
  def main(args: Array[String]): Unit = {
    val sparkConf = new SparkConf()
      .setAppName(this.getClass.getCanonicalName).setMaster("local[2]")
    val ssc = new StreamingContext(sparkConf, Seconds(5))

    val blacks = List("ruoze")
    val blackRDD = ssc.sparkContext.parallelize(blacks).map((_, true))

    /**
     * 现在SS基于DStream进行编程，大部分都是面向DStream API进行开发的
     *
     * DStream如何整合RDD进行开发
     */
     // 接收access数据
    val lines = ssc.socketTextStream("hadoop000", 9527)
    lines.map(x => (x.split(",")(1), x))
        .transform(rdd => {
          rdd.leftOuterJoin(blackRDD).filter(x => {
              !x._2._2.getOrElse(false)
            }).map(x => x._2._1)
        }).print()

    ssc.start()
    ssc.awaitTermination()
  }
}
```



### **foreachRDD**

- pom.xml

```xml
 <dependency>
   <groupId>mysql</groupId>
   <artifactId>mysql-connector-java</artifactId>
   <version>5.1.47</version>
</dependency>

<dependency>
	<groupId>c3p0</groupId>
	<artifactId>c3p0</artifactId>
	<version>0.9.1.2</version>
</dependency>
```



- ConnectionPool

```scala
import com.mchange.v2.c3p0.ComboPooledDataSource;

import java.sql.Connection;
import java.sql.SQLException;

public class ConnectionPool {
       private static ComboPooledDataSource dataSource = new ComboPooledDataSource();
    static {
        dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/test");
        dataSource.setUser("root");
        dataSource.setPassword("xxxx@123");
        dataSource.setMaxPoolSize(40);
        dataSource.setMinPoolSize(5);
        dataSource.setInitialPoolSize(10);
        dataSource.setMaxStatements(100);
    }

    public static Connection getConnection(){
        try {
            return dataSource.getConnection();
        } catch (SQLException throwables) {
            throwables.printStackTrace();
        }
        return null;
    }

    public static void returnConnection(Connection connection){
        if(null != connection) {
            try {
                connection.close();
            } catch (SQLException throwables) {
                throwables.printStackTrace();
            }
        }
    }
}
```

- main

```scala
import org.apache.spark.SparkConf
import org.apache.spark.internal.Logging
import org.apache.spark.streaming.dstream.DStream

object BasicApp extends Logging {
    def main(args: Array[String]): Unit = {
        val conf: SparkConf = new SparkConf().setAppName(this.getClass.getSimpleName).setMaster("local[2]")
        val ssc = new StreamingContext(conf, Seconds(5))

        val lines = ssc.socketTextStream("hadoop000", 9527)
        val result: DStream[(String, Int)] = lines.flatMap(_.split(",")).map((_, 1))

        result.foreachRDD(rdd => {
            rdd.foreachPartition(partition => {
                val connection = ConnectionPool.getConnection()  // Executor
                logError(s"$connection ... ")

                connection.setAutoCommit(false)
                val pstmt = connection.prepareStatement(s"insert into wc(word, cnt) values(?,?)")
                var count = 0;
                partition.foreach{
                    case (word, cnt) => {
                        count = count + 1
                        pstmt.setString(1, word)
                        pstmt.setInt(2, cnt)
                        pstmt.addBatch()

                        if(count % 5 == 0) {
                            logWarning("count: "+count)
                            pstmt.executeBatch()
                            connection.commit()
                        }
                    }
                }

                /**
                 * 1) 不满足if的在什么地方提交
                 * 2) 现在的index做法是不对的
                 */
                pstmt.executeBatch()
                connection.commit()
                ConnectionPool.returnConnection(connection)
            })
        })

        ssc.start()
        ssc.awaitTermination()
    }
}
```



### 有状态计算

#### updateStateByKey

The `updateStateByKey` operation allows you to maintain arbitrary state while continuously updating it with new information. To use this, you will have to do two steps.

1. Define the state - The state can be an arbitrary data type.
2. Define the state update function - Specify with a function how to update the state using the previous state and the new values from an input stream.

In every batch, Spark will apply the state update function for all existing keys, regardless of whether they have new data in a batch or not. If the update function returns `None` then the key-value pair will be eliminated.

- 代码1(重启状态失效)

```scala
val sparkConf = new SparkConf().setAppName(this.getClass.getCanonicalName).setMaster("local[2]")
val ssc = new StreamingContext(sparkConf, Seconds(5))
ssc.checkpoint("chk")  // 只要涉及到state的都需要chk下

val lines = ssc.socketTextStream("hadoop000", 9527)
lines.flatMap(_.split(","))
.map((_,1)).updateStateByKey(updateFunction)
.print()

ssc.start()
ssc.awaitTermination()


/**
   * (a,1) (a,1) (a,1)  ==> (1,1,1)
   */
def updateFunction(newValues:Seq[Int], preValues:Option[Int]) = {
  val current = newValues.sum
  val pre = preValues.getOrElse(0)
  Some(current + pre)
}
```

- 代码2（重启状态恢复）

```scala
import org.apache.spark.SparkConf
import org.apache.spark.streaming.{Seconds, StreamingContext}

object UpdateStateByKeyAppV2 {
  def main(args: Array[String]): Unit = {

    val checkpointDirectory = "chk10"
    def functionToCreateContext(): StreamingContext = {
      val sparkConf = new SparkConf().setAppName(this.getClass.getCanonicalName).setMaster("local[2]")
      val ssc = new StreamingContext(sparkConf, Seconds(5))
      ssc.checkpoint(checkpointDirectory)  // 只要涉及到state的都需要chk下

      val lines = ssc.socketTextStream("ruozedata001", 9527)
      lines.flatMap(_.split(",")).map((_,1)).updateStateByKey(updateFunction).print()
      
      ssc
    }

    val ssc = StreamingContext.getOrCreate(checkpointDirectory, functionToCreateContext _)

    ssc.start()
    ssc.awaitTermination()
  }

  def updateFunction(newValues:Seq[Int], preValues:Option[Int]) = {
    val current = newValues.sum
    val pre = preValues.getOrElse(0)
    Some(current + pre)
  }
}
```



#### mapWithState

```scala
val sparkConf = new SparkConf().setAppName(this.getClass.getCanonicalName).setMaster("local[2]")
val ssc = new StreamingContext(sparkConf, Seconds(5))
ssc.checkpoint("chk2")  // 只要涉及到state的都需要chk下

val lines = ssc.socketTextStream("hadoop000", 9527)
lines.flatMap(_.split(","))
  .map((_,1))
  .mapWithState(StateSpec.function(fun))
  .print()

ssc.start()
ssc.awaitTermination()


// (word, sum)
val fun = (word: String, value: Option[Int], state: State[Int]) => {
  if(state.isTimingOut()) {
    println("..isTimingOut...")
  } else {
    val sum = value.getOrElse(0) + state.getOption().getOrElse(0)
    val tmp = (word, sum)
    state.update(sum)  // 通过新来的和已有的累计计数，并更新状态
    tmp
  }
}
```





# 6 [Spark Structured Streaming](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html)

- a scalable and fault-tolerant stream processing engine built on the Spark SQL engine

- the computation is executed on the same optimized Spark SQL engine
- the system ensures end-to-end exactly-once fault-tolerance guarantees through checkpointing and Write-Ahead Logs

- Structured Streaming queries are processed using a *micro-batch processing* engine, which processes data streams as a series of small batch jobs thereby achieving end-to-end latencies as low as **100 milliseconds and exactly-once fault-tolerance** guarantees
-  since Spark 2.3, we have introduced a new low-latency processing mode called **Continuous Processing**, which can achieve end-to-end latencies as low as 1 millisecond with at-least-once guarantees

## 编程模型

The key idea in Structured Streaming is to treat a live data stream as a table that is being continuously appended. This leads to a new stream processing model that is very similar to a batch processing model. You will express your streaming computation as standard batch-like query as on a static table, and Spark runs it as an *incremental* query on the *unbounded* input table. Let’s understand this model in more detail.



Consider the input data stream as the “Input Table”. Every data item that is arriving on the stream is like a new row being appended to the Input Table.

<img src="pictures/Spark/structured-streaming-stream-as-a-table.png" style="zoom:48%;" />

A query on the input will generate the “Result Table”. Every trigger interval (say, every 1 second), new rows get appended to the Input Table, which eventually updates the Result Table. Whenever the result table gets updated, we would want to write the changed result rows to an external sink.

<img src="pictures/Spark/structured-streaming-model.png" style="zoom:48%;" />

### 基本概念

The "Output" is defined as what gets written out to the external storage. The output can be defined in a different mode:

- *Complete Mode* - The entire updated Result Table will be written to the external storage. It is up to the storage connector to decide how to handle writing of the entire table.
- *Append Mode* - Only the new rows appended in the Result Table since the last trigger will be written to the external storage. This is applicable only on the queries where existing rows in the Result Table are not expected to change.
- *Update Mode* - Only the rows that were updated in the Result Table since the last trigger will be written to the external storage (available since Spark 2.1.1). Note that this is different from the Complete Mode in that this mode only outputs the rows that have changed since the last trigger. If the query doesn’t contain aggregations, it will be equivalent to Append mode.

## Window Operations on Event Time

Aggregations over a sliding event-time window are straightforward with Structured Streaming and are very similar to grouped aggregations. In a grouped aggregation, aggregate values (e.g. counts) are maintained for each unique value in the user-specified grouping column. In case of window-based aggregations, aggregate values are maintained for each window the event-time of a row falls into. Let’s understand this with an illustration.

Imagine our [quick example](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#quick-example) is modified and the stream now contains lines along with the time when the line was generated. Instead of running word counts, we want to count words within 10 minute windows, updating every 5 minutes. That is, word counts in words received between 10 minute windows 12:00 - 12:10, 12:05 - 12:15, 12:10 - 12:20, etc. Note that 12:00 - 12:10 means data that arrived after 12:00 but before 12:10. Now, consider a word that was received at 12:07. This word should increment the counts corresponding to two windows 12:00 - 12:10 and 12:05 - 12:15. So the counts will be indexed by both, the grouping key (i.e. the word) and the window (can be calculated from the event-time).

![](pictures/Spark/structured-streaming-window.png)

- 有序数据

```scala
/**
* 有序数据
* 2030-10-10 11:55:00,cat
* 2030-10-10 11:57:00,cat
* 2030-10-10 12:01:00,cat
* 2030-10-10 12:11:00,owl
*/
val spark = SparkSession.builder()
  .appName(getClass.getSimpleName)
  .config("spark.sql.shuffle.partitions", 2)  // 加快速度
  .master("local[2]")
  .getOrCreate()

import spark.implicits._
val lines = spark.readStream
  .format("socket")
  .option("host", "hadoop000")
  .option("port", 9527)
  .load()
  .as[String].map(x=>{
    val splits: Array[String] = x.split(",")
    (splits(0), splits(1))
  }).toDF("timestamp", "word")
  .groupBy(window($"timestamp", "10 minutes", "5 minutes"), $"word").count()
  .writeStream
  .format("console")
  .option("truncate", "false") 
  .outputMode("complete")
  .start().awaitTermination()
```

- 滑动窗口源码测试

```scala
/**
* 测试数据在哪几个滑动窗口
* org.apache.spark.sql.catalyst.analysis.getWindow
* 
* 2030-10-10 12:02:01 
* 结果
* (2030-10-10 11:55:00,2030-10-10 12:05:00)
* (2030-10-10 12:00:00,2030-10-10 12:10:00)
*/
val spark = SparkSession.builder()
  .appName(getClass.getSimpleName)
  .config("spark.sql.shuffle.partitions", 2)  // 加快速度
  .master("local[2]")
  .getOrCreate()

val format = FastDateFormat.getInstance("yyyy-MM-dd HH:mm:ss")
val eventTime = "2030-10-10 12:02:01"
val eventTimestamp = format.parse(eventTime).getTime
val windowDuration = 10 * 60 * 1000L  // 10min
val slideDuration = 5 * 60 * 1000L
val startTime = 0L

val overlappingWindows = math.ceil(windowDuration * 1.0 / slideDuration).toInt
for(i <- 0 until(overlappingWindows)) {
  val division = (eventTimestamp - startTime) / slideDuration
  val ceil = math.ceil(division)
  // if the division is equal to the ceiling, our record is the start of a window
  val windowId = if (ceil == division) ceil + 1 else ceil
  val windowStart = (windowId + i - overlappingWindows) * slideDuration + startTime
  val windowEnd = windowStart + windowDuration

  println(s"${format.format(windowStart.toLong)}", s"${format.format(windowEnd.toLong)}")
}
```

#### Handling Late Data and Watermarking

Now consider what happens if one of the events arrives late to the application. For example, say, a word generated at 12:04 (i.e. event time) could be received by the application at 12:11. The application should use the time 12:04 instead of 12:11 to update the older counts for the window `12:00 - 12:10`. This occurs naturally in our window-based grouping – Structured Streaming can maintain the intermediate state for partial aggregates for a long period of time such that late data can update aggregates of old windows correctly, as illustrated below.

![](pictures/Spark/structured-streaming-late-data.png)

However, to run this query for days, it’s necessary for the system to bound the amount of intermediate in-memory state it accumulates. This means the system needs to know when an old aggregate can be dropped from the in-memory state because the application is not going to receive late data for that aggregate any more. To enable this, in Spark 2.1, we have introduced **watermarking**, which lets the engine automatically track the current event time in the data and attempt to clean up old state accordingly. You can define the watermark of a query by specifying the event time column and the threshold on how late the data is expected to be in terms of event time. For a specific window ending at time `T`, the engine will maintain state and allow late data to update the state until `(max event time seen by the engine - late threshold > T)`. In other words, late data within the threshold will be aggregated, but data later than the threshold will start getting dropped (see [later](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#semantic-guarantees-of-aggregation-with-watermarking) in the section for the exact guarantees). Let’s understand this with an example. We can easily define watermarking on the previous example using `withWatermark()` as shown below.

```scala
import org.apache.spark.sql.functions._
/**
         * 2030-10-10 12:02:00,cat
         * 2030-10-10 12:02:00,dog
         * 2030-10-10 12:03:00,dog
         * 2030-10-10 12:03:00,dog
         * 2030-10-10 12:07:00,owl
         * 2030-10-10 12:07:00,cat
         * 2030-10-10 12:04:00,dog
         * 2030-10-10 12:13:00,owl
         */
val spark = SparkSession.builder()
  .appName(getClass.getSimpleName)
  .config("spark.sql.shuffle.partitions", 2) // 加快速度
  .master("local[2]")
  .getOrCreate()

import spark.implicits._
spark.readStream
  .format("socket")
  .option("host", "hadoop000")
  .option("port", 9527)
  .load()
  .as[String].map(x => {
    val splits: Array[String] = x.split(",")
    (Timestamp.valueOf(splits(0)), splits(1))
  }).toDF("timestamp", "word")
  .withWatermark("timestamp", "10 minutes")
  .groupBy(window($"timestamp", "10 minutes", "5 minutes"), $"word").count()
  .writeStream
  .format("console")
  .option("truncate", "false")
  .outputMode("complete")
  .start().awaitTermination()
```

In this example, we are defining the watermark of the query on the value of the column “timestamp”, and also defining “10 minutes” as the threshold of how late is the data allowed to be. If this query is run in Update output mode (discussed later in [Output Modes](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#output-modes) section), the engine will keep updating counts of a window in the Result Table until the window is older than the watermark, which lags behind the current event time in column “timestamp” by 10 minutes. Here is an illustration.

![](pictures/Spark/structured-streaming-watermark-update-mode.png)

As shown in the illustration, the maximum event time tracked by the engine is the *blue dashed line*, and the watermark set as `(max event time - '10 mins')` at the beginning of every trigger is the red line. For example, when the engine observes the data `(12:14, dog)`, it sets the watermark for the next trigger as `12:04`. This watermark lets the engine maintain intermediate state for additional 10 minutes to allow late data to be counted. For example, the data `(12:09, cat)` is out of order and late, and it falls in windows `12:00 - 12:10` and `12:05 - 12:15`. Since, it is still ahead of the watermark `12:04` in the trigger, the engine still maintains the intermediate counts as state and correctly updates the counts of the related windows. However, when the watermark is updated to `12:11`, the intermediate state for window `(12:00 - 12:10)` is cleared, and all subsequent data (e.g. `(12:04, donkey)`) is considered “too late” and therefore ignored. Note that after every trigger, the updated counts (i.e. purple rows) are written to sink as the trigger output, as dictated by the Update mode.

Some sinks (e.g. files) may not supported fine-grained updates that Update Mode requires. To work with them, we have also support Append Mode, where only the *final counts* are written to sink. This is illustrated below.

Note that using `withWatermark` on a non-streaming Dataset is no-op. As the watermark should not affect any batch query in any way, we will ignore it directly.

![](pictures/Spark/structured-streaming-watermark-append-mode.png)

Similar to the Update Mode earlier, the engine maintains intermediate counts for each window. However, the partial counts are not updated to the Result Table and not written to sink. The engine waits for “10 mins” for late date to be counted, then drops intermediate state of a window < watermark, and appends the final counts to the Result Table/sink. For example, the final counts of window `12:00 - 12:10` is appended to the Result Table only after the watermark is updated to `12:11`.

#### Types of time windows

Spark supports three types of time windows: tumbling (fixed), sliding and session.

![The types of time windows](pictures/Spark/structured-streaming-time-window-types.jpeg)

Tumbling windows are a series of fixed-sized, non-overlapping and contiguous time intervals. An input can only be bound to a single window.

## Handling Event-time and Late Data

Event-time is the time embedded in the data itself. For many applications, you may want to operate on this event-time. For example, if you want to get the number of events generated by IoT devices every minute, then you probably want to use the time when the data was generated (that is, event-time in the data), rather than the time Spark receives them. This event-time is very naturally expressed in this model – each event from the devices is a row in the table, and event-time is a column value in the row. This allows window-based aggregations (e.g. number of events every minute) to be just a special type of grouping and aggregation on the event-time column – each time window is a group and each row can belong to multiple windows/groups. Therefore, such event-time-window-based aggregation queries can be defined consistently on both a static dataset (e.g. from collected device events logs) as well as on a data stream, making the life of the user much easier.

Furthermore, this model naturally handles data that has arrived later than expected based on its event-time. Since Spark is updating the Result Table, it has full control over updating old aggregates when there is late data, as well as cleaning up old aggregates to limit the size of intermediate state data. Since Spark 2.1, we have support for watermarking which allows the user to specify the threshold of late data, and allows the engine to accordingly clean up old state. These are explained later in more detail in the [Window Operations](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#window-operations-on-event-time) section.

## Fault Tolerance Semantics

Delivering end-to-end exactly-once semantics was one of key goals behind the design of Structured Streaming. To achieve that, we have designed the Structured Streaming sources, the sinks and the execution engine to reliably track the exact progress of the processing so that it can handle any kind of failure by restarting and/or reprocessing. Every streaming source is assumed to have offsets (similar to Kafka offsets, or Kinesis sequence numbers) to track the read position in the stream. The engine uses checkpointing and write-ahead logs to record the offset range of the data being processed in each trigger. The streaming sinks are designed to be idempotent for handling reprocessing. Together, using replayable sources and idempotent sinks, Structured Streaming can ensure **end-to-end exactly-once semantics** under any failure.



## Source

### 文件

```scala
val spark = SparkSession.builder()
                      .appName(getClass.getSimpleName)
                      .config("spark.sql.shuffle.partitions", 2)  // 加快速度
                      .master("local[2]")
                      .getOrCreate()

import spark.implicits._
val schema = new StructType()
                .add("name", StringType)
                .add("age", IntegerType)
val result = spark.readStream.format("csv")
                .schema(schema)
                .load("file:///Users/andy/Documents/sparkdemo/data/csv")
                .groupBy("age").count()

result.writeStream
    .format("console")
    .outputMode(OutputMode.Update())
    .start()
```

### [Kafka](https://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html)

```xml
<dependency>
  <groupId>org.apache.spark</groupId>
  <artifactId>spark-sql-kafka-0-10_${scala.tools.version}</artifactId>
  <version>${spark.version}</version>
</dependency>
```

- Streaming

```scala
val spark = SparkSession.builder()
  .appName(getClass.getSimpleName)
  .config("spark.sql.shuffle.partitions", 2)  // 加快速度
  .master("local[2]")
  .getOrCreate()

import spark.implicits._
val df = spark.readStream.format("kafka")
  .option("kafka.bootstrap.servers", "hadoop000:9092")
  .option("subscribe", "test01")
  .load()
  .selectExpr("CAST(value AS STRING)")
  .as[String].flatMap(_.split(",")).groupBy("value").count()

df.writeStream
  .format("console")
  .option("truncate", "false")  // 显示全部
  .outputMode(OutputMode.Update())
  .start()
  .awaitTermination()
```

- batch

```scala
import spark.implicits._
val df = spark.read.format("kafka")
  .option("kafka.bootstrap.servers", "hadoop000:9092")
  .option("subscribe", "test01")
  .option("startingOffsets","""{"test01":{"0":3}}""")
  .load()
  .selectExpr("CAST(value AS STRING)")
  .as[String].flatMap(_.split(",")).groupBy("value").count()

df.write
  .format("console")
  .option("truncate", "false")  // 显示全部
  .save()
```



## Sink

| Sink                  | Supported Output Modes   | Options                                                      | Fault-tolerant                                               | Notes                                                        |
| :-------------------- | :----------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **File Sink**         | Append                   | `path`: path to the output directory, must be specified. `retention`: time to live (TTL) for output files. Output files which batches were committed older than TTL will be eventually excluded in metadata log. This means reader queries which read the sink's output directory may not process them. You can provide the value as string format of the time. (like "12h", "7d", etc.) By default it's disabled.  For file-format-specific options, see the related methods in DataFrameWriter ([Scala](https://spark.apache.org/docs/latest/api/scala/org/apache/spark/sql/DataFrameWriter.html)/[Java](https://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/DataFrameWriter.html)/[Python](https://spark.apache.org/docs/latest/api/python/reference/api/pyspark.sql.streaming.DataStreamWriter.html#pyspark.sql.streaming.DataStreamWriter)/[R](https://spark.apache.org/docs/latest/api/R/write.stream.html)). E.g. for "parquet" format options see `DataFrameWriter.parquet()` | Yes (exactly-once)                                           | Supports writes to partitioned tables. Partitioning by time may be useful. |
| **Kafka Sink**        | Append, Update, Complete | See the [Kafka Integration Guide](https://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html) | Yes (at-least-once)                                          | More details in the [Kafka Integration Guide](https://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html) |
| **Foreach Sink**      | Append, Update, Complete | None                                                         | Yes (at-least-once)                                          | More details in the [next section](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#using-foreach-and-foreachbatch) |
| **ForeachBatch Sink** | Append, Update, Complete | None                                                         | Depends on the implementation                                | More details in the [next section](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#using-foreach-and-foreachbatch) |
| **Console Sink**      | Append, Update, Complete | `numRows`: Number of rows to print every trigger (default: 20) `truncate`: Whether to truncate the output if too long (default: true) | No                                                           |                                                              |
| **Memory Sink**       | Append, Complete         | None                                                         | No. But in Complete Mode, restarted query will recreate the full table. | Table name is the query name                                 |

### file(用得少)

```scala
val spark = SparkSession.builder()
  .appName(getClass.getSimpleName)
  .config("spark.sql.shuffle.partitions", 2) // 加快速度
  .master("local[2]")
  .getOrCreate()

import spark.implicits._
spark.readStream
  .format("socket")
  .option("host", "hadoop000")
  .option("port", 9527)
  .load()
  .as[String]
  .flatMap(_.split(",")).map(x=>(x, "andy"))
  .toDF("word", "name")
  .writeStream.format("json")
  .option("path", "file:///Users/andy/Documents/sparkdemo/data/csv")
  .option("checkpointLocation", "ck1")
  .trigger(Trigger.ProcessingTime(2000))  // 2s触发一次
  .outputMode("append")
  .start().awaitTermination()
```

### Memory 

```scala
import spark.implicits._
val query = spark.readStream
  .format("socket")
  .option("host", "hadoop000")
  .option("port", 9527)
  .load()
  .as[String]
  .flatMap(_.split(","))
  .groupBy("value").count()
  .writeStream
  .format("memory")
  .queryName("m_wc")
  .outputMode("complete")
  .start()

while (true) {
  Thread.sleep(3000)
  spark.sql("select * from m_wc").show(false)
}

query.awaitTermination()
```

### Foreach

```scala
import java.sql.{Connection, PreparedStatement}

spark.readStream
.format("socket")
.option("host", "hadoop000")
.option("port", 9527)
.load()
.as[String]
.flatMap(_.split(","))
.groupBy("value")
.count()
.writeStream
.outputMode("update")
.foreach(new ForeachWriter[Row] {
  var connection:Connection = _
  var pstmt:PreparedStatement = _

  override def open(partitionId: Long, epochId: Long): Boolean = {
    val sql =
    """
    |insert into wc(word, cnt) values(?, ?)
    |on duplicate key update word = ?, cnt = ?
    |""".stripMargin
    println("-------------open------------")
    connection = ConnectionPool.getConnection()
    connection.setAutoCommit(false)
    pstmt = connection.prepareStatement(sql)
    connection != null && !connection.isClosed
  }

  override def process(value: Row): Unit = {
    val word = value.getString(0)
    val cnt = value.getLong(1)

    pstmt.setString(1, word)
    pstmt.setLong(2, cnt)
    pstmt.setString(3, word)
    pstmt.setLong(4, cnt)

    println(s"$word, $cnt")
    pstmt.execute()
    connection.commit()
  }

  override def close(errorOrNull: Throwable): Unit = {
    ConnectionPool.returnConnection(connection)
  }
})
.start()
.awaitTermination()
```

- MySQL连接池

```scala
import com.mchange.v2.c3p0.ComboPooledDataSource;

import java.sql.Connection;
import java.sql.SQLException;

public class ConnectionPool {
       private static ComboPooledDataSource dataSource = new ComboPooledDataSource();
    static {
        dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/test");
        dataSource.setUser("root");
        dataSource.setPassword("xxxx");
        dataSource.setMaxPoolSize(40);
        dataSource.setMinPoolSize(5);
        dataSource.setInitialPoolSize(10);
        dataSource.setMaxStatements(100);
    }

    public static Connection getConnection(){
        try {
            return dataSource.getConnection();
        } catch (SQLException throwables) {
            throwables.printStackTrace();
        }
        return null;
    }

    public static void returnConnection(Connection connection){
        if(null != connection) {
            try {
                connection.close();
            } catch (SQLException throwables) {
                throwables.printStackTrace();
            }
        }
    }
}
```

### ForeachBatch

With `foreachBatch`, you can do the following.

- **Reuse existing batch data sources** - For many storage systems, there may not be a streaming sink available yet, but there may already exist a data writer for batch queries. Using `foreachBatch`, you can use the batch data writers on the output of each micro-batch.
- **Write to multiple locations** - If you want to write the output of a streaming query to multiple locations, then you can simply write the output DataFrame/Dataset multiple times. However, each attempt to write can cause the output data to be recomputed (including possible re-reading of the input data). To avoid recomputations, you should cache the output DataFrame/Dataset, write it to multiple locations, and then uncache it. Here is an outline.

```scala
spark.readStream
.format("socket")
.option("host", "hadoop000")
.option("port", 9527)
.load()
.as[String]
.flatMap(_.split(","))
.groupBy("value")
.count()
.writeStream
.outputMode("complete")
.foreachBatch((df:DataFrame, batchId:Long) => {
  // 都是批处理
  if (df.count() != 0) {
    df.write.json(s"file:///Users/andy/Documents/sparkdemo/data/json/$batchId")
    //df.write.mode("overwrite").jdbc("")
  }
})
.start()
.awaitTermination()
```

### Kafka

```scala
spark.readStream
.format("socket")
.option("host", "localhost")
.option("port", 9527)
.load()
.as[String]
.flatMap(_.split(","))
.groupBy("value")
.count()
//.map(row =>row.getString(0)+" : "+row.getString(1))
.writeStream
.outputMode("complete")
.format("kafka")
.option("kafka.bootstrap.servers", "hadoop000:9092")
.option("topic", "test01")
.option("checkpointLocation", "file:///Users/andy/Documents/sparkdemo/data/ck1")
.start()
.awaitTermination()
```



## Join Operations

Structured Streaming supports joining a streaming Dataset/DataFrame with a static Dataset/DataFrame as well as another streaming Dataset/DataFrame. The result of the streaming join is generated incrementally, similar to the results of streaming aggregations in the previous section. 

### Stream-static Joins

- Inner join

```scala
val spark = SparkSession.builder()
  .appName(getClass.getSimpleName)
  .config("spark.sql.shuffle.partitions", 2) // 加快速度
  .master("local[2]")
  .getOrCreate()

import spark.implicits._
val staticDS = spark.read.format("csv").option("inferSchema", true).option("header", true)
  .load("file:///Users/andy/Documents/sparkdemo/data/csv/people.csv")
  .as[People]

// 1,[10],100
val streamingDS = spark.readStream
  .format("socket")
  .option("host", "hadoop000")
  .option("port", 9527)
  .load()
  .as[String].map(x=>{
    val spilts = x.split(",")
    Area(spilts(0).toInt, spilts(1).toInt, spilts(2).toFloat)
  })

// inner
val res = streamingDS.join(staticDS, "age")

res.writeStream.format("console")
.outputMode("append")
.option("truncate", "false")
.start().awaitTermination()

case class People(name:String, age:Int)
case class Area(id:Int, age:Int, num:Float)
```

- Left join

```scala
val res = streamingDS.join(staticDS, Seq("age"), "left")
```



### Stream-stream Joins

[支持功能](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#support-matrix-for-joins-in-streaming-queries)

处理方案

- 1 数据临时存在某个地方
- 2 window+watermark
  - eventTime是在同一个window
  - late是要通过watermark来控制

In Spark 2.3, we have added support for stream-stream joins, that is, you can join two streaming Datasets/DataFrames. The challenge of generating join results between two data streams is that, at any point of time, the view of the dataset is incomplete for both sides of the join making it much harder to find matches between inputs. Any row received from one input stream can match with any future, yet-to-be-received row from the other input stream. Hence, for both the input streams, we buffer past input as streaming state, so that we can match every future input with past input and accordingly generate joined results. Furthermore, similar to streaming aggregations, we automatically handle late, out-of-order data and can limit the state using watermarks. 

#### Inner Joins with optional Watermarking

##### 不带wm

```scala
val spark = SparkSession.builder()
  .appName(getClass.getSimpleName)
  .config("spark.sql.shuffle.partitions", 2) // 加快速度
  .master("local[2]")
  .getOrCreate()

import spark.implicits._

val left = spark.readStream
  .format("socket")
  .option("host", "hadoop000")
  .option("port", 9527)
  .load()
  .as[String].map(x=>{
    val splits = x.split(",")
    (splits(0), splits(1), splits(2))
  }).toDF("age","bb","cc")

val right = spark.readStream
  .format("socket")
  .option("host", "hadoop000")
  .option("port", 9528)
  .load()
  .as[String].map(x=>{
    val spilts = x.split(",")
    Area(spilts(0).toInt, spilts(1).toInt, spilts(2).toFloat)
  })

val res = left.join(right, Seq("age"))

res.writeStream.format("console")
  .outputMode("append")
  .option("truncate", "false")
  .start().awaitTermination()

case class Area(id:Int, age:Int, num:Float)
```

##### 带wm



#### Outer Joins with Watermarking

While the watermark + event-time constraints is optional for inner joins, for outer joins they must be specified. This is because for generating the NULL results in outer join, the engine must know when an input row is not going to match with anything in future. Hence, the watermark + event-time constraints must be specified for generating correct results. Therefore, a query with outer-join will look quite like the ad-monetization example earlier, except that there will be an additional parameter specifying it to be an outer-join.



#### Streaming Deduplication(去重)

You can deduplicate records in data streams using a unique identifier in the events. This is exactly same as deduplication on static using a unique identifier column. The query will store the necessary amount of data from previous records such that it can filter duplicate records. Similar to aggregations, you can use deduplication with or without watermarking.

- *With watermark* - If there is an upper bound on how late a duplicate record may arrive, then you can define a watermark on an event time column and deduplicate using both the guid and the event time columns. The query will use the watermark to remove old state data from past records that are not expected to get any duplicates any more. This bounds the amount of the state the query has to maintain.
- *Without watermark* - Since there are no bounds on when a duplicate record may arrive, the query stores the data from all the past records as state.

```scala
/*
2,2030-11-30 11:51:00,d
1,2030-11-30 11:50:01,d
3,2030-11-30 11:53:01,d
1,2030-11-30 11:50:01,d(不参与计算 wm 11:51)
4,2030-11-30 11:45:01,d(不参与计算)
*/
spark.readStream
    .format("socket")
    .option("host", "hadoop000")
    .option("port", 9527)
    .load()
    .as[String].map(x => {
      val splits = x.split(",")
      (splits(0), Timestamp.valueOf(splits(1)), splits(2))
    }).toDF("id", "ts", "word")
    .withWatermark("ts", "2 minutes")
    .dropDuplicates("id")
    .writeStream.format("console")
    .outputMode("append")
    .option("truncate", "false")
    .start().awaitTermination()
```





## 案例

<img src="pictures/Spark/structured-streaming-example-model.png" style="zoom:80%;" />

- DataFrame

```scala
import org.apache.spark.sql.streaming.OutputMode
import org.apache.spark.sql.{DataFrame, SparkSession}

val spark = SparkSession.builder()
                      .appName(getClass.getSimpleName)
                      .config("spark.sql.shuffle.partitions", 2)  // 加快速度
                      .master("local[2]")
                      .getOrCreate()

val lines = spark.readStream.format("socket").option("host", "hadoop000").option("port", 9527).load()

import spark.implicits._
val result: DataFrame = lines.as[String].flatMap(_.split(",")).groupBy("value").count()
result.writeStream
    .format("console")
    .outputMode(OutputMode.Complete)
    .start()
    .awaitTermination()
```

- SQL

```scala
val spark = SparkSession.builder()
                      .appName(getClass.getSimpleName)
                      .config("spark.sql.shuffle.partitions", 2)  // 加快速度
                      .master("local[2]")
                      .getOrCreate()

val lines = spark.readStream.format("socket").option("host", "hadoop000").option("port", 9527).load()

import spark.implicits._
lines.as[String].flatMap(_.split(",")).createOrReplaceTempView("wc")

val result = spark.sql(
  						"""
              |select
              |value, count(1) as cnt
              |from
              |wc
              |group by value
              |""".stripMargin)

/**
* 非聚合
* append模式   ：  ok，数据不发生变化
* complete模式 ：  not support 数据聚合下使用 需要带watermark
* update模式   ：  等于append

* 聚合
* append模式   ：  not support
* complete模式 ：  ok 全量
* update模式   ：  只输出更新的数据
*/
result.writeStream
    .format("console")
    .outputMode(OutputMode.Complete)
    .start()
    .awaitTermination()
```



# 7 常见数据源操作([Data Source](https://spark.apache.org/docs/latest/sql-data-sources.html))

## 重要的类与方法

- org.apache.spark.sql.sources.BaseRelation：定义schema信息
- org.apache.spark.sql.sources.RelationProvider
- org.apache.spark.sql.sources.PrunedScan: 列裁剪 select
- org.apache.spark.sql.sources.PrunedFilteredScan：where

## 标准范式

```scala
/**
* spark.read.format("").load()
* df.write.format("").save("")    
*/
import spark.implicits._

val textDF: DataFrame = spark.read.format(“text”).load("data/input.txt")                        
val jsonDF: DataFrame = spark.read.format(“json”).load("data/people.json")
                                                                                                              textDF.write.format("json").mode(SaveMode.Overwrite).save("out")
```
### 相关接口

- BaseRelation
- TableScan： select * from xxx
- PrunedScan： select a,b from xxx 列裁剪
- PrunedFilteredScan： select a,b from xxx where a=1

```sql
JdbcRelation
CreatableRelationProvider
DataSourceRegister
```

- InsertableRelation: 写相关

### 案例

```scala
import org.apache.spark.sql.SQLContext
import org.apache.spark.sql.sources.{BaseRelation, RelationProvider}

class TextDefaultSource extends RelationProvider{
    override def createRelation(sqlContext: SQLContext, parameters: Map[String, String]): BaseRelation = {
        val path = parameters.get("path")
        path match {
            case Some(x) => new DataSourceRelation(sqlContext, x)
            case _ => throw new IllegalArgumentException("path not exists....")
        }
    }
}
```

- relation

```scala
import org.apache.spark.internal.Logging
import org.apache.spark.rdd.RDD
import org.apache.spark.sql.{Row, SQLContext}
import org.apache.spark.sql.sources.{BaseRelation, TableScan}
import org.apache.spark.sql.types.{LongType, StringType, StructField, StructType}

class DataSourceRelation(val SQLContext: SQLContext, val path:String) extends BaseRelation with TableScan with Logging {
    override def sqlContext: SQLContext = ???

    override def schema: StructType = StructType(
        StructField("id", LongType, false)::
        StructField("name", StringType, false):: Nil
    )

    override def buildScan(): RDD[Row] = {
        val lines = sqlContext.sparkContext.textFile(path)
        val fields: Array[StructField] = schema.fields

        lines.map(_.split(",").map(_.trim)).map(x=>x.zipWithIndex.map({
            case (value, key) =>{
                fields(key)
            }
        }))
        null
    }
}
```



## 文本文件

- 有几个partition就有几个task
- 空=>小文件 (没必要 需要调优)

```scala
import spark.implicits._

// 方式1
val textDF: DataFrame = spark.read.text("data/people.txt")
// 方式2
spark.read.format("text").load("data/people.txt")
// 方式3
val df: Dataset[String] = spark.read.textFile("data/people.txt")

val result: Dataset[(String)] = textDF.map(x => {
  val splits: Array[String] = x.getString(0).split(",")
  (splits(0).trim)
})

// 方式1
result.write.mode(SaveMode.Append).text("out")

// 方式2
result.write.option("compression", "gzip").mode("overwrite").text("out")
```
## csv

```scala
spark.read.format("csv").option("header", "true").option("sep", ",").load("data/a.csv")
```



## Parquet

```scala
// 列式存储 spark 默认模式
import spark.implicits._

val parquetDF: DataFrame = spark.read.parquet("data/users.parquet")

parquetDF.show()
parquetDF.printSchema()

parquetDF.select("name","favorite_color")
        .write.mode("overwrite")
        .option("compression","none")  // 不压缩
        .parquet("out")
```
## JSON
```scala
import spark.implicits._
val jsonDF: DataFrame = spark.read.json("data/people2.json")
jsonDF.select($"name", $"age", $"info.work".as("work")).write.mode(SaveMode.Overwrite).json("out1")
```

## 格式转换

- json ==> parquet

```scala
import spark.implicits._
val jsonDF: DataFrame = spark.read.format(“json”).load("data/people.json")

jsonDF.filter("age >= 30").write.format("parquet").mode(SaveMode.Overwrite).save("out")

spark.read.parquet("out").show()
```
## [MySQL](http://spark.apache.org/docs/latest/sql-data-sources-jdbc.html)(jdbc)
* pom.xml
```xml
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-sql_${scala.tools.version}</artifactId>
    <version>${spark.version}</version>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.47</version>
</dependency>
<dependency>
    <groupId>com.typesafe</groupId>
    <artifactId>config</artifactId>
    <version>1.3.3</version>
</dependency>
<dependency>
    <groupId>org.codehaus.janino</groupId>
    <artifactId>janino</artifactId>
    <version>3.0.8</version>
</dependency>
```
* resourses/application.conf
```yml
db.default.driver="com.mysql.jdbc.Driver"
db.default.url="jdbc:mysql://hadoop000:3306?useSSL=false"
db.default.user="root"
db.default.password="xxxxx@123"
db.default.database="test"
db.default.table="score"
db.default.sink.table="template_1"
```
* 代码
```scala
import java.util.Properties

import com.typesafe.config.{Config, ConfigFactory}
import org.apache.spark.sql.{DataFrame, SaveMode, SparkSession}

def main(args: Array[String]): Unit = {
  val spark = SparkSession.builder().master("local").appName("DataSourceApp").getOrCreate()
  jdbc(spark)
  spark.stop()
}

def jdbc(spark:SparkSession): Unit = {

  import spark.implicits._

  val config: Config = ConfigFactory.load()
  val url: String = config.getString("db.default.url")
  val user: String = config.getString("db.default.user")
  val password: String = config.getString("db.default.password")
  val driver: String = config.getString("db.default.driver")  // 用在YARN，Standalone集群上
  val database: String = config.getString("db.default.database")
  val table: String = config.getString("db.default.table")
  val sinkTable = config.getString("db.default.sink.table")

  // Loading data from a JDBC source
  val connectionProperties = new Properties()
  connectionProperties.put("user",user)
  connectionProperties.put("password",password)

  val jdbcDF: DataFrame = spark.read.jdbc(url, s"$database.$table", connectionProperties)

  jdbcDF.filter($"age"=== 20).write.mode(SaveMode.Overwrite).jdbc(url,s"$database.$sinkTable",connectionProperties)

  spark.read.jdbc(url,s"$database.$sinkTable",connectionProperties).show()
}
```

- 代码2

```scala
val df: DataFrame = spark.read.format("jdbc")
                        .option("url", "jdbc:mysql://hadoop000:3306/test?useSSL=false")
                        .option("user", "root")
                        .option("password", "xxxxxx@123")
                        .option("dbtable", "person")
                        .load()

df.printSchema()
df.show(false)

df.write.format("jdbc").mode(SaveMode.Overwrite)
  .option("url", "jdbc:mysql://hadoop000:3306/test?useSSL=false")
  .option("user", "root")
  .option("password", "xxxxxx@123")
  .option("dbtable", "person3")
  .save()
```



### 总结

#### 问题1：jdbcDF的partition个数是多少？ 1个partition

​          numPartitions partitionColumn, lowerBound, upperBound

​          mysql里面有10W条数据。我期望2个partition，你怎么实现？

        ```properties
        partition1:  id<=1<5W
        partition2:  id<=5W<10W
        
        numPartitions : 2
        partitionColumn : id
        lowerBound: 1
        upperBound: 10W
        ```



#### 延伸问题1

mysql是有增删改操作的 。id被删了->id不连续.造成每个partition数据量不均匀=》数据倾斜，如何使数据均匀的分布到每个partition？

可以对id取模。=> mod(21,10)=1   mod(22,2)=0

#### 延伸问题2

- 对id取模的性能如何？他会触发全表扫描，性能很低。最终方案是什么样？

        ```sql
        # 终极解决方案，借助翻页来实现。
        
        ##步骤1：获取该表的最小ID   id=10
        select min(id) from table  ==>minID=df.collect.head.get(0).toString
        ##步骤2：
        	select * from table where id>=minID order by id limit 100000; =>DF  
        ##步骤3：cache步骤2的DF
        ##步骤4：==>insert到Hive表中 action
        ##步骤5：计算步骤2DF中id的最大值 id=110000 action
        ## 步骤6：循环执行2，3，4，5  
        select * from table where id>110000 order by id limit 100000;
        ##步骤7：步骤2的DF count=0的时候，跳出循环。
        ```



#### 问题2：schema如何进行推断的？  

```sql
select * from table where 1=0
```



## HBase(065)

External DataSource V1 / V2

```scala
// 可以是简写：DataSourceRegister
// ...全称：包名
/*
hbase table: h_user

spark sql: s_user
	select * from user;
	select name,age from user;
	select * from user where name='pk';

h_user和s_user建立映射关系
hbase表中的字段和spark sql表中的字段有映射关系
*/
spark.read.format("...").option("","").load()

val user = spark.read.format("com.rzdata.spark.source.hbase")
  .option("hbase.table.name", "user")  // 设定hbase表
  .option("spark.table.schema", "(age int, name string, sex string)")
  .load()

user注册成一个临时视图： SQL
使用df/ds：API
```



```scala
val spark: SparkSession = SparkSession.builder().appName(getClass.getSimpleName).master("local[2]").getOrCreate()

val user = spark.read.format("com.apache.spark.habse.hbase").option("hbase.table.name", "user")
.option("spark.table.schema", "(age int, name string, sex string)")
.load()

user.printSchema()
user.show(false)
```



```scala
package com.apache.spark.habse

package object hbase {
  case class SparkSchema(fieldName:String, fieldType:String)
}

// DefaultSource
class DefaultSource extends RelationProvider{
  override def createRelation(sqlContext: SQLContext, parameters: Map[String, String]): BaseRelation = {
    HBaseRelation(sqlContext, parameters)
  }
}

// HBaseSourceUtils
object HBaseSourceUtils {
  /**
   * @param sparkTableSchema (age int, name string, sex string)
   * @return
   */
  def extractSparkFields(sparkTableSchema:String):Array[SparkSchema] = {

    val columns = sparkTableSchema.trim.drop(1).dropRight(1).split(",")
    val res: Array[SparkSchema] = columns.map(column => {
      val splits = column.trim.split(" ")
      SparkSchema(splits(0), splits(1))
    })
    res
  }
}
```

- HBaseRelation

```scala
import org.apache.hadoop.hbase.client.Result
import org.apache.hadoop.hbase.io.ImmutableBytesWritable
import org.apache.hadoop.hbase.mapreduce.TableInputFormat
import org.apache.hadoop.hbase.util.Bytes
import org.apache.hadoop.hbase.{HBaseConfiguration, HConstants}
import org.apache.spark.internal.Logging
import org.apache.spark.rdd.RDD
import org.apache.spark.sql.{Row, SQLContext}
import org.apache.spark.sql.sources.{BaseRelation, TableScan}
import org.apache.spark.sql.types.{IntegerType, StringType, StructField, StructType}

import scala.collection.mutable.ArrayBuffer

case class HBaseRelation(val sqlContext: SQLContext, val parameters:Map[String,String])
  extends BaseRelation with TableScan with Logging {

  val hbaseTable = parameters.getOrElse("hbase.table.name", sys.error("hbase.table.name is required..."))
  val sparkTableSchema = parameters.getOrElse("spark.table.schema", sys.error("spark.table.schema is required..."))

  private val sparkFields: Array[SparkSchema] = HBaseSourceUtils.extractSparkFields(sparkTableSchema)


  override def schema: StructType = {
    val fields = sparkFields.map(field => {
      val structField = field.fieldType.toLowerCase match {
        case "string" => StructField(field.fieldName, StringType)
        case "int" => StructField(field.fieldName, IntegerType)
      }
      structField
    })
    new StructType(fields)
  }


  // 符合schema的rdd
  override def buildScan(): RDD[Row] = {
    val configuration = HBaseConfiguration.create()
    configuration.set(HConstants.ZOOKEEPER_QUORUM,"ruozedata001")
    configuration.set(HConstants.ZOOKEEPER_CLIENT_PORT, "2181")
    configuration.set(TableInputFormat.SCAN_COLUMNS,"") // 列裁剪
    configuration.set(TableInputFormat.SCAN_ROW_START,"")
    configuration.set(TableInputFormat.SCAN_ROW_STOP,"")
    configuration.set(TableInputFormat.INPUT_TABLE, hbaseTable)

    val rdd: RDD[(ImmutableBytesWritable, Result)]
        = sqlContext.sparkContext.newAPIHadoopRDD(configuration,
            classOf[TableInputFormat],
            classOf[ImmutableBytesWritable],
            classOf[Result])

    rdd.map(_._2).map(result => {
      val buffer = new ArrayBuffer[Any]()
      sparkFields.foreach(field => {
        field.fieldType.toLowerCase match {
          case "string" =>  {
            buffer += new String(result.getValue(Bytes.toBytes("o"), Bytes.toBytes(field.fieldName)))

          }
          case "int" =>  {
            buffer += Integer.parseInt(new String(result.getValue(Bytes.toBytes("o"), Bytes.toBytes(field.fieldName))))
          }
        }
      })
      Row.fromSeq(buffer)
    })
  }
}
```

- pom

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>sparkhbase</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<target.java.version>1.8</target.java.version>
		<scala.binary.version>2.12</scala.binary.version>
		<maven.compiler.source>${target.java.version}</maven.compiler.source>
		<maven.compiler.target>${target.java.version}</maven.compiler.target>
		<encoding>UTF-8</encoding>
		<log4j.version>2.12.1</log4j.version>
        <hadoop.version>2.6.3</hadoop.version>
        <spark.version>2.4.6</spark.version>
		<hbase.version>2.1.7</hbase.version>
		<jacksom.version>2.6.5</jacksom.version>
	</properties>

    <dependencies>
		<dependency>
			<groupId>io.netty</groupId>
			<artifactId>netty-all</artifactId>
			<version>4.1.17.Final</version>
		</dependency>

		 <dependency>
			<groupId>com.fasterxml.jackson.core</groupId>
			<artifactId>jackson-databind</artifactId>
			<version>${jacksom.version}</version>
	   </dependency>
		<dependency>
			<groupId>com.fasterxml.jackson.core</groupId>
			<artifactId>jackson-core</artifactId>
			<version>${jacksom.version}</version>
		</dependency>
		<dependency>
			<groupId>com.fasterxml.jackson.core</groupId>
			<artifactId>jackson-annotations</artifactId>
			<version>${jacksom.version}</version>
		</dependency>

           <dependency>
			   <groupId>org.apache.hbase</groupId>
			   <artifactId>hbase-client</artifactId>
			   <version>${hbase.version}</version>
			   <exclusions>
				   <exclusion>
					   <artifactId>jackson-core-asl</artifactId>
					   <groupId>org.codehaus.jackson</groupId>
				   </exclusion>
				   <exclusion>
					   <artifactId>jackson-databind</artifactId>
					   <groupId>com.fasterxml.jackson.core</groupId>
				   </exclusion>
			   </exclusions>
		   </dependency>
		   <dependency>
			   <groupId>org.apache.hbase</groupId>
			   <artifactId>hbase-mapreduce</artifactId>
			   <version>2.1.7</version>
			   <exclusions>
				   <exclusion>
					   <artifactId>jackson-databind</artifactId>
					   <groupId>com.fasterxml.jackson.core</groupId>
				   </exclusion>
				   <exclusion>
					   <artifactId>jackson-core</artifactId>
					   <groupId>com.fasterxml.jackson.core</groupId>
				   </exclusion>
				   <exclusion>
					   <artifactId>jackson-annotations</artifactId>
					   <groupId>com.fasterxml.jackson.core</groupId>
				   </exclusion>
			   </exclusions>
		   </dependency>

          <dependency>
                <groupId>org.apache.spark</groupId>
                <artifactId>spark-sql_${scala.binary.version}</artifactId>
                <version>${spark.version}</version>
			  <exclusions>
				  <exclusion>
					  <artifactId>netty-all</artifactId>
					  <groupId>io.netty</groupId>
				  </exclusion>
				  <exclusion>
					  <artifactId>jackson-databind</artifactId>
					  <groupId>com.fasterxml.jackson.core</groupId>
				  </exclusion>
				  <exclusion>
					  <artifactId>jackson-annotations</artifactId>
					  <groupId>com.fasterxml.jackson.core</groupId>
				  </exclusion>
				  <exclusion>
					  <artifactId>jackson-core</artifactId>
					  <groupId>com.fasterxml.jackson.core</groupId>
				  </exclusion>
			  </exclusions>
		  </dependency>
    </dependencies>

</project>
```



# 整合Hive操作及函数

## 原理
* 场景：历史原因积累下来的，很多数据原先是采用Hive来进行处理的，现在想改用Spark来进行数据，我们必须要求Spark能够无缝对接已有的Hive的数据(平滑的过渡)
* 操作是非常简单的，MetaStore
       * Hive底层的元数据信息是存储在MySQL中  $HIVE_HOME/conf/hive-site.xml
       * Spark如果能够直接访问到MySQL中已有的元数据信息  $SPARK_HOME/conf/hive-site.xml
```shell
# step 1 拷贝hive-site.xml
ln -s ~/app/hive-1.1.0-cdh5.15.1/conf/hive-site.xml ~/app/spark-2.4.3-bin-2.6.0-cdh5.15.1/conf/hive-site.xml

# step 2 启动
./spark-shell --master local[2] --jars ~/software/mysql-connector-java-5.1.27-bin.jar

# 启动spark-sql
./spark-sql --master local[2] --jars ~/software/mysql-connector-java-5.1.27-bin.jar --driver-class-path ~/software/mysql-connector-java-5.1.27-bin.jar
```
## Thriftserver使用
* 目前启动spark方式
	* spark-shell
	* spark-sql
	* IDEA local
	(以上单机模式)
	* 代码打包，不管以什么模式运行在服务器

* Server-Client  *****
	* 我们在服务器上启动一个Spark应用程序的Server  7*24一直running
	* 客户端可以连到Server上去干活
```shell
# 启动thriftserver  
cd $SPARK_HOME/sbin
# 启动
./start-thriftserver.sh --master local --jars ~/software/mysql-connector-java-5.1.27-bin.jar

# 内置了一个客户端工具  
cd $SPARK_HOME/bin/beeline
# 启动beeline
./beeline -u jdbc:hive2://hadoop000:10000
```

* 用beeline太麻烦，使用代码来连接ThriftServer
## 通过代码访问数据
* 打成jar包，然后定时执行  d h minute10
	* 1）调度框架
	* 2）crontab
```scala
import java.sql.{Connection, DriverManager, PreparedStatement, ResultSet}

object JDBCClientApp {

  def main(args: Array[String]): Unit = {

    // 加载驱动
    Class.forName("org.apache.hive.jdbc.HiveDriver")

    val conn: Connection = DriverManager.getConnection("jdbc:hive2://hadoop000:10000")
    val pstmt: PreparedStatement = conn.prepareStatement("select * from emp")
    val rs: ResultSet = pstmt.executeQuery()

    while (rs.next()) {
      println(rs.getObject(1) + " : " + rs.getObject(2))
    }
  }
}
```
* ThriftServer  vs 例行Spark Application
    * 1）两者都是应用程序
    * 2）ThriftServer 7*24h都在，例行跑完就结束了
    * 3）例行：启动需要去申请资源   Server：只有启动的时候申请
    * 4）Server：多个提交的作业能资源共享
* 思考题：10分钟  如何设计开发一个Spark作业的Server，每次只要发一个HTTP/REST请求，那么就提交到Spark Server上去运行

## Spark代码访问Hive数据

- 启动窗口

```shell
spark-shell --master local[2] --jars /home/hadoop/app/apache-hive-2.3.8-bin/lib/hive-contrib-2.3.8.jar
```

- 代码

```scala
System.setProperty("HADOOP_USER_NAME", "hadoop")

spark.catalog.listTables("ruozedata").show(3, false)
spark.catalog.listColumns("ruozedata", "emp").show(false)
```

- 操作数据

```scala
case class Template(id:Int, sqls:String, series:String)


//System.setProperty("HADOOP_USER_NAME", "hadoop")
val spark = SparkSession.builder()
  .master("local[2]")
  .appName(getClass.getSimpleName)
  .config("hive.exec.dynamic.partition.mode", "nonstrict")
  .enableHiveSupport()
  .getOrCreate()

import spark.implicits._
val templates = spark.read.format("jdbc")
    .option("url", "jdbc:mysql://hadoop000:3306?useSSL=false")
    .option("dbtable", "test.hive")
    .option("driver", "com.mysql.jdbc.Driver")
    .option("user", "maxwell")
    .option("password", "xxxxxx")
    .load()
    .filter('series === "access2").as[Template].collect()

val time = spark.conf.get("spark.time", "20220103")  // 获取参数
// scala par 并行
templates.par
  .map(x=>{
    // select deptno,dname,loc,day from test.dept where day='20220101'      //spark.sql(x.sqls).write.partitionBy("day").mode(SaveMode.Overwrite).format("hive").saveAsTable("test.student2")
     spark.sql(x.sqls).write.mode(SaveMode.Overwrite).insertInto("test.dept2")
})

spark.stop()
```

- log4j

```properties
log4j.rootLogger=warn, stdout

log4j.appender.stdout = org.apache.log4j.ConsoleAppender
log4j.appender.stdout.target = System.out
log4j.appender.stdout.layout = org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern= "%d{yyyy-MM-dd HH:mm:ss} %p [%c:%L] - %m%n
```

- pom

```xml
<dependency>
  <groupId>mysql</groupId>
  <artifactId>mysql-connector-java</artifactId>
  <version>5.1.45</version>
</dependency>
<dependency>
  <groupId>org.apache.spark</groupId>
  <artifactId>spark-hive_2.12</artifactId>
  <version>2.4.3</version>
</dependency>
<dependency>
  <groupId>org.apache.spark</groupId>
  <artifactId>spark-sql_${scala.binary.version}</artifactId>
  <version>${spark.version}</version>
</dependency>
<dependency>
  <groupId>org.apache.hadoop</groupId>
  <artifactId>hadoop-client</artifactId>
  <version>${hadoop.version}</version>
</dependency>

<dependency>
  <groupId>org.apache.logging.log4j</groupId>
  <artifactId>log4j-core</artifactId>
  <version>${log4j.version}</version>
</dependency>
```



# Spark源码编译

- 版本 3.1.1
- https://spark.apache.org/docs/latest/building-spark.html#building-a-runnable-distribution

## 编译命令

```shell
./dev/make-distribution.sh --name hadoop-2.7 --pip --tgz -Phive -Phive-thriftserver -Pyarn -Pkubernetes -Dhadoop.version=2.7
```

## 修改subl make-distribution.sh

```sh
VERSION=3.1.1
SCALA_VERSION=2.12
SPARK_HADOOP_VERSION=2.7
SPARK_HIVE=1
# VERSION=$("$MVN" help:evaluate -Dexpression=project.version $@ \
#     | grep -v "INFO"\
#     | grep -v "WARNING"\
#     | tail -n 1)
# SCALA_VERSION=$("$MVN" help:evaluate -Dexpression=scala.binary.version $@ \
#     | grep -v "INFO"\
#     | grep -v "WARNING"\
#     | tail -n 1)
# SPARK_HADOOP_VERSION=$("$MVN" help:evaluate -Dexpression=hadoop.version $@ \
#     | grep -v "INFO"\
#     | grep -v "WARNING"\
#     | tail -n 1)
# SPARK_HIVE=$("$MVN" help:evaluate -Dexpression=project.activeProfiles -pl sql/hive $@ \
#     | grep -v "INFO"\
#     | grep -v "WARNING"\
#     | fgrep --count "<id>hive</id>";\
#     # Reset exit status to 0, otherwise the script stops here if the last grep finds nothing\
#     # because we use "set -o pipefail"
#     echo -n)
```

## 修改案例

1. 在sparkcontext中加deleteFile，相对应addFile
2. 在SparkSQL中加ANTLR4(IDEA加插件antlr v4)  
   1. sqlbase.g4(加delete 语法)
   2. org.apache.spark.sql.execution.SparkSqlParser#visitManageResource
   3. SparkSQLCLIDriver#processCmd
      1. proc.isInstanceOf[DeleteResourceProcessor]

# Spark 脚本

## spark-shell

```shell
cygwin=false
case "$(uname)" in
  CYGWIN*) cygwin=true;;
esac
```

- 案例

```shell
#!/usr/bin/env bash

read -p "press " KEY

case $KEY in
[a-z]|[A-Z])
echo "letter"
;;
[0-9])
echo "digit"
;;
*)
echo "other"
esac
```



# Spark面试

## Yarn的client与cluster总结(057)

- 两者最大的区别在于Driver在何处执行，cluster模式，SparkSubmit进程起来后干掉进程不会影响driver端运行

- 对应进程有哪些？各自职责
  - cluster
    - SparkSubmit
    - ApplicationMaster
    - YarnCoarseGrainedExecutorBackend（2个）
  - client
    - SparkSubmit
    - ExecutorLauncher(也是启动 ApplicationMaster)
    - YarnCoarseGrainedExecutorBackend（2个）
- Spark执行流程是什么样？

![](pictures/Spark/源码2.jpg)

1. 提交最后一个RDD的5个属性给DAG（由于血缘关系，最后一个包含之前RDD的关联信息）
2. DAG中，如果是宽依赖，会拆成多个stage，然后每个stage会拆成taskSet，提交到TaskScheduler
3. TaskScheduler根据模式运行

### **Yarn Client** **模式**

第一步，Driver端在任务提交的本地机上运行

第二步，Driver启动之后就会和ResourceManager通讯，申请启动一个ApplicationMaster

第三步，ResourceManager就会分配container容器，在合适的nodemanager上启动ApplicationMaster，负责向ResourceManager申请Executor内存

第四步，ResourceManager接到ApplicationMaster的资源申请后会分配container，然后ApplicationMaster在资源分配指定的NodeManager上启动Executor进程

第五步，Executor进程启动后会向Driver反向注册，Executor全部注册完成后Driver开始执行main函数

第六步，之后执行到Action算子时，触发一个Job，并根据宽依赖开始划分stage，每个stage生成对应的TaskSet，之后将task分发到各个Executor上执行。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f715056bc8a24a47b29adaf85ce07029~tplv-k3u1fbpfcp-watermark.awebp)

### **Yarn Cluster** **模式**

第一步，在YARN Cluster模式下，任务提交后会和ResourceManager通讯申请启动ApplicationMaster

第二步， 随后ResourceManager分配container，在合适的NodeManager上启动ApplicationMaster，此时的ApplicationMaster就是Driver。

第三步， Driver启动后向ResourceManager申请Executor内存，ResourceManager接到ApplicationMaster的资源申请后会分配container，然后在合适的NodeManager上启动Executor进程

第四步，Executor进程启动后会向Driver反向注册，Executor全部注册完成后Driver开始执行main函数，

第五步，之后执行到Action算子时，触发一个Job，并根据宽依赖开始划分stage，每个stage生成对应的TaskSet，之后将task分发到各个Executor上执行

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ef13e83f0cd848d3bf28a80c6d1709dd~tplv-k3u1fbpfcp-watermark.awebp)



### SparkSubmit执行流程

<img src="pictures/Spark/源码2.jpeg" style="zoom:50%;" />

- SparkSubmit

```scala
// org.apache.spark.deploy.InProcessSparkSubmit#main
SparkSubmit.main{
 val submit = new SparkSubmit()
 submit.doSubmit{
   val appArgs = parseArguments(args) {
     // 解析传进来的参数 org.apache.spark.deploy.SparkSubmitArguments
     mergeDefaultSparkProperties()  // 从配置文件加载     
     ignoreNonSparkProperties()     // Remove keys that don't start with "spark." from `sparkProperties`.
     loadEnvironmentArguments() {   // 加载环境变量
       action = Option(action).getOrElse(SUBMIT)
     } 
   }
   
   submit(appArgs, uninitLog){
     doRunMain() {
       runMain(args, uninitLog) {
         // child class ? 反射
         val (childArgs, childClasspath, sparkConf, childMainClass) = prepareSubmitEnvironment(args){
           if (deployMode == CLIENT){
             childMainClass = args.mainClass  // 用户指定的类 --class
           }
           if (isYarnCluster){
             childMainClass = "org.apache.spark.deploy.yarn.YarnClusterApplication"
           }
         }
         
         var mainClass = Utils.classForName(childMainClass)  // 反射
         val app: SparkApplication  // 需要一个无参构造器 不然生成一个有参构造器就没有无参构造器
         app.start(childArgs.toArray, sparkConf){
           new Client(new ClientArguments(args), conf, null).run(){  // 开始连接Yarn
             // Submit an application running our ApplicationMaster to the ResourceManager
             this.appId = submitApplication(){
                val newApp = yarnClient.createApplication()  // 链接到YARN代码
              	// Set up the appropriate contexts to launch our AM
                val containerContext = createContainerLaunchContext(newAppResponse){
                  val amClass =
                  if (isClusterMode) {
                    Utils.classForName("org.apache.spark.deploy.yarn.ApplicationMaster").getName // cluster
                  } else {
                    Utils.classForName("org.apache.spark.deploy.yarn.ExecutorLauncher").getName  // client
                  }
                }
                val appContext = createApplicationSubmissionContext(newApp, containerContext) 
             }
           }
         }
       }
     }
   }
 }
}
```

- ApplicationMaster

```scala
// cluster
// org.apache.spark.deploy.yarn.ApplicationMaster#main
ApplicationMaster#main{
    val amArgs = ApplicationMasterArguments(args)
    master = new ApplicationMaster(amArgs, sparkConf, yarnConf)
    master.run(){
        runImpl(){
            if (isClusterMode) {
            runDriver(){
                userClassThread = startUserApplication(){
                  // 进入 --class 类中
                    val mainMethod = 
                  		userClassLoader.loadClass(args.userClass).getMethod("main", classOf[Array[String]])
                  
                    mainMethod.invoke(null, userArgs.toArray)  // 反射获取SparkContext等
                  
                    // Driver是包含main方法，并启动Spark Context，但在cluster模式下就是个线程 过程如下 SparkContext
                    userThread.setName("Driver")  // 线程
                }

                val sc = ThreadUtils.awaitResult(sparkContextPromise.future,
                    Duration(totalWaitTime, TimeUnit.MILLISECONDS))
                registerAM(host, port, userConf, sc.ui.map(_.webUrl))
                createAllocator(driverRef, userConf){
                    allocator.allocateResources(){
                        handleAllocatedContainers(allocatedContainers.asScala){
                            runAllocatedContainers(containersToUse){
                                for (container <- containersToUse){
                                    launcherPool.execute(Thread).run(){  // 线程池启动一个线程
                                        startContainer(){  // 启动container
                                            val commands = prepareCommand(){
                                                org.apache.spark.executor.CoarseGrainedExecutorBackend
                                            }
                                        }

                                        nmClient.startContainer(container.get, ctx)
                                    }  
                                }
                            }
                        }
                    }
                }
            }
        } else {
            runExecutorLauncher()
        }
        }
    }
}

// SparkContext
SparkContext{
    private var _schedulerBackend: SchedulerBackend = _  // 通信
    private var _taskScheduler: TaskScheduler = _
    private var _heartbeatReceiver: RpcEndpointRef = _
    @volatile private var _dagScheduler: DAGScheduler = _

    val (sched, ts) = SparkContext.createTaskScheduler(this, master, deployMode){ // 初始化
        val scheduler = cm.createTaskScheduler(sc, masterUrl){
            // local 走 org.apache.spark.scheduler.TaskSchedulerImpl
            case "cluster" => new YarnClusterScheduler(sc)
            case "client" => new YarnScheduler(sc)
        }
        val backend = cm.createSchedulerBackend(sc, masterUrl, scheduler){
            sc.deployMode match {
                case "cluster" =>
                    new YarnClusterSchedulerBackend(scheduler.asInstanceOf[TaskSchedulerImpl], sc)
                case "client" =>
                    new YarnClientSchedulerBackend(scheduler.asInstanceOf[TaskSchedulerImpl], sc)
            }
        }
        cm.initialize(scheduler, backend){
            scheduler.asInstanceOf[TaskSchedulerImpl].initialize(backend){
                schedulableBuilder.buildPools()  // 调度池
            }
        }
        (backend, scheduler)
    }
    _dagScheduler = new DAGScheduler(this)
    _taskScheduler.start(){
        backend.start(){  // org.apache.spark.scheduler.cluster.CoarseGrainedSchedulerBackend
            driverEndpoint = createDriverEndpointRef(properties){  // 与Driver端通信
                rpcEnv.setupEndpoint(ENDPOINT_NAME, createDriverEndpoint(properties)){
                    new DriverEndpoint(rpcEnv, properties){
                        override def onStart(){  // 生命周期方法
                            Option(self).foreach(_.send(ReviveOffers))  // TODO 发信息是否存活
                        }
                        override def receive{
                            makeOffers(executorId)
                        }
                    }
                }
            }
        }
    }
}
```

- Scheduler继承关系

```scala
TaskScheduler
	TaskSchedulerImpl
		YarnScheduler
			YarnClusterScheduler
```

- CoarseGrainedExecutorBackend

```scala
CoarseGrainedExecutorBackend.main{
    run(driverUrl, executorId, hostname, cores, appId, workerUrl, userClassPath){
        val driver = fetcher.setupEndpointRefByURI(driverUrl)
        val env = SparkEnv.createExecutorEnv(driverConf, executorId, hostname, cores, cfg.ioEncryptionKey, isLocal = false)
        env.rpcEnv.setupEndpoint("Executor", new CoarseGrainedExecutorBackend){
            onstart{
                // CoarseGrainedSchedulerBackend.DriverEndpoint#receiveAndReply
                executorRef.send(RegisteredExecutor){
                    CoarseGrainedExecutorBackend#receive{
                        case RegisteredExecutor =>{
                            executor = new Executor
                        }
                        
                    }
                }
            }
        }
    }
}
```

- RDD流程

触发action算子才会真正生成Job

```scala
sc.runJob{
    N个runJob{
        dagScheduler.runJob(rdd, cleanedFunc, partitions, callSite, resultHandler, localProperties.get){
            submitJob{
                val maxPartitions = rdd.partitions.length // 分区数是最后一个rdd的分区数
                val waiter = new JobWaiter
                JobSubmitted{
                    dagScheduler.handleJobSubmitted
                }
            }

            handleJobSubmitted{
                // 最后一个stage
                var finalStage: ResultStage = null
                finalStage = createResultStage(finalRDD, func, partitions, jobId, callSite){
                    val parents = getOrCreateParentStages(rdd, jobId) {  // stage的拆分原则：从后往前拆
                        val parents = new HashSet[ShuffleDependency[_, _, _]]  // 所有的shuffle遗留
                        val visited = new HashSet[RDD[_]]  // 查找过的
                        val waitingForVisit = new ArrayStack[RDD[_]]  // 待查找的 
                    }
                }

                submitStage(finalStage){
                    val missing = getMissingParentStages(stage).sortBy(_.id)
                    if (missing.isEmpty) {  // 如果为空 提交当前的stage
                        logInfo("Submitting " + stage + " (" + stage.rdd + "), which has no missing parents")
                        submitMissingTasks(stage, jobId.get){
                            ShuffleMapStage => ShuffleMapTask
                            ResultStage     => ResultTask

                            taskScheduler.submitTasks
                        }
                        } else {
                        for (parent <- missing) {
                            submitStage(parent)  // 不为空 把父stage提交
                        }
                        waitingStages += stage
                        }
                    }
                }
            }
        }
    }
}

TaskScheduler.submitTasks{
    val manager = createTaskSetManager(taskSet, maxTaskFailures)
    backend.reviveOffers(){
        localEndpoint.send(ReviveOffers){
            receiveOffers(){
                launchTasks(taskDescs){
                    case LaunchTask(data) =>
                        if (executor == null) {
                            exitExecutor(1, "Received LaunchTask command but executor was null")
                        } else {
                            val taskDesc = TaskDescription.decode(data.value)
                            logInfo("Got assigned task " + taskDesc.taskId)
                            executor.launchTask(this, taskDesc){
                                val res = task.run(
                                    taskAttemptId = taskId,
                                    attemptNumber = taskDescription.attemptNumber,
                                    metricsSystem = env.metricsSystem
                                )
                            }
                        }
                }
            }
        }
    }
}
```

- 拆分stage

​    根据finalRDD创建ResultStage实例finalStage

​    从finalStage开始，根据RDD的Dependency关系，从后往前推导

​    只要发现是ShuffleDependency，就会把父的加到一个新的stage中

- 提交stage

​    和拆分的道理类似，也是提交finalStage

​    做一个遍历：判断该stage是否有未提交的父stage

​        有：先提交父stage

​        无：提交finalStage



executor接收到task后做了什么？

-  TaskRunner封装反序列后的task，将TaskRunne提交给threadPool运行
-   在多线程的run方法中根据task具体的类型来执行 runTask(context)
   -   ShuffleMapStage => ShuffleMapTask: 读数据 => flatMap => map => Shuffle Write
   -  ResultStage => ResultTask: Shuffle Read => reduceByKey => collect

### 运行脚本

```shell
# yarn cluster
spark-submit \
--class org.apache.spark.examples.SparkPi \
--master yarn \
--deploy-mode cluster \
--driver-memory 1g \
--executor-memory 2g \
--executor-cores 1 \
--queue thequeue \
$SPARK_HOME/examples/jars/spark-examples*.jar 10

# yarn client
## verbose 可以看到比较多信息
spark-submit \
--class org.apache.spark.examples.SparkPi \
--master yarn \
--verbose \
--deploy-mode client \
--driver-memory 1g \
--executor-memory 2g \
--executor-cores 1 \
--queue thequeue \
$SPARK_HOME/examples/jars/spark-examples*.jar 10
```

- pom

```xml
<dependency>
  <groupId>org.apache.spark</groupId>
  <artifactId>spark-yarn_${scala.tools.version}</artifactId>
  <version>${spark.version}</version>
</dependency>
```



## [RDD DF DS](https://databricks.com/blog/2016/07/14/a-tale-of-three-apache-spark-apis-rdds-dataframes-and-datasets.html)

- 开发过程中：绝大部分场景都是后面两个
- RDD：
  - 单RDD ：value 、kv
  - 多RDD：union
  - 分区调整
  - 编程是灵活

- DF  =   RDD + Schema
  - 数据  +  Schema ==> Table
  - 1.3版本推出
  - 底层优化Catalyst、Tungsten（钨丝计划 Unsafe Row）
  - DataFrame = Dataset[Row]
- Spark SQL愿景
  - 写更少的代码
  - 读更少的数据
  - 优化交给底层的执行引擎去处理  org.apache.spark.sql.catalyst.rules.Rule

```sql
# spark-shell --master local[2]
spark.sql("select * from test.emp").explain(true)

# 执行计划
== Parsed Logical Plan ==
'Project [*]
+- 'UnresolvedRelation [test, emp], [], false

== Analyzed Logical Plan ==
empno: int, ename: string, job: string, mgr: string, hiredate: string, sal: double, comm: double, deptno: int
Project [empno#49, ename#50, job#51, mgr#52, hiredate#53, sal#54, comm#55, deptno#56]
+- SubqueryAlias spark_catalog.test.emp
   +- HiveTableRelation [`test`.`emp`, org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe, Data Cols: [empno#49, ename#50, job#51, mgr#52, hiredate#53, sal#54, comm#55, deptno#56], Partition Cols: []]

== Optimized Logical Plan ==
HiveTableRelation [`test`.`emp`, org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe, Data Cols: [empno#49, ename#50, job#51, mgr#52, hiredate#53, sal#54, comm#55, deptno#56], Partition Cols: []]

== Physical Plan ==
Scan hive test.emp [empno#49, ename#50, job#51, mgr#52, hiredate#53, sal#54, comm#55, deptno#56], HiveTableRelation [`test`.`emp`, org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe, Data Cols: [empno#49, ename#50, job#51, mgr#52, hiredate#53, sal#54, comm#55, deptno#56], Partition Cols: []]
```



## OOM（067）

- Driver端

  - 作业调度（DAG拆分） 参与计算可能性不大

    ```scala
    collect  
    show
    sc.parallelize等创建DataFrame
    ThriftServer 拉取计算结果，需要调大Driver内存
    ```

- Executor端

  - org.apache.spark.memory.MemoryManager
  - 数据倾斜
    - join  ==> map join union
    - group by  => 1个拆 2个stage
    - count(distinct x)  =>  distinct子查询



smb  => join sort by cluster by(桶与桶的join)



左边随机   右边扩容
1_a
6_a
5_a
4_a
2_a
3_a

a ==> 0_a   1_a ....   9_a
a
a
a

## 动态分区裁剪 动态扩容

[SPARK-11150](https://issues.apache.org/jira/browse/SPARK-11150)



## 如何保证exactly-once

1. source

2. transform
3. sink

从Kafka可以保证source，transform天生可以保证，从而关键点在sink，保证sink有2中方式：事务和幂等性

- 事务MySQL可以
- HBase、Redis、ES 可以保证幂等性（upsert、docId）





# Spark3.0新版本[#](https://spark.apache.org/releases/spark-release-3-0-0.html)

- Adaptive Query Execution ([SPARK-31412](https://issues.apache.org/jira/browse/SPARK-31412))
- Dynamic Partition Pruning ([SPARK-11150](https://issues.apache.org/jira/browse/SPARK-11150))
- Redesigned pandas UDF API with type hints ([SPARK-28264](https://issues.apache.org/jira/browse/SPARK-28264))
- Structured Streaming UI ([SPARK-29543](https://issues.apache.org/jira/browse/SPARK-29543))
- Catalog plugin API ([SPARK-31121](https://issues.apache.org/jira/browse/SPARK-31121))
- Java 11 support ([SPARK-24417](https://issues.apache.org/jira/browse/SPARK-24417))
- Hadoop 3 support ([SPARK-23534](https://issues.apache.org/jira/browse/SPARK-23534))
- Better ANSI SQL compatibility

**Extensibility Enhancements**

- Data source V2 API refactoring ([SPARK-25390](https://issues.apache.org/jira/browse/SPARK-25390))
- Hive 3.0 and 3.1 metastore support ([SPARK-27970](https://issues.apache.org/jira/browse/SPARK-27970), [SPARK-24360](https://issues.apache.org/jira/browse/SPARK-24360))

- Built-in source migration using DSV2: parquet, ORC, CSV, JSON, Kafka, Text, Avro ([SPARK-27589](https://issues.apache.org/jira/browse/SPARK-27589))

## csv

- 2.x中分隔符只能是单个字符，3.X可以多个字符[SPARK-24540](https://issues.apache.org/jira/browse/SPARK-24540)

- 递归查找文件[SPARK_27990](https://issues.apache.org/jira/browse/SPARK-27990)

## DataFrame

- tail
- org.apache.spark.sql.catalyst.analysis.FunctionRegistry
  - max_by/min_by

## AQE

[参考文章](https://blog.csdn.net/u013332124/article/details/90677676#Shuffle_partition_3)

```scala
// spark.sql.adaptive.skewJoin.enabled
val spark = SparkSession.builder()
  .appName(getClass.getSimpleName)
  .master("local[2]")
  .config("spark.sql.shuffle.partitions", "10")  // 2.X
  .config("spark.sql.adaptive.enabled", "true")  // 3.X  AQE  org.apache.spark.sql.internal.SQLConf
  .getOrCreate()

val df = spark.read.format("csv")
  .option("header", "true")
  .option("inferSchema", "true")
  .load("data/people.txt")

df.groupBy("id").count().show()

Thread.sleep(Int.MaxValue)
spark.stop()
```
