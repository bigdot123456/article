个人博客原文：[64位Ubuntu14.04下安装hadoop2.6单机配置和伪分布配置详解](http://blog.taohuawu.club/article/36)

# 环境

## 系统： Ubuntu 14.04 64bit

## Hadoop版本： Hadoop 2.6.0 (stable)

## JDK版本： oracle jdk7

# 操作
## 在Ubuntu下创建hadoop用户组和用户
1. 创建hadoop用户组
```
sudo addgroup hadoop
```
2. 创建hadoop用户
```
sudo adduser -ingroup hadoop hadoop
```
３. 给hadoop用户添加权限，打开/etc/sudoers文件
```
sudo gedit /etc/sudoers
```
在root   ALL=(ALL:ALL)   ALL下添加hadoop   ALL=(ALL:ALL)  ALL.

## 安装SSH server、配置SSH无密码登陆
>ssh 是一个很著名的安全外壳协议 Secure Shell Protocol。 rsync 是文件同步命令行工具

```
sudo apt-get install ssh rsync
```

切换到hadoop用户
```
su - hadoop
```
配置 ssh 免登录
```
ssh-keygen -t dsa -f ~/.ssh/id_dsa
```


执行这条命令生成 ssh 的公钥/私钥,执行过程中,会一些提示让输入字符,直接一路回车就可以

    cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys


ssh 进行远程登录的时候需要输入密码,如果用公钥/私钥方式,就不需要输入密码了。上述方式就是设置公钥/私钥登录。

    ssh localhost


第一次执行本命令,会出现一个提示,输入 ” yes”然后回车即可

## 安装hadoop2.6

下载Hadoop 2.6.0后,解压到/usr/local/中

```
sudo tar -zxvf hadoop-2.6.0.tar.gz
sudo mv hadoop-2.6.0 /usr/local/hadoop
sudo chmod -R 775 /usr/local/hadoop
sudo chown -R hadoop:hadoop /usr/local/hadoop  //否则ssh会拒绝访问
```

Hadoop解压后即可使用。输入如下命令Hadoop检查是否可用，成功则会显示命令行的用法：

    /usr/local/hadoop/bin/hadoop

环境变量配置

    sudo gedit ~/.bashrc





在文件尾添加以下内容：

```
#HADOOP VARIABLES START

export JAVA_HOME=/usr/lib/jvm/jdk1.8.0_25

export HADOOP_INSTALL=/usr/local/hadoop

export PATH=$PATH:$HADOOP_INSTALL/bin

export PATH=$PATH:$HADOOP_INSTALL/sbin

export HADOOP_MAPRED_HOME=$HADOOP_INSTALL

export HADOOP_COMMON_HOME=$HADOOP_INSTALL

export HADOOP_HDFS_HOME=$HADOOP_INSTALL

export YARN_HOME=$HADOOP_INSTALL

export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_INSTALL/lib/native

export HADOOP_OPTS="-Djava.library.path=$HADOOP_INSTALL/lib"

#HADOOP VARIABLES END
```


执行以下命令时刚才的配置生效：

    source ~/.bashrc




修改hadoop-env.sh的配置：

    sudo gedit /usr/local/hadoop/etc/hadoop/hadoop-env.sh


将JAVA_HOME修改成配置文件里面的内容．

## 测试
通过执行hadoop自带实例WordCount验证是否安装成功
/usr/local/hadoop路径下创建input文件夹

```
mkdir input
cp README.txt input
```




在hadoop目录下执行WordCount：
```
bin/hadoop jar share/hadoop/mapreduce/sources/hadoop-mapreduce-examples-2.6.0-sources.jar
org.apache.hadoop.examples.WordCount input output
```

Hadoop伪分布式配置
输入：sudo gedit /usr/local/hadoop/etc/hadoop/core-site.xml

```
<configuration>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/usr/local/hadoop/tmp</value>
        <description>Abase for other temporary directories.</description>
    </property>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
```



sudo gedit /usr/local/hadoop/etc/hadoop/mapred-site.xml //此项非必要 
```
<configuration>
 <property>   
      <name>mapred.job.tracker</name>  
      <value>localhost:9001</value>   
     </property>   
</configuration>
```



sudo gedit /usr/local/hadoop/etc/hadoop/yarn-site.xml


```
<configuration>
<property>
<name>mapreduce.framework.name</name>
<value>yarn</value>
</property>

<property>
<name>yarn.nodemanager.aux-services</name>
<value>mapreduce_shuffle</value>
</property>
</configuration>
```



sudo gedit /usr/local/hadoop/etc/hadoop/hdfs-site.xml

```
<configuration>
<property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:/usr/local/hadoop/dfs/name</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:/usr/local/hadoop/dfs/data</value>
    </property>
    <property>                 //这个属性节点是为了防止后面eclopse存在拒绝读写设置的
            <name>dfs.permissions</name>
            <value>false</value>
     </property>
#启用webhdfs，可用web接口管理hdfs文件系统
<property>
<name>dfs.webhdfs.enabled</name>
<value>true</value>
</property>

 </configuration>
```


sudo gedit /usr/local/hadoop/etc/hadoop/masters 添加：localhost

sudo gedit /usr/local/hadoop/etc/hadoop/slaves  添加：localhost



关于配置的一点说明：上面只要配置 fs.defaultFS 和 dfs.replication 就可以运行，不过有个说法是如没有配置 hadoop.tmp.dir 参数，此时 Hadoop 默认的使用的临时目录为 /tmp/hadoo-hadoop，而这个目录在每次重启后都会被干掉，必须重新执行 format 才行（未验证），所以伪分布式配置中最好还是设置一下。

配置完成后，首先在 Hadoop 目录下创建所需的临时目录：
```
cd /usr/local/hadoop
mkdir tmp dfs dfs/name dfs/data
```

初始化文件系统HDFS
```
bin/hdfs namenode -format
```


成功的话，最后的提示如下，Exitting with status 0 表示成功，Exitting with status 1: 则是出错
```
sbin/start-dfs.sh
sbin/start-yarn.sh //启动map-reduce框架yarn，这个必须要启动否则后面无法运行ｗordcount程序
```


>Unable to load native-hadoop library for your platform这个提示,解决方式：

1. 重新编译源码后将新的lib/native替换到集群中原来的lib/native
2. 修改hadoop-env.sh ，增加
  export HADOOP_OPTS="-Djava.library.path=$HADOOP_PREFIX/lib:$HADOOP_PREFIX/lib/native"

Namenode information: 通过http://localhost:50070 来查看Hadoop的信息。

All Applications：http://2xx.81.8x.1xx:8088/， 将其中的2xx.81.8x.1xx替换为你的实际IP地址。



运行例子：
1. 先在hdfs上建个文件夹  bin/hdfs dfs -mkdir -p /user/ha1/input
  bin/hdfs dfs -mkdir -p /user/ha1/output
2. 上传一些文件：bin/hdfs dfs -put etc/hadoop/  /user/ha1/input  把etc/hadoop文件上传到hdfs的/user/ha1/input中
3. 执行指令
```
bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.6.0.jar grep /user/ha1/input/hadoop  /user/ha1/output/temp 'dfs[a-z.]+'
```
4. 查看结果
```
bin/hdfs dfs -cat /user/ha1/output/temp/*
```

8	dfs.audit.logger  
4	dfs.class  
3	dfs.server.namenode  
2	dfs.audit.log.maxbackupindex  
2	dfs.period  
2	dfs.audit.log.maxfilesize  
1	dfsmetrics.log  
1	dfsadmin  
1	dfs.servers  
1	dfs.replication  
1	dfs.file  
1	dfs.datanode.data.dir  
1	dfs.namenode.name.dir

