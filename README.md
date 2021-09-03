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

















