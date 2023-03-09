---
title: Hadoop之InvertedIndex

date: '2020-05-25'
categories:
    - 笔记
tags:
    - Hadoop

---

> 倒排索引，简单地说就是：根据内容查文档，而不是根据文档查内容。

# [](#案例需求 "案例需求")案例需求

## [](#输入 "输入")输入

假设有三个txt文件，内容举例如下：

1.txt

```java
a b c
a b b b b
hello world
hello b
```

其他文件内容类似

## [](#输出 "输出")输出

要求输出文件的内容如下：

```java
a    1.txt:2;2.txt:2;
b    2.txt:1;3.txt:1;1.txt:6;
c    1.txt:1;2.txt:1;
d    2.txt:1;
hello    1.txt:2;3.txt:1;
j    2.txt:1;
s    2.txt:1;
world    1.txt:1;
```

## [](#分析 "分析")分析

1.  Map拿到手的内容举例如下：

```java
<0,"a b c">
```

Map输出的内容形式如下：

```java
<a:1.txt,1>
<a:1.txt,1>
<b:1.txt,1>
```

2.  经过Map之后，单纯依靠后面的Reduce阶段，不能同时完成词频统计和生成文档列表，所以必须增加一次Combine阶段

Combine拿到的内容形式上就是Map输出的内容，Combine的输出内容形式如下：

```java
<a,1.txt:2>
<b,1.txt:1>
```

其他文档的内容类似，里面也可能含有a、b等。

3.  Reduce阶段就只需将文件中相同key的value进行统计，组合成文档名加词频的形式就行

Reduce接收的内容形如Combine输出的内容，它输出内容的kv形式如下：

```java
<a,1.txt:2;2.txt:3>
```

# [](#编程实现 "编程实现")编程实现

## [](#Map "Map")Map

```java
package com.gx.InvertedIndex;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.lib.input.FileSplit;
import java.io.IOException;
import java.util.StringTokenizer;

public class InvertedIndexMapper extends Mapper<LongWritable, Text,Text,Text> {
    //存储单词和文档名称
    private Text keyInfo = new Text();
    //存储词频，初始化为1
    private Text valueInfo = new Text("1");

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        //得到这行数据所在的文件分片
        FileSplit fileSplit = (FileSplit)context.getInputSplit();
        //根据文件切片得到文件名
        String fileName = fileSplit.getPath().getName();
        StringTokenizer itr = new StringTokenizer(value.toString());
        while (itr.hasMoreTokens()){
            keyInfo.set(itr.nextToken()+":"+fileName);
            context.write(keyInfo,valueInfo);
        }
    }
}
```

## [](#Combine "Combine")Combine

```java
package com.gx.InvertedIndex;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;
import java.io.IOException;

public class InvertedIndexCombiner extends Reducer<Text, Text,Text,Text> {
    private Text keyInfo = new Text();
    private Text valueInfo = new Text();

    @Override
    protected void reduce(Text key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
        int sum = 0;//词频统计
        for(Text value : values){
            sum += Integer.parseInt(value.toString());
        }
        int splitIndex = key.toString().indexOf(":");
        //重新设置key为单词
        keyInfo.set(key.toString().substring(0,splitIndex));
        //重新设置value为文档名加词频
        valueInfo.set(key.toString().substring(splitIndex+1)+":"+sum);
        context.write(keyInfo,valueInfo);
    }
}
```

## [](#Reduce "Reduce")Reduce

```java
package com.gx.InvertedIndex;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;
import java.io.IOException;

public class InvertedIndexReducer extends Reducer<Text,Text,Text,Text> {
    private static Text valueInfo = new Text();

    @Override
    protected void reduce(Text key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
        //生成文档列表
        String fileList = new String();
        for(Text value : values){
            fileList += value.toString()+";";//注意这是分号，记录了单词所属的那些文档
        }
        valueInfo.set(fileList);
        context.write(key,valueInfo);
    }
}
```

## [](#Driver "Driver")Driver

```java
package com.gx.InvertedIndex;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import java.io.IOException;

public class InvertedIndexDriver {
    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
        Job job = Job.getInstance(new Configuration());
        job.setJarByClass(InvertedIndexDriver.class);
        job.setMapperClass(InvertedIndexMapper.class);
        job.setCombinerClass(InvertedIndexCombiner.class);
        job.setReducerClass(InvertedIndexReducer.class);
        //Map和Reduce输出类型一致时可以简写成下面两句，否则要分别单独指明类型
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(Text.class);
        FileInputFormat.setInputPaths(job,new Path(args[0]));
        FileOutputFormat.setOutputPath(job,new Path(args[1]));
        boolean result = job.waitForCompletion(true);
        System.exit(result ? 0 : 1);
    }
}
```

# [](#易错点 "易错点")易错点

1.  各个阶段的KV类型一定要小心，没什么特别要求的话，可以统一为Text类型。这样后面的Driver类也可以简写。
2.  在设置各个阶段的输出内容，也就是KV值时，为了防止出错，应该预先创建要输出的key对象和value对象，而不应该直接对本阶段输入的key对象进行处理，否则容易造成逻辑错误。（这是一次教训，就因为设置kv，造成了逻辑错误，程序运行结束，只有输出文件包，但没有结果文件，查了一个多小时的错。）
3.  分割字符串时，可以考虑用StringTokenizer类。
