---
title: Hadoop之PageRank

date: '2020-05-28'
categories:
    - 笔记
tags:
    - Hadoop

---

> PageRank，网页排名，是Google专有的算法，用于衡量特定网页相对于搜索引擎索引中的其他网页而言的重要程度。

# [](#原图 "原图")原图

![20160514130534469.png](https://i.loli.net/2020/05/28/FpN4RVG7UmXAyZJ.png)

# [](#设计数据存储格式 "设计数据存储格式")设计数据存储格式

## [](#输入 "输入")输入

```java
A 0.25 B C D
B 0.25 A D
C 0.25 C
D 0.25 B C
```

*   第一列代表当前网页
*   第二列代表当前网页PR值
*   之后的字母代表当前网页指向的链接网页
*   字符之间用一个空格隔开

## [](#输出 "输出")输出

因为PageRank算法需要迭代计算，所以输出格式应与输入格式保持一致。

# [](#MapReduce编程实现 "MapReduce编程实现")MapReduce编程实现

## [](#Map阶段 "Map阶段")Map阶段

map得到的内容如下：

```java
A 0.25 B C D
```

这里面有三类信息：

1.  当前网页名，A
2.  A当前的PR值，0.25
3.  A指向的链接网页，B、C、D

map的输出内容如下：

```java
A @B C D
B $0.0833
C $0.0833
D $0.0833
```

map的输出有两类：

1.  继续保存当前网页的所有指向链接
2.  各个被指向链接从当前网页得到的PR值

代码如下：

```java
package com.gx.pagerank;

import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;
import java.io.IOException;
import java.util.StringTokenizer;

public class PageRankMapper extends Mapper<LongWritable, Text,Text,Text> {
    Text keyInfo = new Text();
    Text valueInfo = new Text();
    private String id;//记录网页名
    private float pr;//网页PR值
    private int count;//当前网页拥有的链接数
    private float avg_pr;//当前网页分出去的平均PR值

    @Override//输入<A,0.25 B C D>
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        StringTokenizer itr = new StringTokenizer(value.toString());
        id = itr.nextToken();
        pr = Float.parseFloat(itr.nextToken());
        count = itr.countTokens();
        avg_pr = pr/count;
        String linkIds = "@";
        while (itr.hasMoreTokens()){
            String linkId = itr.nextToken();
            keyInfo.set(linkId);
            valueInfo.set("$"+avg_pr);
            linkIds += " " + linkId;
            context.write(keyInfo,valueInfo);//第一种输出类型 <B,$0.0833>、<C,$0.0833>、<D,$0.0833>
        }
        keyInfo.set(id);
        valueInfo.set(linkIds);
        context.write(keyInfo,valueInfo);//第二种输出类型 <A,@ B C D>
    }
}
```

## [](#Reduce阶段 "Reduce阶段")Reduce阶段

从Map到Reduce，框架会自动将Key相同的Value合并

所以Reduce得到的内容如下：

```java
A <@B C D,$0.125>
B <@A D,$0.125,$0.0833>
...
```

Reduce阶段就做两件事;

1.  将以$开头的PR值进行求和计算，作为此次迭代的网页PR值
2.  拼接各类字符，组成和输入格式一样的字符串输出

代码如下：

```java
package com.gx.pagerank;

import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;
import java.io.IOException;

public class PageRankReducer extends Reducer<Text, Text,Text, Text> {
    private Text keyInfo = new Text();
    private Text valueInfo = new Text();
    @Override
    protected void reduce(Text key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
        float pr = 0;
        String link = "";
        for(Text value : values){
            if('$'==value.toString().charAt(0)){//以$开头
                pr += Float.parseFloat(value.toString().substring(1));
            }else {//以@开头
                link += value.toString().substring(1);
            }
        }
        pr = 0.8f * pr + 0.2f * 0.25f;//加入跳转因子，进行平滑处理。最后的0.25f其实是网页总数分之一
        keyInfo.set(key);
        valueInfo.set(pr+link);
        context.write(keyInfo,valueInfo);
    }
}
```

## [](#Driver "Driver")Driver

驱动类都是套路，唯一要注意的就是，PageRank算法需要迭代，在驱动类里需要增加一个循环，一般迭代三四十次就收敛了。这里只迭代3次，简单表示一下。

代码如下：

```java
package com.gx.pagerank;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import java.io.IOException;

public class PageRankDriver {
    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
        Configuration conf = new Configuration();
        String pathIn = "F:\\hadooptemp\\input3";
        String pathOut = "F:\\hadooptemp\\output3";
        for (int i = 0; i < 3; i++) {
            Job job = Job.getInstance(conf);
            job.setJarByClass(PageRankDriver.class);
            job.setMapperClass(PageRankMapper.class);
            job.setReducerClass(PageRankReducer.class);
            job.setOutputKeyClass(Text.class);
            job.setOutputValueClass(Text.class);
            FileInputFormat.setInputPaths(job,new Path(pathIn));
            FileOutputFormat.setOutputPath(job,new Path(pathOut));
            pathIn = pathOut;
            pathOut = pathOut + i;
            boolean result = job.waitForCompletion(true);
        }
    }
}
```

迭代三次后的输出内容如下：

```java
A    0.12066667 B C D
B    0.15711111 A D
C    0.56511116 C
D    0.15711111 B C
```

# [](#总结 "总结")总结

## [](#没有细说的两个问题 "没有细说的两个问题")没有细说的两个问题

PageRank算法还有一些值得探讨的问题，这里没记录，比如Rank leak和Rank sink问题。简单说明一下：

1.  Rank leak：一个独立的网页如果没有外出的链接就产生等级泄漏

解决办法：

*   将无出度的节点递归的从图中去掉，待其他节点计算完毕后再添加。
*   对无出度的节点添加一条边，指向那些指向它的顶点。

2.  Rank sink：整个网页图中的一组紧密链接成环的网页如果没有外出的链接就产生Rank sink。

解决办法：

*   引入随机浏览模型。

## [](#一个网页PR值的计算公式 "一个网页PR值的计算公式")一个网页PR值的计算公式

![Q.bmp](https://i.loli.net/2020/05/28/DaGpOgI4CysfQZV.png)

其中Mpi是所有对pi网页有出链的网页集合，L(pj)是网页pj的出链数目，N是网页总数，α一般取0.85。（上述代码里α取得是0.8，N为4）也就是这一行代码：

```java
pr = 0.8f * pr + 0.2f * 0.25f;//加入跳转因子，进行平滑处理。最后的0.25f其实是网页总数分之一
```

## [](#要注意的细节问题 "要注意的细节问题")要注意的细节问题

*   还是要对每一个阶段的KV值的设定格外注意。
*   注意变量的作用域，设为全局变量还是局部变量，道理很简单，但马虎了，不容易查错。就因为这个问题，查了一个小时的错。本来应该是局部变量，结果大意了，设为了全局变量，搞混了几个变量的作用域，导致程序结果错误。
