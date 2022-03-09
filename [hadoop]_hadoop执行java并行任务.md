# hadoop执行java并行任务

摘要：记录hadoop的map-reduce的一个实例任务（WordCount）

此任务需要引入的jar包：

*commons-cli-1.2.jar*

hadoop-common-2.8.1.jar

hadoop-mapreduce-client-core-2.8.1.jar

jar包所在位置：share/hadoop/common/, share/hadoop/mapreduce/

#### 1、编写Map-Reduce代码

```java
import java.io.IOException;
import java.util.StringTokenizer;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.util.GenericOptionsParser;
public class WordCount {
	public static class TokenizeMapper extends Mapper<Object, Text, Text, IntWritable> {
		private final static IntWritable one = new IntWritable(1);
		private Text word = new Text();
		
		@Override
		protected void map(Object key, Text value, Mapper<Object, Text, Text, IntWritable>.Context context)
			throws IOException, InterruptedException {
			StringTokenizer itr = new StringTokenizer(value.toString());
			while (itr.hasMoreTokens()) {
				word.set(itr.nextToken());
				context.write(word, one);
			}
		}
	}
	
	public static class IntSumReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
		private IntWritable result = new IntWritable();
		
		@Override
		protected void reduce(Text key, Iterable<IntWritable> values,
			Reducer<Text, IntWritable, Text, IntWritable>.Context context) throws IOException, InterruptedException {
			int sum = 0;
			for (IntWritable val : values) {
				sum += val.get();
			}
			result.set(sum);
			context.write(key, result);
		}
	}
	
	public static void main(String[] args) throws Exception {
		Configuration conf = new Configuration();
		String[] otherArgs = new GenericOptionsParser(conf, args).getRemainingArgs();
		
		if (otherArgs.length < 2) {
			System.err.println("Usage: wordcount <in> [<in>...] <out>");
			System.exit(2);
		}
		
		Job job = Job.getInstance(conf, "word count");
		job.setJarByClass(WordCount.class);
		job.setMapperClass(TokenizeMapper.class);
		job.setCombinerClass(IntSumReducer.class);
		job.setReducerClass(IntSumReducer.class);
		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(IntWritable.class);
		
		for (int i = 0; i < args.length - 1; ++i) {
			FileInputFormat.addInputPath(job, new Path(otherArgs[i]));
		}
		FileOutputFormat.setOutputPath(job, new Path(otherArgs[otherArgs.length - 1]));
		System.exit(job.waitForCompletion(true) ? 0 : 1);
	}
}
```

#### 2、配置java环境

主要是在已有的环境基础上增加HADOOP_HOME变量，便于依赖hadoop编译打包。

```shell
export JAVA_HOME=/usr/local/java
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export HADOOP_CLASSPATH=$JAVA_HOME/lib/tools.jar
```

#### 3、编译打包

```shell
bin/hadoop com.sun.tools.javac.Main WordCount.java
jar cf wc.jar WordCount*.class
```

#### 4、准备hdfs文件

```shell
bin/hadoop fs -mkdir /dictxwang/w03
bin/hadoop fs -cp /dictxwang/w01/*.log /dictxwang/w03/
bin/hadoop fs -rm -r /dictxwang/w03-out
```

#### 5、执行并行任务

```shell
bin/hadoop jar wc.jar WordCount /dictxwang/w03/ /dictxwang/w03-out/
```

#### 6、查看执行结果

```shell
bin/hadoop fs -ls /dictxwang/w03-out/
Found 2 items
-rw-r--r--   1 ocm supergroup          0 2020-12-11 18:07 /dictxwang/w03-out/_SUCCESS
-rw-r--r--   1 ocm supergroup        140 2020-12-11 18:07 /dictxwang/w03-out/part-r-00000
```