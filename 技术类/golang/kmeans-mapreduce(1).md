个人博客原文：[Kmeans算法解析及基于MapReduce的并行化实现](http://blog.taohuawu.club/article/20)
>Kmeans算法，最为经典的基于划分的聚类方法

# Kmeans算法：
k-means 算法接受参数 k ；然后将事先输入的n个数据对象划分为 k个聚类以便使得所获得的聚类满足：同一聚类中的对象相似度较高；而不同聚类中的对象相似度较小。聚类相似度是利用各聚类中对象的均值所获得一个“中心对象”（引力中心）来进行计算的。

K-means算法是最为经典的基于划分的聚类方法，是十大经典数据挖掘算法之一。K-means算法的基本思想是：以空间中k个点为中心进行聚类，对最靠近他们的对象归类。通过迭代的方法，逐次更新各聚类中心的值，直至得到最好的聚类结果。

假设要把样本集分为c个类别，算法描述如下：

（1）适当选择c个类的初始中心；

（2）在第k次迭代中，对任意一个样本，求其到c个中心的距离，将该样本归到距离最短的中心所在的类；

（3）利用均值等方法更新该类的中心值；

（4）对于所有的c个聚类中心，如果利用（2）（3）的迭代法更新后，值保持不变，则迭代结束，否则继续迭代。

该算法的最大优势在于简洁和快速。算法的关键在于初始中心的选择和距离公式。


## 算法流程：
>首先从n个数据对象任意选择 k 个对象作为初始聚类中心；而对于所剩下其它对象，则根据它们与这些聚类中心的相似度（距离），分别将它们分配给与其最相似的（聚类中心所代表的）聚类；然后再计算每个所获新聚类的聚类中心（该聚类中所有对象的均值）；不断重复这一过程直到标准测度函数开始收敛为止。一般都采用均方差作为标准测度函数. k个聚类具有以下特点：各聚类本身尽可能的紧凑，而各聚类之间尽可能的分开。


### 具体流程：
输入：k, data[n];

（1） 选择k个初始中心点，例如c[0]=data[0],…c[k-1]=data[k-1];

（2） 对于data[0]….data[n], 分别与c[0]…c[k-1]比较，假定与c[i]差值最少，就标记为i;

（3） 对于所有标记为i点，重新计算c[i]={ 所有标记为i的data[j]之和}/标记为i的个数；

（4） 重复(2)(3),直到所有c[i]值的变化小于给定阈值。



## 基于mapreduce的K-means算法的实现：
### KmeansMapper.java
```java
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

public class KmeansMapper extends Mapper<Object, Text, IntWritable, Text> {
    public void map(Object key, Text value, Context context)
    throws IOException, InterruptedException{
        String line = value.toString();
        String[] fields = line.split("\t");
        List<ArrayList<Float>> centers = Assistance.getCenters(context.getConfiguration().get("centerpath"));
        int k = Integer.parseInt(context.getConfiguration().get("kpath"));
        float minDist = Float.MAX_VALUE;
        int centerIndex = 0;
        //计算样本点到各个中心的距离，并把样本聚类到距离最近的中心点所属的类
        for (int i = 0; i < k; ++i){
            float currentDist = 0;
            for (int j = 0; j < fields.length; ++j){
                float tmp = Math.abs(centers.get(i).get(j) - Float.parseFloat(fields[j]));
                currentDist += Math.pow(tmp, 2);
            }
            if (currentDist<minDist ){
                minDist = currentDist;
                centerIndex = i;
            }
        }
        System.out.println("Mapper输出的键值对："+centerIndex+"——>"+value.toString());
        context.write(new IntWritable(centerIndex), new Text(value));
    }
}
```

### KeansReducer.java
```java
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

public class KmeansReducer extends Reducer<IntWritable, Text, IntWritable, Text> {
    public void reduce(IntWritable key, Iterable<Text> value, Context context)
    throws IOException, InterruptedException{
        List<ArrayList<Float>> assistList = new ArrayList<ArrayList<Float>>();
        String tmpResult = "";
        for (Text val : value){
            String line = val.toString();
            String[] fields = line.split("\t");
            List<Float> tmpList = new ArrayList<Float>();
            for (int i = 0; i < fields.length; ++i){
                tmpList.add(Float.parseFloat(fields[i]));
            }
            assistList.add((ArrayList<Float>) tmpList);
        }
        //计算新的聚类中心
        for (int i = 0; i < assistList.get(0).size(); ++i){
            float sum = 0;
            for (int j = 0; j < assistList.size(); ++j){
                sum += assistList.get(j).get(i);
            }
            float tmp = sum / assistList.size();
            if (i == 0){
                tmpResult += tmp;
            }
            else{
                tmpResult += " " + tmp;
            }
        }
        Text result = new Text(tmpResult);
        context.write(key, result);
    }
}
```

### KmeansDriver.java
```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.util.GenericOptionsParser;

import java.io.IOException;

public class KmeansDriver{
    public static void main(String[] args) throws Exception{
        int repeated = 0;

        /*
        不断提交MapReduce作业指导相邻两次迭代聚类中心的距离小于阈值或到达设定的迭代次数
        */
        do {
            Configuration conf = new Configuration();
            String[] otherArgs  = new GenericOptionsParser(conf, args).getRemainingArgs();
            if (otherArgs.length != 6){
                System.err.println("Usage: <int> <out> <oldcenters> <newcenters> <k> <threshold>");
                System.exit(2);
            }
            conf.set("centerpath", otherArgs[2]);
            conf.set("kpath", otherArgs[4]);
            Job job = new Job(conf, "KMeansCluster");//新建MapReduce作业
            job.setJarByClass(KmeansDriver.class);//设置作业启动类

            Path in = new Path(otherArgs[0]);
            Path out = new Path(otherArgs[1]);
            FileSystem fs0 = out.getFileSystem(conf);
			fs0.delete(out,true);
            fs0.close();
            FileInputFormat.addInputPath(job, in);//设置输入路径
            /*FileSystem fs = FileSystem.get(conf);
            if (fs.exists(out)){//如果输出路径存在，则先删除之
                fs.delete(out, true);
            }*/
           /* FileSystem fs = out.getFileSystem(conf);
			
            fs.delete(out,true);
            fs.close();*/
            FileOutputFormat.setOutputPath(job, out);//设置输出路径

            job.setMapperClass(KmeansMapper.class);//设置Map类
            job.setReducerClass(KmeansReducer.class);//设置Reduce类

            job.setOutputKeyClass(IntWritable.class);//设置输出键的类
            job.setOutputValueClass(Text.class);//设置输出值的类

            job.waitForCompletion(true);//启动作业

            ++repeated;
            System.out.println("We have repeated " + repeated + " times.");
         } while (repeated < 300 && (Assistance.isFinished(args[2], args[3], Integer.parseInt(args[4]), Float.parseFloat(args[5])) == false));
        //根据最终得到的聚类中心对数据集进行聚类
        Cluster(args);
    }
    public static void Cluster(String[] args)
            throws IOException, InterruptedException, ClassNotFoundException{
        Configuration conf = new Configuration();
        String[] otherArgs  = new GenericOptionsParser(conf, args).getRemainingArgs();
        if (otherArgs.length != 6){
            System.err.println("Usage: <int> <out> <oldcenters> <newcenters> <k> <threshold>");
            System.exit(2);
        }
        conf.set("centerpath", otherArgs[2]);
        conf.set("kpath", otherArgs[4]);
        Job job = new Job(conf, "KMeansCluster");
        job.setJarByClass(KmeansDriver.class);

        Path in = new Path(otherArgs[0]);
        Path out = new Path(otherArgs[1]);
        FileInputFormat.addInputPath(job, in);
       /* FileSystem fs = FileSystem.get(conf);
        if (fs.exists(out)){
            fs.delete(out, true);
        }
        */
        FileSystem fs0 = out.getFileSystem(conf);
		fs0.delete(out,true);
        fs0.close();
        
        FileOutputFormat.setOutputPath(job, out);

        //将样本点聚类，不需要reduce操作，故不设置Reduce类
        job.setMapperClass(KmeansMapper.class);

        job.setOutputKeyClass(IntWritable.class);
        job.setOutputValueClass(Text.class);

        job.waitForCompletion(true);
    }
}
```

### 辅助类
### Assistance.java
```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FSDataInputStream;
import org.apache.hadoop.fs.FSDataOutputStream;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.util.LineReader;

import java.io.IOException;
import java.util.*;

public class Assistance {
	//读取聚类中心点信息：聚类中心ID、聚类中心点
    public static List<ArrayList<Float>> getCenters(String inputpath){
        List<ArrayList<Float>> result = new ArrayList<ArrayList<Float>>();
        Configuration conf = new Configuration();
        try {
            
            Path in = new Path(inputpath);
            FileSystem hdfs = in.getFileSystem(conf);
            FSDataInputStream fsIn = hdfs.open(in);
            LineReader lineIn = new LineReader(fsIn, conf);
            Text line = new Text();
            ArrayList<Float>  center = null;
            while (lineIn.readLine(line) > 0){
                String record = line.toString();
                center = new ArrayList<Float>();
                /*
				因为Hadoop输出键值对时会在键跟值之间添加制表符，
				所以用空格代替之。
                */
                String[] fields = record.split("\t");
                //List<Float> tmplist = new ArrayList<Float>();
                for (int i = 0; i < fields.length; ++i){
                    center.add(Float.parseFloat(fields[i]));
                }
                result.add(center);
            }
            fsIn.close();
        } catch (IOException e){
            e.printStackTrace();
        }
        return result;
    }

    //删除上一次MapReduce作业的结果
    public static void deleteLastResult(String path){
        Configuration conf = new Configuration();
        try {
            
            Path path1 = new Path(path);
            FileSystem hdfs = path1.getFileSystem(conf);
            hdfs.delete(path1, true);
        } catch (IOException e){
            e.printStackTrace();
        }
    }
    //计算相邻两次迭代结果的聚类中心的距离，判断是否满足终止条件
    public static boolean isFinished(String oldpath, String newpath, int k, float threshold)
    throws IOException{
        List<ArrayList<Float>> oldcenters = Assistance.getCenters(oldpath);
        List<ArrayList<Float>> newcenters = Assistance.getCenters(newpath);
        float distance = 0;
        int dimension=oldcenters.get(0).size();
        System.out.println("簇数:"+k);
        System.out.println("维度数:"+dimension);
        for (int i = 0; i < k; ++i){
            for (int j = 0; j <dimension; ++j){
                float tmp = Math.abs(oldcenters.get(i).get(j) - newcenters.get(i).get(j));
                distance += Math.pow(tmp, 2);
            }
        }
        System.out.println("Distance = " + distance + " Threshold = " + threshold);
        if (distance < threshold)
            return true;
        /*
		如果不满足终止条件，则用本次迭代的聚类中心更新聚类中心
        */
        Assistance.deleteLastResult(oldpath);
        Configuration conf = new Configuration();
        //FileSystem hdfs = FileSystem.get(conf);
        Path path0 = new Path(newpath);
        FileSystem hdfs=path0.getFileSystem(conf);
        hdfs.copyToLocalFile(new Path(newpath), new Path("/home/hadoop/hadoop-tmp/oldcenter.data"));
        hdfs.delete(new Path(oldpath), true);
        hdfs.moveFromLocalFile(new Path("/home/hadoop/hadoop-tmp/oldcenter.data"), new Path(oldpath));
        return false;
    }
}
```