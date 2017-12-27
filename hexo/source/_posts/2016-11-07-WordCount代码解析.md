---
layout: post
title: 'WordCount代码解析'
date: 2016-11-07
tags: [MapReduce]
---

##### 在了解MapReduce的工作原理之后尝试运行HelloWord级别的WordCount程序，牢记7个过程（嗯= =渣渣都是从HelloWord开始学起的，从头开始mark下~）

主要参考了以下几篇blog：

- [初学Hadoop之图解MapReduce与WordCount示例分析](http://www.cnblogs.com/hehaiyang/p/4484442.html#label_2)  
- [Hadoop开发周期（二）：编写mapper和reducer程序](http://blog.csdn.net/yangzhongblog/article/details/8712028)  
- [第一个MapReduce程序——WordCount](http://songlee24.github.io/2015/07/29/mapreduce-word-count/)
- 【附】[在 Sublime 中配置 Markdown 环境](http://frank19900731.github.io/blog/2015/04/13/zai-sublime-zhong-pei-zhi-markdown-huan-jing/)

#### 〇、知识准备

##### 0.0 Hadoop数据类型  

由于MapReduce程序主要操作key-value键值对，Hadoop提供了预定义的数据类型，全部实现了WritableComparable接口，便于被序列化后网络传输，文件存储和比较等。常用的以下几种：

- Text: 用utf-8格式存储的文本（类似String类型）；
- IntWritable: 整形；
- LongWritable: 长整形；
- 还有浮点型布尔型blabla~~  

##### 0.1 StringTokenizer类的使用  

StringTokenizer类是java.util包中用于字符串分隔解析的工具类（可以指定分隔符类型）  
p.s.据说是出于兼容性原因的保留类，不建议使用，现多使用split方法

1. `构造函数`  

1) 默认构造函数： 
StringTokenizer(String str)     // 默认分隔符为空格、制表符\t、换行符\n、回车符\r  

2) 指定分隔符  
StringTokenizer(String str, String delim)

3) 指定分隔符 && 是否返回分隔符（即分隔符也算在nextToken之内）
   StringTokenizer(String str, String delim, boolean returnDelims)  

2. `常用方法`  

1) int countTokens(): 返回分隔符数量（使用前两种构造函数方式）

2) boolean hasMoreTokens()/ boolean hasMoreElements(): 是否还有分隔符  

3) String nextToken(): 返回到下一个分隔符之前的字符串  


```java
// StringTokenizer示例使用
import java.util.StringTokenizer;

String s = new String("Haha a nice Day, isn't it?");
StringTokenizer st = new StringTokenizer(s);
System.out.println("Total tokens: " + st.countTokens());
while(st.hasMoreTokens()) {
  System.out.println(st.nextToken());
}

```

##### 0.2 Mapper类和Reducer类  

MR的重点是编写map和reduce程序，每个job可以分为map和reduce阶段：  
> map: (K1, V1) => list(K2, V2)
> reduce: (K2, list(V2)) => list(K3, V3)  // 注意reduce过程的输入是map输出经过shuffle合并(排序)过的，每个相同的key对应一个value的list

在后面的代码中可以看到Mapper和Reducer类是作为被继承的对象，实际上是一个泛型类（抽象类），4个形参`Mapper<Object, Text, Text, IntWritable>`，分别表示：map函数输入key的类型（这里后面没有用到，实际为该行首字母相对于文本文件首地址的偏移量）、输入value的类型（这里为Text字符串类型，读入一行文本）、输出key的类型（这里为分割后的单词）、输出value类型（这里为单词出现次数），具体的见后面代码注释~ ~ 

#### 一、Map过程  

在看具体代码之前还是先啰嗦几句创建工程的事儿~这里的src下package分为三个源文件：WordCount.java, TokenizerMapper.java, CountReducer.java，功能显而易见哈~然后是导入hadoop lib的各种包，由于不清楚具体要导入哪几个就尽量把关联的都导进去了= =在工程下的properties -> Java Build Path -> Libiraries -> Add External JARS


```java
package wordcount;

import java.util.StringTokenizer;
import java.io.IOException;

import org.apache.hadoop.io.Text;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.mapreduce.Mapper;

//TokenizerMapper类继承了Mapper抽象类，实现map函数
public class TokenizerMapper extends Mapper<Object,Text,Text,IntWritable>{

	// 成员one声明为常量，作为map输出的value；word变量为单词，作为map输出的key
	private final static IntWritable one = new IntWritable(1);
	private Text word = new Text();
	/*
	 * map函数的参数：
	 * @key: Object, 表示map输入的key，这里是Object类型，表示该行相对于文本文件的偏移地址（后面没有用到这个参数）；
	 * @value：Text，表示map输入的value，这里为文本文件的一行内容；
	 * @context: Context类型，用于容纳map的输出<k-v>对
	 *  context.write(<k,v>)方法可以将输出保存到context中
	 * */
	public void map(Object key, Text value, Context context) throws IOException, InterruptedException {
		// 分隔字符串
		StringTokenizer itr = new StringTokenizer(value.toString());
		while(itr.hasMoreTokens()) {
			// 将切分出的单词存入Text类型的成员变量word中，用word.set(String)方法
			word.set(itr.nextToken());
			context.write(word, one);
		}
	}

}
```


#### 二、Reduce过程  

```java
package wordcount;

import java.io.IOException;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.mapreduce.Reducer;

// CountReducer继承自Reducer类，参数依次为输入key类型，输入value类型，输出key类型，输出value类型
public class CountReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
  // 输出的value，int类型代表count的次数
	IntWritable result = new IntWritable();
	
  // 重写reduce方法，参数依次为reduce输入key，输入value，输出容器Context类型
  // 【注意】reduce方法的输入value类型并不是map的输出value(Intwritable)，因为map的输出经过shuffle的合并排序后，可使相同的key进入相同的reduce，并进行了合并操作，将相同的key对应的value存到Iterable<IntWritable>中，可通过迭代器取出相加  
	public void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException{
		int sum = 0;
    for (IntWritable val:values) {
      sum += val.get();
    }
    result.set(sum);
    context.write(key, result);
	}
}

```


#### 三、主程序  

```java
package wordcount;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.util.GenericOptionsParser;

// WordCount的主程序
public class WordCount {

	public static void main(String[] args) throws Exception{
		// 执行Job作业
    // 运行MapReduce程序之前初始化Configuration，读取系统配置信息（包括HDFS, MapReduce的core-site.xml, hdfs-site.xml, mapred-site.xml等，具体可以在conf包里）
    Configuration conf = new Configuration();
    // 用conf和输入参数args(输入输出文件的路径)初始化GenericOptionsParser类，用来解释常用的Hadoop命令
    String[] otherArgs = new GenericOptionsParser(conf, args).getRemainingArgs();
    if (otherArgs.length != 2) {
      System.err.println("Usage: wordcount <in> <out>");
      System.exit(2);   // 运行程序需要两个参数，指明输入输出文件路径，否则异常退出
    }

    // 用conf配置信息创建一个名为wordcount的MapReduce-Job
    Job job = new Job(conf, "wordcount");
    // job中装载MapReduce程序，类名为WordCount
    job.setJarByClass(WordCount.class);
    // 装载map, reduce函数实现类
    job.setMapperClass(TokenizerMapper.class);
    job.setCombinerClass(CountReducer.class);
    job.setReducerClass(CountReducer.class);
    // 定义输出的key-value类型，即存储在HDFS上的结果文件k-v类型
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);
    // 构建输入输出数据文件
    FileInputFormat.addInputPath(job, new Path(otherArgs[0]));
    FileOutputFormat.setOutputPath(job, new Path(otherArgs[1]));
    System.exit(job.waitForCompletion(true) ? 0 : 1);

	}
}
```

#### 四、环境配置 && 打包运行程序  

`环境说明`：实验室集群上的java -version为1.7版本的，因此在本地打包程序的时候要选择JRE为1.7版本的才可以，否则hadoop运行jar包的时候会提示"Exception in thread "main" java.lang.UnsupportedClassVersionError: WordCount : Unsupported major.minor version 52.0"（之前eclipse装个最新版就是jdk1.8版本，连1.7都木有= =）


##### 4.1 创建工程 && 打包编译  

1) Eclipse中新建个java工程，src下准备好三个程序；  

2) 导入jar包（可以在hadoop官网上下载需要的版本，这里用的是hadoop1.x版本，现在多用2.x版本，流程是一样的~）

p.s.作为小白再啰嗦一下添加jar包的过程，工程名右键 => properties => java build path => Libraries => Add External JARS.添加之后大概张这个样子滴~  
![工程目录结构](https://github.com/shanxiaohan/markdownPic/blob/master/img/wordcount.jpg?raw=true)

3) 打包之前一定要在Properties => Java Compiler选择1.7版本的！

4) 打包编译，Export => Java => JAR File到指定输出路径

##### 4.2 放到集群上运行jar包  

这里需要注意两点：  
- hadoop执行jar命令运行jar包的时候要指定主程序的类名，如果代码中指定了package名也要带上；  
- 执行命令时需要指定输入输出路径，接在类名后面，输入路径可以为本地文件(夹)也可以为hdfs系统上的路径，输出路径`必须为hdfs文件系统`的路径，并且是一个新的文件夹名，**不能是已经存在的路径**，否则会报错File Exits= =  

执行代码：  

```java
 hadoop jar wordcount.jar wordcount.WordCount /user/hadooptest/shanxiaohan/wordcount/input/access.log /user/hadooptest/shanxiaohan/wordcount/outtest
```

可以看到控制台输出的日志信息：  

```java
16/11/09 10:19:21 INFO client.RMProxy: Connecting to ResourceManager at Master/192.168.83.30:8032
16/11/09 10:19:22 INFO input.FileInputFormat: Total input paths to process : 1
16/11/09 10:19:22 INFO lzo.GPLNativeCodeLoader: Loaded native gpl library
16/11/09 10:19:22 INFO lzo.LzoCodec: Successfully loaded & initialized native-lzo library [hadoop-lzo rev b928d466da6e6d34b836620d1639c9ee14b29431]
16/11/09 10:19:23 INFO mapreduce.JobSubmitter: number of splits:1
16/11/09 10:19:23 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1477278312906_0292
16/11/09 10:19:23 INFO impl.YarnClientImpl: Submitted application application_1477278312906_0292
16/11/09 10:19:24 INFO mapreduce.Job: The url to track the job: http://Master:8088/proxy/application_1477278312906_0292/
16/11/09 10:19:24 INFO mapreduce.Job: Running job: job_1477278312906_0292
16/11/09 10:19:36 INFO mapreduce.Job: Job job_1477278312906_0292 running in uber mode : false
16/11/09 10:19:36 INFO mapreduce.Job:  map 0% reduce 0%
16/11/09 10:19:44 INFO mapreduce.Job:  map 100% reduce 0%
16/11/09 10:19:53 INFO mapreduce.Job:  map 100% reduce 13%
16/11/09 10:19:54 INFO mapreduce.Job:  map 100% reduce 25%
16/11/09 10:19:58 INFO mapreduce.Job:  map 100% reduce 38%
16/11/09 10:20:00 INFO mapreduce.Job:  map 100% reduce 50%
16/11/09 10:20:03 INFO mapreduce.Job:  map 100% reduce 63%
16/11/09 10:20:05 INFO mapreduce.Job:  map 100% reduce 75%
16/11/09 10:20:07 INFO mapreduce.Job:  map 100% reduce 88%
16/11/09 10:20:11 INFO mapreduce.Job:  map 100% reduce 100%
16/11/09 10:20:12 INFO mapreduce.Job: Job job_1477278312906_0292 completed successfully
16/11/09 10:20:13 INFO mapreduce.Job: Counters: 49
	File System Counters
		FILE: Number of bytes read=552492
		FILE: Number of bytes written=1932302
		FILE: Number of read operations=0
		FILE: Number of large read operations=0
		FILE: Number of write operations=0
		HDFS: Number of bytes read=9672609
		HDFS: Number of bytes written=920034
		HDFS: Number of read operations=27
		HDFS: Number of large read operations=0
		HDFS: Number of write operations=16
	Job Counters 
		Launched map tasks=1
		Launched reduce tasks=8
		Data-local map tasks=1
		Total time spent by all maps in occupied slots (ms)=5457
		Total time spent by all reduces in occupied slots (ms)=32991
		Total time spent by all map tasks (ms)=5457
		Total time spent by all reduce tasks (ms)=32991
		Total vcore-seconds taken by all map tasks=5457
		Total vcore-seconds taken by all reduce tasks=32991
		Total megabyte-seconds taken by all map tasks=5587968
		Total megabyte-seconds taken by all reduce tasks=33782784
	Map-Reduce Framework
		Map input records=127001
		Map output records=1016007
		Map output bytes=13609497
		Map output materialized bytes=552460
		Input split bytes=138
		Combine input records=1016007
		Combine output records=87139
		Reduce input groups=87139
		Reduce shuffle bytes=552460
		Reduce input records=87139
		Reduce output records=87139
		Spilled Records=174278
		Shuffled Maps =8
		Failed Shuffles=0
		Merged Map outputs=8
		GC time elapsed (ms)=318
		CPU time spent (ms)=18450
		Physical memory (bytes) snapshot=2015809536
		Virtual memory (bytes) snapshot=13858652160
		Total committed heap usage (bytes)=2117074944
	Shuffle Errors
		BAD_ID=0
		CONNECTION=0
		IO_ERROR=0
		WRONG_LENGTH=0
		WRONG_MAP=0
		WRONG_REDUCE=0
	File Input Format Counters 
		Bytes Read=9672471
	File Output Format Counters 
		Bytes Written=920034

```

最后在hdfs上查看  

```java
  hadoop fs -ls shanxiaohan/wordcount/outtest

```



##### p.s. 终于看Sublime Text3自带的Markdown主题不爽了，搜下Monokai Extended & Markdown Extended插件看着美多了哈哈哈~ ~附上大神的链接~[在 Sublime 中配置 Markdown 环境](http://frank19900731.github.io/blog/2015/04/13/zai-sublime-zhong-pei-zhi-markdown-huan-jing/)

