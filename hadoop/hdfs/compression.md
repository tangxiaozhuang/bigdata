压缩
===
目前hadoop的压缩有gzip, lzo, snappy等，许多文章都有对这几种压缩的对比，此不赘述。从效率和可分块的角度来看，这里选择lzo做为压缩方式。文件压缩后，Hadoop会根据文件扩展名来选取相应的codec，codec从配置文件的`io.compression.codec`中查找，MapReduce在读取文件时根据codec会自动解压文件。

安装
---

### 集群节点安装lzo
在Yarn的各节点上安装lzo：`yum -y install lzo-devel`。当然也可在[官网](http://www.oberhumer.com/opensource/lzo/download/)下载后编译安装:

```
# ./configure --enable-shared --prefix /usr/local/lzo-2.09
# make && make install
```

若需要在本地生成lzo文件，还需要安装lzop：`yum -y install lzop`。

### 非CDH生成Lzop jar包
**注意：若使用CDH，可使用CDH提供的jar包，本步骤可忽略。**
生成的jar用于给hdfs中的lzo文件添加索引。

- 下载`git clone https://github.com/twitter/hadoop-lzo/`。
- 修改pom.xml文件：
因为使用的Hadoop版本为2.3.0，因此修改如下地方
```
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <hadoop.current.version>2.4.0</hadoop.current.version>
    <hadoop.old.version>1.0.4</hadoop.old.version>
</properties>
```
为
```
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <hadoop.current.version>2.3.0</hadoop.current.version>
    <hadoop.old.version>1.0.4</hadoop.old.version>
  </properties>
```

这里的版本号根据相应hadoop不同而不同。

- 生成jar包，如下：
```
# export C_INCLUDE_PATH=/usr/local/lzo-2.09/include
# export LIBRARY_PATH=/usr/local/lzo-2.09/lib
# mvn clean package -Dmaven.test.skip=true`
```
- 将生成的jar拷贝到hadoop的lib目录。

### CDH安装hadoop-lzo

CDH中[GPL Extras](http://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_install_gpl_extras.html)包含了LZO函数，因此安装GPL Extras包即可。CDH5.4中gplextras的yum源地址为https://archive.cloudera.com/gplextras5/parcels/{latest_supported}。

若使用CDH的话，按照如下步骤操作：

1. 在 Administation --> Settings --> Parcels 页面中，对于`Remote Parcel Repository URLs`，添加url：https://archive.cloudera.com/gplextras5/parcels/{latest_supported} 。保存。
2. 在 Hosts --> Parcels --> Downloadable 页面中，下载GPLEXTRAS，然后分配并激活。

配置
---
### CDH

- 修改hdfs中`io.compression.codecs`的属性值，添加`com.hadoop.compression.lzo.LzopCodec`
- 修改Yarn中`mapreduce.application.classpath`的属性值，添加`/opt/cloudera/parcels/GPLEXTRAS/lib/hadoop/lib/*`
- 修改Yarn中`mapreduce.admin.user.env`的属性值，添加`/opt/cloudera/parcels/GPLEXTRAS/lib/hadoop/lib/native`
- 若希望所有数据都被压缩，配置Yarn中的`mapreduce.output.fileoutputformat.compress`属性值为true，`mapreduce.output.fileoutputformat.compress.codec`属性值为`org.apache.hadoop.io.compress.LzoCodec`，`mapreduce.output.fileoutputformat.compress.type`属性值为`BLOCK`
- 若希望map的输出结果也被压缩，还需要修改Yarn中`mapreduce.map.output.compress`属性值为ture，并将`mapreduce.map.output.compress.codec`属性值改为`org.apache.hadoop.io.compress.LzoCodec`。若没有修改的话，也可在代码中设置。

修改完毕后保存并deploy配置。

### 非CDH
非CDH可参考CDH的各参数修改，如下

- core-site.xml中修改`io.compression.codecs`属性值，添加`com.hadoop.compression.lzo.LzopCodec`
- 修改hadoop-env.sh，添加lzo库的路径
- 对于map输出结果和最终结果的压缩，参考CDH的修改。

*LzoCodec与LzopCodec的比较*
>
> LzoCodec 与 LzopCodec的区别如同Lzo与Lzop的区别，前者是一种快速的压缩库，后者在前者的基础上添加了额外的文件头。
> 
> 若使用LzoCodec作为Reduce输出，则输出文件的扩展名为`.lzo_deflate`，其无法作为MapReduce的的输入，`DistributedLzoIndexer`也无法为其创建索引；若使用LzopCodec作为Reduce输出，则输出文件的扩展名为 `.lzo`。可参考[What's the difference between the LzoCodec and the LzopCodec in Hadoop-LZO?](https://www.quora.com/Whats-the-difference-between-the-LzoCodec-and-the-LzopCodec-in-Hadoop-LZO)。
>

生成lzo索引文件
---
若hdfs中已有lzo文件，还需要生成index文件。执行如下命令可在本地压缩：

```
hadoop jar /path/to/your/hadoop-lzo.jar com.hadoop.compression.lzo.LzoIndexer big_file.lzo
```

若希望通过mapreduce来进行压缩，命令如下：
```
hadoop jar /path/to/your/hadoop-lzo.jar com.hadoop.compression.lzo.DistributedLzoIndexer big_file.lzo
```

MapReduce压缩文件
---
若需要Reduce结果为lzo，并添加索引。需要自己编写代码并生成jar包用来压缩文件。如下：

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.compress.CompressionCodec;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.mapreduce.Job;
import com.hadoop.compression.lzo.LzopCodec;
import com.hadoop.compression.lzo.LzoIndexer;

public class Compress {

    public static void main(String[] args) throws Exception {

        if (args.length != 2) {
            System.out.println("Usage: hadoop jar compress-1.0-SNAPSHOT.jar Compress INPUT OUTPUT");
            System.exit(-1);
        }

        Configuration conf = new Configuration();
        conf.setBoolean("mapreduce.map.output.compress", true);
        conf.setClass("mapreduce.map.output.compress.codec", LzopCodec.class, CompressionCodec.class);
        Job job = Job.getInstance(conf, "compress_hdfs");
        job.setJarByClass(Compress.class);

        Path fileInput = new Path(args[0]);
        Path fileOutput = new Path(args[1]);
        FileInputFormat.addInputPath(job, fileInput);
        FileOutputFormat.setOutputPath(job, fileOutput);
        FileOutputFormat.setCompressOutput(job, true);
        FileOutputFormat.setOutputCompressorClass(job, LzopCodec.class);

        if(job.waitForCompletion(true)) {
            LzoIndexer indexer = new LzoIndexer(conf);
            indexer.index(fileOutput);
        } else {
            System.exit(1);
        }
    }
}
```

执行`mvn clean install`打包后，运行包，可查看执行结果。压缩完数据后，可通过比较压缩前后数据行数来大致判断压缩过程中是否有数据丢失：
```
# 压缩前
> hdfs dfs -cat /origin_file |wc -l

# 压缩后
> sum=0;for i in {0..15};do if [ $i -le 9 ];then t=0$i; else t=$i;fi; ((sum+=`hdfs dfs -text /compress_file/part-r-000$t.lzo|wc -l`));done; echo $sum
```

解压
---
使用压缩后，即使不指定inputformat，Mapreduce也能根据文件后缀来读取文件，但对于不能分区的压缩方式，整个文件将只能由一个map来处理。相关内容可参考[Hadoop2 Mapreduce输入输出压缩](http://www.winseliu.com/blog/2014/09/01/hadoop2-mapreduce-compress/)。若使用lzo生成索引后，文件能支持分区，但inputformat必须指定为`LzoTextInputFormat`：`job.setInputFormatClass(LzoTextInputFormat.class);`，以防将索引文件也当成输入文件。

hive
---
若希望hive使用lzo压缩，需在建表时指定`STORED AS INPUTFORMAT 'com.hadoop.mapred.DeprecatedLzoTextInputFormat' OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'`

执行
```
SET mapreduce.output.fileoutputformat.compress.codec=com.hadoop.compression.lzo.LzopCodec;
SET hive.exec.compress.output=true;
SET mapreduce.output.fileoutputformat.compress=true;
```
再写入数据，可看到lzo文件。

在测试hive过程，直接load lzo文件和将lzo文件拷贝到hdfs，hive都能正常读取压缩文件。hive表结构没有修改。

spark
---
spark使用lzo需要`spark-conf/spark-env.sh`配置如下：
```
SPARK_CLASSPATH=$SPARK_CLASSPATH:/etc/hive/conf:/opt/cloudera/parcels/GPLEXTRAS/lib/hadoop/lib/hadoop-lzo.jar
SPARK_LIBRARY_PATH=$SPARK_LIBRARY_PATH:/opt/cloudera/parcels/CDH/lib/hadoop/lib/native:/opt/cloudera/parcels/GPLEXTRAS/lib/hadoop/lib/native
LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/cloudera/parcels/CDH/lib/hadoop/lib/native:/opt/cloudera/parcels/GPLEXTRAS/lib/hadoop/lib/native
```

**UPDATE: 业务方反馈添加SPARK_CLASSPATH后，所有文件都会被压缩为lzo，因此取消该配置。**

测试如下：
```
scala> val lf = sc.textFile("/user/hive/warehouse/rp_sdk_tmp.db/use_id/test.txt.lzo")
lf: org.apache.spark.rdd.RDD[String] = MapPartitionsRDD[1] at textFile at <console>:21

scala> lf.count
res0: Long = 3
```

问题
---
- Failed to load/initialize native-lzo library

当运行压缩时，报错如下：

```
16/01/18 18:36:45 INFO lzo.GPLNativeCodeLoader: Loaded native gpl library from the embedded binaries
16/01/18 18:36:45 WARN lzo.LzoCompressor: java.lang.UnsatisfiedLinkError: Cannot load liblzo2.so.2 (liblzo2.so.2: cannot open shared object file: No such file or directory)!
16/01/18 18:36:45 ERROR lzo.LzoCodec: Failed to load/initialize native-lzo library
```
在hadoop-env.sh文件中，配置安装lzo的路径即可：

```
export LD_LIBRARY_PATH=/usr/local/lzo-2.09/lib
```

- [todo]压缩后mr数据和压缩前不太一致

在压缩后，除了比较文件行数外，还写了个wordcount来对比文件，但出乎意料的是压缩后的数据wordcount出来的结果比压缩前多了些无用数据。


Reference
===
- [Choosing a Data Compression Format](http://www.cloudera.com/content/www/en-us/documentation/enterprise/5-2-x/topics/admin_data_compression_performance.html)
- [Data Compression in Hadoop](http://comphadoop.weebly.com/)
- [Snappy and Hadoop](http://blog.cloudera.com/blog/2011/09/snappy-and-hadoop/)
- [Is Snappy splittable or not splittable](http://stackoverflow.com/questions/32382352/is-snappy-splittable-or-not-splittable)
- [LZO vs Snappy vs LZF vs ZLIB, A comparison of compression algorithms for fat cells in HBase](http://blog.erdemagaoglu.com/post/4605524309/lzo-vs-snappy-vs-lzf-vs-zlib-a-comparison-of)
- [How to Use Intermediate and Final Output Compression (MR1 & YARN)](https://datameer.zendesk.com/hc/en-us/articles/204258750-How-to-Use-Intermediate-and-Final-Output-Compression-MR1-YARN-)
- [Hadoop at Twitter (part 1): Splittable LZO Compression](http://blog.cloudera.com/blog/2009/11/hadoop-at-twitter-part-1-splittable-lzo-compression/)
