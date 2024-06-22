# hadoop

## 1. HDFS

### 1.1 GFS（The Google File System）
GFS定了三个非常重要的设计原则，这三个原则带来了很多和传统的分布式系统研究大相径庭的设计决策。但是这三个原则又带来了大量工程上的实用性，使得GFS的设计思路后续被Hadoop这样的系统快速借鉴并予以实现

![GFS设计原则.png](GFS设计原则.png)

#### 1.1.1 GFS读流程
![目录服务.png](目录服务.png)

每个data会被切分成54M的block，三副本，存放到不同的chuck

客户端去读取GFS里面的数据的时候，有以下三个步骤：
1. 先去问master想要读取的数据在哪里。客户端会发出两部分信息，一个是文件名，另一个则是要读取哪一段数据，也就是读取文件的offset及length
2. master拿到了这个请求之后，就会把这个chunk对应的所有副本所在的chunk server，告诉客户端
3. 等客户端拿到chunk所在的chunk server信息后，客户端就可以直接去找其中任意的一个chunk server读取自己所要的数据

master其实就是一个目录服务，本身不存储数据，而是只是存储目录这样的元数据

#### 1.1.2 高可用(High Availability)--快速恢复性和可用性保障

master节点的所有数据，保存在内存里才能跟得上几百个客户端的并发访问，但是放在内存里的问题，就是一旦master挂掉，数据就会都丢了。所以，master会通过记录操作日志和定期生成对应的Checkpoints进行持久化

- master节点重启的时候

  就会先读取最新的Checkpoints，然后重放Checkpoints之后的操作日志，把master节点的状态恢复到之前最新的状态
- master节点的硬件彻底故障

  Backup Master + Shadow Master

![master故障.png](master故障.png)

Backup Master是**同步复制**的，而Shadow Master的数据，要到t_2时间点才会追上来

在集群外部还有监控master的服务在运行。如果只是master的进程挂掉了，那么这个监控程序会立刻重启master。如果master所在的硬件或者硬盘出现损坏，那么监控程序就会在Backup Master里面找一个出来，变成新的master

- 快速恢复

  为了让集群中的其他chunk server以及客户端不感知变化，GFS通过一个Canonical Name指定master，而不是通过IP

#### 1.1.3 异步复制
从监控程序发现master节点故障、启动备份节点上的master进程、读取Checkpoints和操作日志，仍然是一个几秒级别乃至分钟级别的过程

shadow master可以用于保障“读场景”，shadow Master不断同步master输入的写入，尽可能保持追上master的最新状态，而并不需要等到shadow Master也写入完成才返回成功

但是可能因为延时读到一些过时的master里面的信息，比如命名空间、文件名等这些元数据，只会发生在以下三个条件都满足的情况下：
1. master挂掉了
2. 挂掉的master或者 Backup Master上的Checkpoints和操作日志，还没有被影子Master同步完
3. 要读取的内容，恰恰是在没有同步完的那部分操作上

#### 1.1.4 减少网络带宽

- GFS写流程（2PC?）
  ![GFS写.png](GFS写.png)

  1. 客户端会去问master要写入的数据，应该在哪些chunkserver上
  2. 和读数据一样，master会告诉客户端所有的次副本（secondary replica）所在的chunkserver。这还不够，master还会告诉客户端哪个replica是主副本（primary replica），数据此时以它为准。
  3. 拿到数据应该写到哪些chunkserver里之后，客户端会把要写的数据发给所有的replica。不过此时，chunkserver 拿到发过来的数据后还不会真的写下来，只会把数据放在一个LRU的缓冲区里
  4. 等到所有次副本都接收完数据后，客户端就会发送一个写请求给到主副本。GFS面对的是几百个并发的客户端，所以主副本可能会收到很多个客户端的写入请求。主副本自己会给这些请求排一个顺序，确保所有的数据写入是有一个固定顺序的。接下来，主副本就开始按照这个顺序，把刚才LRU的缓冲区里的数据写到实际的chunk里去。
  5. 主副本会把对应的写请求转发给所有的次副本，所有次副本会和主副本以同样的数据写入顺序，把数据写入到硬盘上
  6. 次副本的数据写入完成之后，会回复主副本，我也把数据和你一样写完了
  7. 主副本再去告诉客户端，这个数据写入成功了。而如果在任何一个副本写入数据的过程中出错了，这个出错都会告诉客户端，也就意味着这次写入其实失败了

- 分离控制流和数据流
- 流水线式的网络数据传输
- 独特的Snapshot操作

## 2. MR计算框架

![mapreduce.png](mapreduce.png)

- buffer in memory

  防止频繁的写数据带来的系统调用，默认100M

- merge on disk

  为了reduce只拉取某一块的数据
- reduce merge

  reduce最后一次合并的结果作为输入到reduce函数

**计算如何向数据移动?**

![计算向数据移动.png](计算向数据移动.png)

hdfs暴露数据的位置：
1. 资源管理
2. 任务调度

角色： 这个是Hadoop 1.x维度的， 2.x开始由yarn替代

JobTracker:
  1. 资源管理
  2. 任务调度

TaskTracker:
1. 任务管理
2. 资源汇报

Client：**很重要的角色**，会做计算向数据移动的准备

1. 会根据每次的计算数据，向NameNode询问元数据，得到一个切片split的清单，map的数量就有了,split是逻辑的，block是物理的, block身上有(offset, locations),split和block是有映射关系，split包含偏移量以及对应的map任务应该移动到哪些节点(locations)
  
    | id      | file | offset | locations  |
    |---------|------|--------|------------|
    | split01 | A    | 0, 500 | n1, n3, n5 |

2. 生成计算程序未来运行时的相关配置文件,...xml，比如堆大小等等
3. 未来的移动应该相对可靠，client会将jar, split清单，配置xml上传到hdfs的目录中（上传的数据副本数是10，原因是map可能很多，需要向DataNode请求）
4. client会调用JobTracker，通知要启动一个计算程序，并告知文件都放在hdfs哪些地方
5. JobTracker会去做决定，哪个TaskTracker执行，是因为TaskTracker对JobTracker上报

JobTracker收到启动程序之后：
1. 从HDFS中取回split清单
2. 根据自己收到的TaskTracker汇报的资源，最终确定每一个split对应的map应该去哪一个节点
3. TaskTracker在心跳的时候会获取分配到自己的任务信息

TaskTracker在心跳取回任务之后
1. 从hdfs从下载jar，xml到本机
2. 最终启动任务描述中的,MapTask/ReduceTask，最终代码在某一个节点被启动，是通过client上传，TaskTracker下载


## yarn

yarn出现的原因：支持未来的框架复用资源管理; 因为各自实现资源管理，但是他们部署在同一批硬件上，因为隔离，所以不能感知对方的使用，造成**资源争抢**

![yarn_run_job.png](yarn_run_job.png)

- 模型

container容器，（不是docker）

架构：

![run_mapreduce_job.png](run_mapreduce_job.png)

## wc样例和源码分析

```Java
// 主函数
 public class MyWordCount {
    // bin/hadoop command [genericOptions] [commandOptions]
    // hadoop jar ooxx.jar ooxx -D ooxx=ooxx inpath outpath
    // args : 2类参数  genericOptions commandOptions
    public static void main(String[] args) throws Exception {

        Configuration conf = new Configuration(true);
        
        //工具类帮我们把-D 等等的属性直接set到conf，会留下commandOptions
        GenericOptionsParser parser = new GenericOptionsParser(conf, args);
        String[] othargs = parser.getRemainingArgs();

        //让框架知道是windows异构平台运行
        conf.set("mapreduce.app-submission.cross-platform","true");

        Job job = Job.getInstance(conf);

        // job.setInputFormatClass(ooxx.class);

        job.setJar("C:\\Users\\admin\\hadoop-hdfs-1.0-0.1.jar");

        //必须必须写的
        job.setJarByClass(MyWordCount.class);
        job.setJobName("wc");

        Path infile = new Path(othargs[0]);
        TextInputFormat.addInputPath(job, infile);

        Path outfile = new Path(othargs[1]);
        if (outfile.getFileSystem(conf).exists(outfile)) 
            outfile.getFileSystem(conf).delete(outfile, true);
        TextOutputFormat.setOutputPath(job, outfile);

        job.setMapperClass(MyMapper.class);
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(IntWritable.class);
        job.setReducerClass(MyReducer.class);

        // job.setNumReduceTasks(2);
        // Submit the job, then poll for progress until the job is complete
        job.waitForCompletion(true);
    }
}
```

```Java
public class MyMapper extends Mapper<Object, Text, Text, IntWritable> {
    // hadoop框架中，它是一个分布式  数据 ：序列化、反序列化
    // hadoop有自己一套可以序列化、反序列化
    // 或者自己开发类型必须：实现序列化，反序列化接口，实现比较器接口
    // 排序 -》  比较  这个世界有2种顺序：  8  11，    字典序、数值顺序

    private final static IntWritable one = new IntWritable(1);
    private Text word = new Text();

    //hello hadoop 1
    //hello hadoop 2
    //TextInputFormat
    //key 是每一行字符串自己第一个字节面向源文件的偏移量
    public void map(Object key, Text value, Context context) throws IOException, InterruptedException {
        StringTokenizer itr = new StringTokenizer(value.toString());
        while (itr.hasMoreTokens()) {
            word.set(itr.nextToken());
            context.write(word, one);
        }
    }
}
```

```Java
public class MyReducer extends Reducer<Text, IntWritable, Text, IntWritable>{
    private final IntWritable result = new IntWritable();
    //相同的key为一组 ，这一组数据调用一次reduce
    //hello 1
    //hello 1
    //hello 1
    //hello 1
    public void reduce(Text key, Iterable<IntWritable> values,
                       Context context) {
        int sum = 0;
        for (IntWritable val : values) {
            sum += val.get();
        }
        result.set(sum);
        context.write(key, result);
    }
}
```
提交方式
1. 开发-> jar -> 上传到集群中的某一个节点 -> bin/hadoop jar /usr/joe/wordcount.jar org.myorg.WordCount /usr/joe/wordcount/input /usr/joe/wordcount/output
2. 嵌入【linux，windows】（非hadoop jar）的集群方式  on yarn

    集群：M、R
    ```
    client -> RM -> AppMaster
    mapreduce.framework.name -> yarn   //决定了集群运行
    conf.set("mapreduce.app-submission.cross-platform","true");
    job.setJar("....hadoop-hdfs-1.0-0.1.jar");
    ```
3. local自测

源码的分析: what？why？how？

主要看3个毕竟重要的环节：
1. 计算向数据移动
2. 并行度、分治
3. 数据本地化读取

### 客户端

没有计算发生，但是很重要，因为支撑了计算向数据移动和计算的并行度

对于一个MapReduce任务来说，除去开始的加载配置项，最后最后肯定会提交任务
```Java
// MyWordCount.java
// Submit the job, then poll for progress until the job is complete
 job.waitForCompletion(true);

// Job.java waitForCompletion => submit
 public boolean waitForCompletion(boolean verbose) {
    if (state == JobState.DEFINE) {
    // 一定是异步的，不然后面也不好有监控
      submit();
    }
    if (verbose) {
      monitorAndPrintJob(); // 监控和打印job的细节
    } else { // 循环判断有没有结束
      // get the completion poll interval from the client.
      int completionPollIntervalMillis = 
        Job.getCompletionPollInterval(cluster.getConf());
      while (!isComplete()) {
        try {
          Thread.sleep(completionPollIntervalMillis);
        } catch (InterruptedException ie) {
        }
      }
    }
    return isSuccessful();
  }


// internal call          
public void submit() 
    ....
 
    final JobSubmitter submitter = 
    getJobSubmitter(cluster.getFileSystem(), cluster.getClient());

    status = ugi.doAs(new PrivilegedExceptionAction<JobStatus>() {
      public JobStatus run() throws IOException, InterruptedException, 
      ClassNotFoundException {
        return submitter.submitJobInternal(Job.this, cluster); // 提交作业
      }
    });
    ......
}
```
submitJobInternal主要会做如下几个步骤（源码注释）:

Internal method for submitting jobs to the system.

The job submission process involves:

- Checking the input and output specifications of the job. 检测输入输出路径
- Computing the InputSplits for the job. 有多少map是有split决定的，并行度和计算向数据移动就可以实现了
- Setup the requisite accounting information for the DistributedCache of the job, if necessary.
- Copying the job's jar and configuration to the map-reduce system directory on the distributed file-system.
- Submitting the job to the JobTracker and optionally monitoring it's status.

先看第二步Computing the InputSplits for the job.

![](split.png)

```Java
// Create the splits for the job
LOG.debug("Creating splits at " + jtFs.makeQualified(submitJobDir));
int maps = writeSplits(job, submitJobDir);

// writeSplits
List<InputSplit> splits = input.getSplits(job);
```
TextInputFormat的继承结构
![inputFormat.png](inputFormat.png)

MR框架默认的输入格式化类： TextInputFormat

因为Job的父类是JobContextImpl
```Java
public Class<? extends InputFormat<?,?>> getInputFormatClass() 
 throws ClassNotFoundException {
return (Class<? extends InputFormat<?,?>>) 
  conf.getClass(INPUT_FORMAT_CLASS_ATTR, TextInputFormat.class);
}
```

```Java
int writeNewSplits(JobContext job, Path jobSubmitDir) throws IOException,
  InterruptedException, ClassNotFoundException {
Configuration conf = job.getConfiguration();

// 默认是TextInputFormat
InputFormat<?, ?> input = 
  ReflectionUtils.newInstance(job.getInputFormatClass(), conf);

List<InputSplit> splits = input.getSplits(job);
T[] array = (T[]) splits.toArray(new InputSplit[splits.size()]);

// sort the splits into order based on size, so that the biggest
// go first
Arrays.sort(array, new SplitComparator());
JobSplitWriter.createSplitFiles(jobSubmitDir, conf, 
    jobSubmitDir.getFileSystem(conf), array);
return array.length;
}
```

```Java
  public List<InputSplit> getSplits(JobContext job) throws IOException {
    Stopwatch sw = new Stopwatch().start();
    // minSize,maxSize都是自己在conf配置的
    long minSize = Math.max(getFormatMinSplitSize(), getMinSplitSize(job));
    long maxSize = getMaxSplitSize(job);

    // generate splits
    List<InputSplit> splits = new ArrayList<InputSplit>();
    List<FileStatus> files = listStatus(job);
    for (FileStatus file: files) {
      Path path = file.getPath();
      long length = file.getLen();
      if (length != 0) {
        BlockLocation[] blkLocations;
        if (file instanceof LocatedFileStatus) {
          blkLocations = ((LocatedFileStatus) file).getBlockLocations();
        } else {
          FileSystem fs = path.getFileSystem(job.getConfiguration());
          
          // 这个重要：获取每个file的Block Location
          blkLocations = fs.getFileBlockLocations(file, 0, length); 
        }
        if (isSplitable(job, path)) {
          long blockSize = file.getBlockSize();
          // 切片split是一个窗口机制：调大split改小(minSize),调小split改大(maxSize)
          long splitSize = computeSplitSize(blockSize, minSize, maxSize);

          // 把文件切割成一个个split
          long bytesRemaining = length;
          while (((double) bytesRemaining)/splitSize > SPLIT_SLOP) {
            int blkIndex = getBlockIndex(blkLocations, length-bytesRemaining);
            splits.add(makeSplit(path, length-bytesRemaining, splitSize,
                        blkLocations[blkIndex].getHosts(),
                        blkLocations[blkIndex].getCachedHosts()));
            bytesRemaining -= splitSize;
          }
```

hosts: Split所在的位置，支撑的计算向数据移动，可以选择Split所在的空闲hosts进行计算
```Java
protected FileSplit makeSplit(Path file, long start, long length, 
                             String[] hosts, String[] inMemoryHosts) {
  return new FileSplit(file, start, length, hosts, inMemoryHosts);
}
```

### MapTask



我们自己只需要编写Map函数
```Java
public class MyMapper extends Mapper<Object, Text, Text, IntWritable> {
```

map函数是每条都会运行，而setup和cleanup只会运行一次
```Java
public class Mapper<KEYIN, VALUEIN, KEYOUT, VALUEOUT> {
  public void run(Context context) throws IOException, InterruptedException {
    setup(context);
    try {
      while (context.nextKeyValue()) {
        map(context.getCurrentKey(), context.getCurrentValue(), context);
      }
    } finally {
      cleanup(context);
    }
  }
```

MapTask是Map框架中任务的主类

```Java
  @Override
  public void run(final JobConf job, final TaskUmbilicalProtocol umbilical)
    throws IOException, ClassNotFoundException, InterruptedException {
    this.umbilical = umbilical;

    if (isMapTask()) {
      // If there are no reducers then there won't be any sort. Hence the map 
      // phase will govern the entire attempt's progress.
      // 只有map阶段，没有reduce阶段
      if (conf.getNumReduceTasks() == 0) {
        mapPhase = getProgress().addPhase("map", 1.0f);
      } 
      // 有reduce阶段，有map和sort
      else {
        // If there are reducers then the entire attempt's progress will be 
        // split between the map phase (67%) and the sort phase (33%).
        mapPhase = getProgress().addPhase("map", 0.667f);
        sortPhase  = getProgress().addPhase("sort", 0.333f);
      }
    }
    .......
    if (useNewApi) {
      runNewMapper(job, splitMetaInfo, umbilical, reporter);
    }
```

```Java
void runNewMapper(final JobConf job,
                    final TaskSplitIndex splitIndex,
                    final TaskUmbilicalProtocol umbilical,
                    TaskReporter reporter
                    ) throws IOException, ClassNotFoundException,
                             InterruptedException {
          // make a task context so we can get the classes
    org.apache.hadoop.mapreduce.TaskAttemptContext taskContext =
    // job里面包含了configration的信息
    new org.apache.hadoop.mapreduce.task.TaskAttemptContextImpl(job, 
         getTaskID(),reporter);
     
    // make a mapper
    org.apache.hadoop.mapreduce.Mapper<INKEY,INVALUE,OUTKEY,OUTVALUE> mapper =
      (org.apache.hadoop.mapreduce.Mapper<INKEY,INVALUE,OUTKEY,OUTVALUE>)
        ReflectionUtils.newInstance(taskContext.getMapperClass(), job);
    // make the input format
    org.apache.hadoop.mapreduce.InputFormat<INKEY,INVALUE> inputFormat =
      (org.apache.hadoop.mapreduce.InputFormat<INKEY,INVALUE>)
        ReflectionUtils.newInstance(taskContext.getInputFormatClass(), job);
    // rebuild the input split
    org.apache.hadoop.mapreduce.InputSplit split = null;
    split = getSplitDetails(new Path(splitIndex.getSplitLocation()),
        splitIndex.getStartOffset());
    LOG.info("Processing split: " + split);     
         
    // split: 就是需要处理数据的哪一个部分
    org.apache.hadoop.mapreduce.RecordReader<INKEY,INVALUE> input =
      new NewTrackingRecordReader<INKEY,INVALUE>
        (split, inputFormat, reporter, taskContext);     
         
      try {
      input.initialize(split, mapperContext);
      mapper.run(mapperContext);
      mapPhase.complete();
      
      // sort 
      setPhase(TaskStatus.Phase.SORT);
      statusUpdate(umbilical);
      input.close();
      input = null;
      
      // 输出
      output.close(mapperContext);
      output = null;
    }
    .....                       
```

input ->  map  -> output

input:(split+format) 通用的知识，未来的spark底层也是

来自于我们的输入格式化类给我们实际返回的记录读取器对象

```Java
TextInputFormat->LineRecordreader(split: file , offset , length)
init():
    in = fs.open(file).seek(offset)
	 除了第一个切片对应的map，之后的map都在init环节，
	 从切片包含的数据中，让出第一行，并把切片的起始更新为切片的第二行。
     换言之，前一个map会多读取一行，来弥补hdfs把数据切割的问题~！

	nextKeyValue():
	1，读取数据中的一条记录对key，value赋值
	2，返回布尔值
	getCurrentKey():
	getCurrentValue():
```





		output：
			NewOutputCollector
				partitioner
				collector
					MapOutputBuffer:
						*：
							map输出的KV会序列化成字节数组，算出P，最中是3元组：K,V,P
							buffer是使用的环形缓冲区：
								1，本质还是线性字节数组
								2，赤道，两端方向放KV,索引
								3，索引：是固定宽度：16B：4个int
									a)P
									b)KS
									c)VS
									d)VL
								5,如果数据填充到阈值：80%，启动线程：
									快速排序80%数据，同时map输出的线程向剩余的空间写
									快速排序的过程：是比较key排序，但是移动的是索引
								6，最终，溢写时只要按照排序的索引，卸下的文件中的数据就是有序的
									注意：排序是二次排序（索引里有P，排序先比较索引的P决定顺序，然后在比较相同P中的Key的顺序）
										分区有序  ： 最后reduce拉取是按照分区的
										分区内key有序： 因为reduce计算是按分组计算，分组的语义（相同的key排在了一起）
								7，调优：combiner
									1，其实就是一个map里的reduce
										按组统计
									2，发生在哪个时间点：
										a)内存溢写数据之前排序之后
											溢写的io变少~！
										b)最终map输出结束，过程中，buffer溢写出多个小文件（内部有序）
											minSpillsForCombine = 3
											map最终会把溢写出来的小文件合并成一个大文件：
												避免小文件的碎片化对未来reduce拉取数据造成的随机读写
											也会触发combine
									3，combine注意
										必须幂等
										例子：
											1，求和计算
											1，平均数计算
												80：数值和，个数和
						init():
							spillper = 0.8
							sortmb = 100M
							sorter = QuickSort
							comparator = job.getOutputKeyComparator();
										1，优先取用户覆盖的自定义排序比较器
										2，保底，取key这个类型自身的比较器
							combiner ？reduce
								minSpillsForCombine = 3

							SpillThread
								sortAndSpill()
									if (combinerRunner == null)
