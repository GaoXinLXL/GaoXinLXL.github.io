---
title: Hadoop之TopN

date: '2020-05-31'
categories:
    - 笔记
tags:
    - Hadoop

---

> TopN分析法是指从研究对象中按照某一个指标进行倒序或正序排列，取其中所需的N个数据，并对这N个数据进行重点分析的方法。

# [](#算法 "算法")算法

输入数据格式如下：

```java
10 3 8 7
12 23 45 3 2
19 28 34
```

*   Map阶段：可以用TreeMap来保存TopN的数据。另外，以往的Mapper都是处理一行数据之后就用context.write()方法输出，而现在需要包所有数据保存在TreeMap后再写入，所以把context.write()方法放在cleanup()里执行。cleanup()方法就是整个MapTask执行完毕后才执行的一个方法。
*   Reduce阶段：汇总数据，取TopN数据即可。

# [](#编程实现 "编程实现")编程实现

## [](#Map "Map")Map

```java
package com.gx.topn;

import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;
import java.io.IOException;
import java.util.TreeMap;

public class TopnMapper extends Mapper<LongWritable, Text, NullWritable, IntWritable> {
    private TreeMap<Integer,String> map = new TreeMap<Integer, String>();
    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        String line = value.toString();
        String[] nums = line.split(" ");
        for (String num : nums){
            map.put(Integer.parseInt(num)," ");
            if(map.size()>5){
                map.remove(map.firstKey());
            }
        }
    }

    @Override
    protected void cleanup(Context context) throws IOException, InterruptedException {
        for (Integer i : map.keySet()){
            try {
                context.write(NullWritable.get(),new IntWritable(i));
            }catch (Exception e){
                e.printStackTrace();
            }
        }
    }
}
```

## [](#Reduce "Reduce")Reduce

```java
package com.gx.topn;

import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.mapreduce.Reducer;
import java.io.IOException;
import java.util.Comparator;
import java.util.TreeMap;

public class TopnReducer extends Reducer<NullWritable, IntWritable,NullWritable,IntWritable> {
    private TreeMap<Integer,String> map = new TreeMap<Integer, String>(new Comparator<Integer>() {
        public int compare(Integer a, Integer b) {
            return b - a;
        }
    });
    @Override
    protected void reduce(NullWritable key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
        for (IntWritable value : values){
            map.put(value.get()," ");
            if (map.size()>5){
                map.remove(map.firstKey());
            }
        }
        for (Integer i : map.keySet()){
            context.write(NullWritable.get(),new IntWritable(i));
        }
    }
}
```

## [](#Driver "Driver")Driver

```java
package com.gx.topn;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import java.io.IOException;

public class TopnDriver {
    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf);
        job.setJarByClass(TopnDriver.class);
        job.setMapperClass(TopnMapper.class);
        job.setReducerClass(TopnReducer.class);
        job.setMapOutputKeyClass(NullWritable.class);
        job.setMapOutputValueClass(IntWritable.class);
        job.setOutputKeyClass(NullWritable.class);
        job.setOutputValueClass(IntWritable.class);
        FileInputFormat.setInputPaths(job,new Path(args[0]));
        FileOutputFormat.setOutputPath(job,new Path(args[1]));
        boolean result = job.waitForCompletion(true);
        System.exit(result ? 0 : 1);
    }
}
```

# [](#总结 "总结")总结

TopN算法本身不难，但可以将其作为一个模板，考虑对其进行改进。
