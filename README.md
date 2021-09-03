### 一、Hadoop

#### 1. 请说一下HDFS读写流程

* HDFS写流程

  1）client 客户端发送上传请求，通过RPC与namenade建立通信，namenode检查用户是否有有上传权限，以及上传的文件是否在hdfs上对应的目录重名，如果这两者任意一个不满足，则直接报错，如果两者都满足，则返回给客户端一个可以上传的信息。

  2）client根据文件的大小进行切分，默认128M一块，切分完成之后给namenode发送请求第一个block块上传到那些服务器上。

  3）namenode收到请求之后，根据网络拓扑、机架感知、副本机制进行文件分配，返回可用的DataNode的地址。（注：hadoop在设计时考虑到数据的安全与高效，数据文件默认在HDFS上存放三份，存储策略为本地一份，同机架内其他某一级店一份，不同机架的某一节点一份）

  4） 客户端收到地址之后，与服务器地址列表中的一个节点如A进行通信，本质上就是RPC调用，建立pipeline，A收到请求后会继续调用B，B在调用C，将整个pipeline建立完成，逐级返回client。

  5） client开始向A上发送第一个block（先从磁盘读取数据然后放到本地内存缓存），以packet(数据包，64kb)为单位，A收到一个packet就会发送给B， 然后B发送个C，A每传完一个packet就会放入一个应答队列等待应答。

  6）数据被分割成一个个的packet数据包在pipeline上依次传输，在pipeline反向传输中，逐个发送ack（命令正确应答），最终由pipeline中第一个DataNode节点A将pipeline ack发给client。

  7） 当一个block传输完成之后，Client再次请求NameNode上传第二个block，namenode重新选择三台DataNode给client。

  

* HDFS读流程

  1） client向namenode发送RPC请求，请求文件block的位置。

  2）namenode收到请求之后会检查用户权限以及是否有个文件，如果都符合，则会视情况返回部分或者全部的block列表，对于每个block，NameNode都会含有该block副本的DataNode地址，这些返回的DataNode地址，会按照集群拓扑得出DataNode与客户端的距离，然后进行排序，排序有两个规则：网络拓扑结构中距离client近的靠前；心跳机制中超时汇报的DataNode状态为stale,这样的靠后。

  3）client选择排序靠前的DataNode来读取block,如果客户端本身就是DataNode，那么将从本地直接获取数据（短路径读取特性）

  4） 底层上本质是建立Socket Stream （FSDataInputStream）,重复的调用父类DataInputStream的read方法，直到这个块的数据读取完毕。

  5） 当读完列表的block后，若文件读取还没结束，客户端会继续向NameNode获取下一批的block列表。

  6）读取完一个block都会进行checkSum验证，如果读取DataNode时出现错误，客户端会通知NameNode,然后再从一个拥有该block副本的DataNode继续读。

  7）read方法是并行的读取block信息，不是一块一块的读取，NameNode只是返回client请求包含的块的DataNode地址，并不是返回块的数据。

  8） 最终读取所有的block会合并成一个完整的最终文件。

  

#### 2. HDFS在读取文件的时候，如果其中一个块突然损坏了怎么办？

客户端读取完DataNode上的块之后会进行checksum验证，也就是客户端读取本地的块与HDFS上的原始块进行校验，如果发现校验结果不一致，客户端会通知NameNode，然后再从下一个拥有该block的DataNoe继续读。

#### 3. HDFS在上传文件的时候，如果其中一个DataNode突然挂掉怎么办？

客户端上传文件时与DataNode建立了pipeline管道，管道正向是客户端向DataNode发送数据包，管道反向是DataNode向客户端发送ack确认，也就是正确接收到数据包之后发送一个已确认接收的应答，当DataNode突然挂掉，客户端接收不到这个DataNode的ack确认，客户端会通知NameNode，NameNode检查该块的副本与规定的不符，就会通知DataNode去复制副本，并将挂掉的DataNode作下线处理，不再让它参与文件上传与下载。

#### 4. NameNode在启动的时候会做那些操作？

NameNode数据存储在内存和本地的磁盘，本地磁盘数据存储在fsimage镜像文件和edits编辑日志文件。

* 首次启动NomeNode

  1. 格式化文件系统，为了生成fsimage镜像文件。

  2. 启动NameNode

     1）读取fsimage文件，将文件内容加载到内存

     2）等待DataNode注册与发送block report

  3. 启动DataNade

     1）向NameNode注册

     2）发送block report

     3）检查fsimage中记录的块的数量和block report中块的总数是否相同

  4. 对文件系统进行操作（创建目录、上传文件、删除文件等）

     此时内存已经有文件系统改变的信息，但是磁盘中没有文件系统改变的信息，此时会将这些信息写到edits文件中，edits文件中存储的是文件系统元数据改变的信息。

* 第二次启动NameNode

  1. 读取fsimage和edits文件
  2. 将fsimage和edits文件合并成新的fsimge文件
  3. 创建新的edits文件，内容为空。
  4. 启动DataNode

#### 5. Secondary NameNode了解吗，它的工作机制是怎样的?

Secondary NameNode是合并NameNode的Edit logs 到fsimage文件中；

它的具体工作机制：

1）Secondary NameNode 询问nameNode是否需要checkpoint。直接带回NameNode是否检查结果。

2）Secondary NameNode 请求执行checkpoint

3）NameNode开始滚动正在写的edits文件日志

4）将滚动前的编辑日志和镜像文件拷贝到Secondary NameNode

5） Secondary NameNode加载编辑日志和镜像文件到内存，并合并

6） 生成新的镜像文件fsimage.chkpoint

7） 拷贝fsimage.chkpoint文件到NameNode

8）nameNode将fsimage.chkopoint重新命名成fsimage

所以如果Name中的元数据丢失，是可以从SecondarynameNode恢复一部分元数据信息的，但是不是全部，因为namenode正在写的edits日志还没拷贝到secondary nameNode，这部分恢复不了。

#### 6. Secondary NameNode 不能恢复NameNode的全部数据，那如何保证NameNode的数据存储安全？

这个问题就要说到Name的高可用了，即NameNode HA

一个NameNode是有单点故障问题，那就配置双NameNode，配置有两个关键点，一个必须要保证这两个NN的元数据必须同步，而是一个NN挂掉之后，另一个要立马补上。

1. 元数据信息同步在HA方案采用的是“共享存储”。每次写文件时，需要将日志同步谢傲共享存储，这个步骤成功才认定写文件成功。然后备份节点定期从共享同步日志，以便主备切换。

2. 监控NN状态采用Zookeeper，这两个NN节点的状态存放在ZK中，另外两个节点分别有一个进程监控程序，实施读取Zk中NN中的NN状态，来判断当前的NN是不是已经down机。如果standy的NN节点的ZkFC发现主节点已经挂掉，name就会强制给原本的active NN节点发送强制关闭请求，之后将备用的NN设置为active。

3. 如果面试官再问HA中共享存储是怎么实现的？

   可以解释：NameNode共享存储的方案有很多 ，比如Linux HA, VMare FT, QJM等，目前社区cloudera公司实现了基于QJM的方案合并HDFS的trunk之中并且作为默认的共享存储实现。

   基于QJM的共享存储系统只要用于Editlog，并不保存FSImage文件。FSImage文件还是在NameNode的本地磁盘中，QJM共享存储的基本思想来自于Paxos算法，采用多个称为JournalNode的节点组成，来集群存储editlog，每个journalNode保存同样的Editlog副本，每次NameNode写Editlog的时候，除了向本地磁盘写入Editlog之外，也会并行的向JournalNode节点集群中的每一个JNode发送写请求，只要大多数majority的jNode节点返回成功就任务jNode集群写入了editlog成功。如果有2N+1台JNode,那么根据大多数的原则，最多可以容忍有N台jNode节点挂掉。

#### 7. 在NameNodeHA中会有脑裂问题吗？怎么解决？

脑裂问题对于NameNode这类数据一致性要求是非常高的系统是灾难性的，数据会发生错乱并且无法恢复。zookeeper社区对这种问题的解决方法叫做fencing，中问翻译为隔离，也就是想办法把旧的active NameNode隔离起来，使它不能正常对外提供服务。

在进行fencing的时候，会执行以下操作：

1. 首先尝试调用这个旧ActiveNameNode的HAServiceProtocalRPC接口的transitionToStandby方法，看能不能把它转换成standy状态。
2. 如果transitionToStandby方法调用失败，那么就执行Hadoop配置文件中预定义的隔离措施，Hadoop目前主要提供两种隔离措施，通常会选择sshfence;
   * ssh fence： 通过ssh登录到目标机器，执行命令fuser将对应的进程杀死。
   * shell fence: 执行一个用户自定义的shell脚本来将进程隔离。

#### 8. 小文件过多会有什么危害，如何避免？

Hadoop 上大量HDFS元数据信息存储在NameNode内存中，因此过多的小文件必定会压垮NameNode的内存。

每个元数据约占150byte，所以如果一千万个小文件，每个文件占用一个block，则NameNode大约需要2G空间，如果存储1亿个文件，则nameNode需要20G空间。

显而易见，解决这个问题就是合并小文件，可以选择在客户端上传时执行一定的策略先合并，或者使用Hadoop的comebineFileInputFormat<K,v>实现小文件合并。

#### 9. 请说下HDFS的组织架构

* client 客户端

  切分文件，文件上传到HDFS的时候，client将文件切分成一个一个Block，然后进行存储

  与NameNode交互，获取文件位置信息

  与DataNode交互，读取或者写入文件

  client提供一些命令来管理HDFS，比如启动关闭HDFS/访问HDFS目录及内容等。

* NameNode： 名称节点，也称主节点，存储数据的元数据信息，不存储具体的数据

  管理HDFS的名称空间

  管理数据块（block）映射信息

  配置副本策略

  处理客户端读写请求

* DataNode: 数据节点，也称从节点。NameNode下达命令，DataNode执行实际的操作。

  存储实际的数据块

  执行数据的读、写操作

* Secondary NameNode ： 并非NameNode的热备，当NameNode挂掉的时候，它并不能马上替换NameNode并提供服务

  辅助NameNode，分担其工作量

  定期合并Fsimage和推送给NomeNode

  在紧急情况下，可以辅助恢复NameNode.



#### 10. 请说一下MR中MapTask 的工作机制。

简单概述：

inputFile 通过split 被分割多个split文件，通过Record按行读取内容给Map(自己写的处理逻辑的方法)，数据被map处理完之后交给OutPutcollect收集器，对结果进行分区（默认使用hashpartitoner），然后写入到buffer,每个map task 都有一个内存缓冲区（环形缓冲区），存放着map的输出结果，当缓冲区快满的时候，将缓冲区的数据以一个临时文件的方式溢写到磁盘，当整个map task 结束后再对磁盘中的 map task 产生的所有临时文件做合并，生成最终的正式文件，然后等待reduce task 拉取。

详细步骤：

1. 读取数据组件InputFormat（默认 textInputFormat） 会通过getSplits 方法对输入目录的文件逻辑切片规划得到block，有多少个block就启动多少个MapTask.
2. 将输入的文件切分为block之后，由RecordReader对象（默认是LineRecordReader）进行读取，以\n作为分割符，读取一行数据，返回<key, value>。key 表示的是每行首字符偏移值，value表示这一行文本内容。
3. 读取block返回<key, value> ,进入到用户自己继承的Mapper类中，执行用户重写的map函数，RecordReader读取一行这里调用一次。
4. Mapper逻辑结束之后，将Mapper的每一条结果通过context.write进行cllocet数据收集，在collect中，会先对其进行分区处理，默认使用hashpartioner。
5. 接下来，就会将数据写入到内存，内存中这片区域叫做环形缓冲区（默认100M），缓冲区的作用是批量收集Mapper结果，减少磁盘IO的影响，我们的key/value对以及Partion的结果都会写到缓冲区，当然写入之前key与value值都会被系列化成字节数组。
6. 当环形缓冲区的数据导到溢写比例（默认0.8）,也就是80M,溢写线程启动，需要将80M空间内的key做排序，排序是MapReduce模型默认的行为。这里额排序是对序列化的自己做排序。
7. 合并溢写文件，每次溢写会在磁盘上生成一个临时文件（写之前判断是否有Combiner），如果Mapper的输出结果真的很大，有很多次的溢写发生，磁盘上相应的就会有有很多临时文件存在，当整个数据处理结束之后开始对磁盘中的临时文件进行Merge合并，因为最终的文件只有一个，写入磁盘，并且为这个文件提供索引文件，以记录每个reduce对应的偏移量。

#### 11. 请说一下MR中Reduce Task 的工作机制。

简单描述：

reduce 大致分为copy/sort/reduce 三个阶段，重点在前两个阶段。copy阶段包含一个eventFetcher来获取已经完成的map列表，由Fetcher线程去copy数据，在此过程中会启动两个merge线程，分别为inMemoryMerger 和 onDiskMerger，分别将内存中的数据merge到磁盘和将磁盘的数据进行merge。待数据copy完成之后，copy 阶段就完成了，开始进行sort阶段，sortj阶段主要是执行finalMerge操作，纯粹的sort阶段，完成之后，就是reduce阶段，调用用户定义的reduce函数进行处理。

详细步骤：

1. Copy 阶段，简单地拉取数据，redcue进程启动一些数据copy线程（fecher）,通过Http方式请求maptask 获取属于自己的文件（map task 的分区会标识每个map task 属于那个 reduce task ，默认reduce task 的标0开始）
2. Merge阶段： 这里的merge如map端的merge动作，只是数组中存放的是不同map端copy 来的数值。copy 过来的数据会先放入内存缓冲区中，这里的缓冲区大小要被map端更为灵活。merge有三种形式：内存到内存，内存到磁盘，磁盘到磁盘。默认情况下第一种形式不会启用。当内存中的数据量达到一定阀值，就启动内存到磁盘的merge。与map端类似，这是溢写的过程，这个过程中，如果你设置了combiner,也是会启用的，然后在磁盘中生产众多溢写文件。第二种merge方式一直在运行。直到map端的数据时才结束，然后启动第三种磁盘到磁盘的merge方式生成最终文件。
3. 合并排序： 把分散的数据合并成一个大数据后，还会再对合并的数据排序。
4. 对排序后的键值对调用reduce方法，键相等的键值对调用一次reduce方法，每次调用会产生零个或者多个键值对，最后把这些键值对写到HDFS文件中。

#### 12. 请说一下MR中的shuffle阶段

Shuffle阶段分为四个步骤：依次为：

分区、排序、规约、分组

其中前三个步骤在map阶段完成，最后一个步骤在reduce阶段完成。

shuffle是MapReduce的核心，它分布在Mapreduce的map阶段和reduce阶段。一般把从map产生输开始到Reduce取得数据作为输入之前的过程称为shuffle。

1. Collect阶段： 将MapTask的结果输出到默认大小的100M的环形缓冲区，保存的是key/value，parition分区信息等。

2. split阶段：当内存中的数据量达到一定的阀值的时候，就会将数据写到本地磁盘，再将数据写入到磁盘之前需要对数据进行一次排序操作，如果配置了combiner，还会将有相同分区号和key的数据进行排序。

3. Merge阶段：把所有溢出的临时文件进行一次合并操作，以确保一个MapTask最终只产生一个中间数据文件。

4. copy阶段：ReduceTask启动Fetcher线程到已经完成maptask的节点复制一份属于自己的数据，这些数据默认会保存在内存的缓冲区中，当内存的缓冲区达到一定的阀值，就会将数据写到磁盘中。

5. Merge阶段：在ReduceTask远程复制数据的同时，会在后台开启两个线程对内存到本地的数据文件进行合并操作。

6. Sort阶段：在对数据进行合并的同时，会进行排序操作，由于MapTask 阶段已经对数据进行局部的排序，Reducetask只需保证Copy的数据的最终整体有效性即可。

   注：shuffle中缓冲区大小会影响到mapreduce程序的执行效率，原则上说，缓冲区越大，磁盘的io次数也少，执行速度越快。

   缓冲区的大小可以通过参数调整，参数：mapreduce.task.io.sort.mb 默认为100M。

#### 13. shuffle阶段的数据压缩机制了解吗？

在shuffle阶段中，可以看到数据的大量拷贝，从map阶段输出的数据，都要通过网络拷贝，发送到reduce阶段，这个一个过程中，涉及到大量的网络IO,如果数据能够进行压缩，那么数据的发送量就会少的多。

hadoop当中支持压缩算法：

gzip、bzip2、LZO、Lz4、snappy,这几种压缩算法综合压缩和压缩速率，谷歌中的snappy是最优的，一般都选择snappy压缩。

#### 14. 在写MR时，什么情况下可以使用规约？

规约(combiner)是不能够影响任务的运行结果的，局部汇总，适用于求和类，不适用于求平均值，如果reduce的输入参数类型和输出类型参数类型是一样的，则规约的类可以使用reduce类，只需要在驱动类中指明规约的类即可。

#### 15. yarn集群的架构和工作原理知道多少？

yarn 的基本设计思想是将MapReduceV1中JobTracker拆分为两个独立的服务：

ResourceManager 和 ApplicationMaster.

ResouceManager 负责整个系统的资源管理和分配，ApplicationMaster负责单个应用程序的管理。

1. ResouceManager:

   RM是一个全局的资源管理器，负责整个系统的资源管理和分配，它主要由两个部分组成：调度器（Scheduler）和应用程序管理器（Application Manager）。调度器根据容量、队列等限制条件，将系统中的资源分配给正在运行的应用程序，在保证容量、公平性和服务等级的前提下，优化了集群资源利用率，让所有的资源都被充分利用应用程序管理器负责整个系统中的所有的应用程序，包括应用程序的提交、与调度器协商资源启动ApplicationMaster、监控ApplictionMaster运行状态并在失败的时候重启它。

2. ApplictionMaster：

   用户提交的一个应用程序会对一个ApplicationMaster,他的主要功能有：

   与RM调度器协商获得资源，资源以container表示。

   将得到的任务进行一部分配给内部任务。

   与NM通信以启动、停止任务。

   监控所有的内部任务状态，并在任务运行失败的时候重新为任务申请资源以重启任务。

3. NodeManger:

   NodeManger 是每个节点上的资源和任务管理器，一方面，他会定期的向RM汇报本地点上的资源使用情况和各个Container的运行状态；l另一方面，它接收并处理来自AM的Container的启动和停止请求。

4. Container:

   Container是Yarn中资源抽象，封装了各种资源，一个应用程序会分配一个Container，这个应用程序只能使用这个Container中描述的资源。

   不同于mapReduceV1中槽位slot的资源封装，Container是一个动态资源的划分单位，更能充分利用资源。

#### 16. yarn的任务提交流程是怎样的？

当JobClient向yarn提交一个应用程序后，yarn将分成两个阶段运行这个应用程序：一是启动applicationmaster;第二个阶段是由ApplictionMaster创建程序，为它申请资源，监控运行知道结束。

1. 用户向yarn提交一个应用程序，并指定Application程序、启动ApplicationMaster的命令、用于程序。
2. RM为这个应用程序分配第一个Container，并与之对应的NM通讯，要求他在这个Container中启动应用程序ApplicationMaster。
3. ApplicationMaster向RM注册，然后拆分为内部各个子任务，为各个内部任务申请资源，并监控这些任务的运行，知道结束。
4. AM采用轮询的方式向RM申请和领取资源
5. RM申请到资源后，便与之对应的NM通讯，要求NM启动任务。
6. NodeManager为任务设置号运行环境，将任务启动命令写到一个脚本中，并通过运行这个脚本启动任务。
7. 各个任务向AM汇报自己的状态和进度，以便任务失败时可以重启任务。
8. 应用程序完成之后，applicationMaster向ResoureManager注销关闭自己。

#### 17. yarn资源调度三种模型了解吗？

在yarn有三种调度器可以选择：FIFO scheduler、Capacity Scheduler、Fair scheduler。

apache版本的hadoop默认使用的是capacity调度方式，CDH版本默认使用的是fair scheduler调度方式。

FIFO Scheduler（先来先服务）

FIFO Scheduler把应用按提交的顺序排成一个队列，这是一个先进先出的队列，在进行资源分配的时候，先给队列中最头上的应用进行分配资源，待最头上的应用需求满足后，在给下一个分配，以此类推。

FIFO Scheduler是最简单的调度器，也不需要任何配置，但是它并不适用共享集群。大的应用可能会占用所有的集群资源，这就导致其他应用被阻塞，比如有一个大任务执行，占了全部的资源，再提交一个小任务，则此小任务一直被阻塞。

Capacity Scheduler （能力调度器）：

对于Capacity 调度器，有一个专门的队列用于运行小任务，但是为小任务专门设置一个队列会预先占用一定的集群资源，这就导致大任务的执行时间会落后于FIFO调度器时间。

Fair scheduler （公平调度器）：

在Fair调度器中，我们不需要预先占用一定的系统资源，Fair调度器会为所有运行的Job动态的调整系统资源。

比如：当第一个Job提交的时候，只有一个job在运行，此时它获取了所有的集群资源，当第二小任务提交后，Fair 调度器会分配一半资源给这个小任务，让这两个任务公平的共享集群资源。

需要注意的是，在Fair的调度器中，第二个任务提交到资源会有一定的延迟，因为它需要等待第一个任务释放占用的Container。小任务执行完成之后也会释放自己占用的资源。大任务又获得了全部的系统资源。最终的效果的就是Fair调度器得到了高的资源利用率又能保证小任务的及时完成。

### 二、Hive

#### 1. hive内部表和外部表区别

未被extenal修饰的是内部表（managed table）, 被external 修饰的为外部表（external table）

区别：

1） 内部表数据由Hive自身管理，外部表数据由HDFS管理；

2） 内部表数据存储的位置是hive.metastore.warehouse.dir（默认：/user/hive/warehouse）,外部表数据存储位置由自己制定（如果没有Location , hive 将在HDFS上的/user/hive/ware/house文件夹下以外表的表名创建一个文件夹，并将属于这个表的数据存放到这里）；

3） 删除内部表会直接删除元数据（metadata）及存储数据；删除外部表仅仅会删除元数据，HDFS上的文件并不会被删除；

#### 2. hive有索引吗？

Hive支持索引，但是Hive的索引与关系型数据库中的索引并不相同，比如：Hive不支持主键或者外键。

Hive索引可以建立在表中的某些列上，以提升一些操作效率，例如减少MapReduce任务中需要读取的数据块的数量。

在可以预见到分区数据非常庞大的情况下，索引常常是优于分区的。

虽然hive 并不像事物数据库那样针对个表的行来执行查询、更新、删除操作等，它更多是用在任务节点场景下，快速的全表扫描大规模数据。但是在某些场景下，建立索引还是可以提高hive表指定的查询速度。

适用场景：

适用于不更新的静态字段，以免总是重建索引数据，每次建立、更新数据后，都要重建索引以构建索引表。

hive索引机制如下：

hive 在指定列上建立索引，会产生一张索引表（hive的一张物理表），里面的字段包括，索引列的值、该值对应的HDFS文件路径、该值在文件中的偏移量；

v0.8后引入bitmap索引处理器，这个处理器适用于排重后，值较少的列（例如，某字段的取值只能是几个枚举值），因为索引是用空间换时间，索引列的取值过多会到值bitmap索引表过大。

但是，很少遇到hive用索引的。说明还是与缺陷的or不适合的地方。

#### 3. 运维如何对hive 进行调度

1. 将hive的sql定义 到脚本中
2. 使用azkaban 或者 oozie进行任务的调度
3. 监控任务调度页面



#### 4.  ORC、Parquet等列时存储优点。

ORC和Parquet都是高性能的存储方式，这两种格式都会带来存储和性能上的提升。

* parquet:

  1. parquet 支持嵌套的数据模型，类似于protocol buffers,每一个数据模型的schema包含多个字段，每个字段都有三个属性：重复次数、数据类型、字段名。

     重复次数可以是以下三种：required（只出现1次）， repeated（出现0次或者多次），optional（出现0次或1次）。每一个字段的数据类型可以分成两种：

     group(复杂类型)和primitive(基本类型)。

  2. Parquet中没有Map、Array这样的复杂数据结构，但是可以通过repeated和group组合来实现的。

  3. 由于parquet支持的数据模型比较松散，可能一条记录中存在比较深的嵌套关系，如果每一条记录都维护一个类似的树状结可能会占用较大的存储空间，因此Dremal 论文中提出一种高效的对于嵌套数据格式的压缩算法，striping/assembly算法，parquet 可以使用较少的存储空间表示负责的嵌套格式。并且通常repetition level 和 Definition level 都是较小的整数数值，可以通过RLE算法对其进行压缩，进行一步降低村粗空间。

  4. parquet 文件是以二进制方式进行存储的，是不可能直接读取和修改的，parquet 文件自解析的，文件中包括该文件的数据和元数据。

* ORC:

  1. ORC 文件是自描述的，它的元数据使用protocol buffers序列化，并且文件中的数据尽可能的压缩以减低存储空间的消耗。
  2. 和parquet类似，ORC文件也是以二进制方式存储，所以是不可以直接读取，ORC文件也是自解析的，它包括许多元数据，这些元数据都是同构ProtoBuffer进行序列化的。
  3. ORC会尽可能合并多个离散的区间尽可能的减少I/O次数。
  4. ORC中使用了更加精确索引信息，使得在读取数据时可以指定从任意一行后开始读取，更细粒度的统计信息使得读取ORC文件跳过整个ROW group , orc 默认会对任何一块数据和索引信息使用ZLIB压缩，因此ORC文件占用的存储空间也更小。
  5. 在型的ORC中也入了Bloom Filter 的支持，他可以进一步提升谓词下推的效率，在Hive 1.2.0版本以后也加入了对此的支持。

#### 5. 数据建模用的那些模型？

**星型模型**

**雪花模型**

**星座模型**

#### 6. 为什么要对数据仓库分层？

* 用空间换时间，通过大量的预处理来提升应用的系统的用户体验（效率）， 因此数据仓库会存在大量冗余数据。
* 如果不分层的话，如果业务系统的业务规则发生变化将会影响整个数据清洗过程，工作量巨大。
* 通过数据分层管理可以简化数据清洗的过程，因为把原来一步的工作分到了多个步骤去完成，相当于把一个复杂的工作拆分成了多个简单的工作，把一个大的黑盒变成了白盒，每一层处理逻辑都相对简单和容易理解，这样我们比较容易保证每一步骤的正确型，当数据发生错误的时候，往往我们只需要局部调整步骤即可。

#### 7. 使用过Hive解析JSON串吗？

* hive处理json数据总体有两个方向走
  1. 将json以字符串的方式整个入Hive表，然后通过使用UDF函数解析已经导入的hive中的数据，比如使用lateral view json_tuple的方法，获取所需要的列名。
  2. 在导入之前将json拆成各个字段，导入Hive标的数据已经解析过的。这将要使用第三方工具serDe.

#### 8. sort by  和 order by 的区别

order by 会对输入做全局排序，因此只有一个reducer(多个reducer无法保证全局有序)，只有一个reducer，会导致当前输入的规模较大时，需要较长的计算时间。

sort by  不是全局排序，其数据进入到reducer前完成排序。因此，如果用sort by 进行排序，并且设置mapred。reduce.tasks > 1, 则sort by 只保证每个reudcer的输出有序，不保证全局有序。

#### 9. 怎么排查是哪里出现了数据倾斜

#### 10. 数据侵袭怎么解决

#### 11. Hive 小文件过多怎么解决

#### 12. hive优化有那些？

**数据存储及压缩**：

针对hive中表的存储格式通常有ORC 和 parquet, 压缩格式一般使用snappy。相比与textfile格式表，orc占有更少的存储，因为hive底层使用MR计算架构，数据流是hdfs到磁盘再到hdfs，而且会有很多次，所以使用orc数据格式和snappy压缩策略可以降低IO读写，还能降低网络传输量，这样一定程度上可以省存储，还能提升hql任务执行效率。

**通过调参优化**

并行执行，调节parallel参数；

调节jvm参数，重用jvm;

设置map/reduce 的参数，开启strict mode 模式

关闭推测执行设置。

**有效地减小数据集，将大表拆分成子表，结合外部表和分区表**

**SQL 优化**

大表对大表：尽量减少数据集，可以通过分区表，避免扫描全表字段；

大表对小表：设置自动识别小表，将小表放入到内存中去执行。















