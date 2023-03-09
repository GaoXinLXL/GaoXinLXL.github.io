---
title: Hadoop之WordCount

date: '2020-05-25'
categories:
    - 笔记
tags:
    - Hadoop

---

> 词频统计是MapRedurce编程的一个经典案例

# [](#继承Mapper类 "继承Mapper类")继承Mapper类

```java
public class WordCountMapper extends Mapper<LongWritable, Text,Text, IntWritable>{
    Text k = new Text();
    IntWritable v = new IntWritable(1);

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        String line = value.toString();
        String[] words = line.split(" ");
        for(String word : words){
            k.set(word);
            context.write(k,v);
        }
    }
}
```

# [](#继承Reducer类 "继承Reducer类")继承Reducer类

```java
public class WordCountReducer extends Reducer<Text, IntWritable,Text,IntWritable> {
    IntWritable v = new IntWritable();

    @Override
    protected void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
        //局部汇总
        int count = 0;
        for(IntWritable value : values){
            count += value.get();
        }
        v.set(count);
        context.write(key,v);
    }
}
```

# [](#编写驱动类 "编写驱动类")编写驱动类

```java
public class WordCountDriver {
    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
        //通过Job来封装本次MR信息
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf);

        //指定驱动类、Mapper类、Reducer类
        job.setJarByClass(WordCountDriver.class);
        job.setMapperClass(WordCountMapper.class);
        job.setReducerClass(WordCountReducer.class);

        //指定Mapper类的输出KV类型
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(IntWritable.class);
        //指定Reducer类的输出KV类型
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);

        //设置输入、输出路径
        FileInputFormat.setInputPaths(job,new Path(args[0]));
        FileOutputFormat.setOutputPath(job,new Path(args[1]));

        //提交Job
        boolean result = job.waitForCompletion(true);
        System.exit(result ? 0 : 1);
    }
}
```

# [](#注意 "注意")注意

1.  这里驱动类里的输入输出路径没有写死，可以带参运行。
2.  Map类里的一点细节，尽量不要在map方法里反复创建对象。
