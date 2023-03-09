---
title: Hadoop之DataDeduplication

date: '2020-05-31'
categories:
    - 笔记
tags:
    - Hadoop

---

> 数据去重就是去除重复数据的操作，较为简单。

# [](#输入文件格式 "输入文件格式")输入文件格式

算法的思路是个大方向，但具体的编码要具体分析，比如要考虑数据的格式。这里数据的格式如下：

```java
2019-2-1 a
2020-1-1 n
...
```

# [](#算法 "算法")算法

*   Map阶段：将key设为需要去重的数据，value设为空
*   Reduce阶段：将得到的key继续复制为输出的key，value还是设为空

# [](#代码实现 "代码实现")代码实现

## [](#Map "Map")Map

```java
package com.gx.quchong;

import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;
import java.io.IOException;

public class MyMapper extends Mapper<LongWritable, Text,Text,NullWritable> {
    private static Text keyInfo = new Text();
    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        keyInfo = value;
        context.write(keyInfo, NullWritable.get());
    }
}
```

## [](#Reduce "Reduce")Reduce

```java
package com.gx.quchong;

import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;
import java.io.IOException;

public class MyReducer extends Reducer<Text, NullWritable,Text,NullWritable> {
    @Override
    protected void reduce(Text key, Iterable<NullWritable> values, Context context) throws IOException, InterruptedException {
        context.write(key,NullWritable.get());
    }
}
```

## [](#Driver "Driver")Driver

```java
package com.gx.quchong;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import javax.xml.soap.Text;
import java.io.IOException;

public class Driver {
    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf);
        job.setJarByClass(Driver.class);
        job.setMapperClass(MyMapper.class);
        job.setReducerClass(MyReducer.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(NullWritable.class);
        FileInputFormat.setInputPaths(job,new Path(args[0]));
        FileOutputFormat.setOutputPath(job,new Path(args[1]));
        boolean result = job.waitForCompletion(true);
        System.exit(result ? 0 : 1);
    }
}
```

# [](#总结 "总结")总结

数据去重算法本身较为简单，进一步体会MapReduce编程。