# CHARPTER 6 MapReduce Development


## Running on a Cluster
### 打包
Hadoop会在driver的classpath中搜索任务的jar包，用户也可以显式指定jar包的路径。在打包时可以将任何依赖打包到jar包的`lib`目录中，资源文件可以打包到`classes`目录。

**client classpath**由`hadoop jar *<jar>*`设置，包含三个部分：

- 任务jar包
- 任务jar包中的`lib`和`classes`目录
- 环境变量`HADOOP_CLASSPATH`（这个环境变量仅对driver JVM有效）

**task classpath**：map和reduce运行在不同的JVM中，`HADOOP_CLASSPATH`无效。

- 任务jar包
- 任务jar包中的`lib`和`classes`目录
- 通过`-libjars`参数指定的文件
- `DistributedCache`（old API）或`Job`实例的`addFileToClassPath()`方法指定的文件

**打包依赖**有三种方式：

- 打包到任务jar包中
- 打包到`lib`目录下
- 通过`HADOOP_CLASSPATH`和`-libjars`单独指定

**依赖优先级**：

- client classpath：将环境变量`HADOOP_USER_CLASSPATH_FIRST`设为`true`
- task classpath：`mapreduce.job.user.classpath.first`设为`true`

> 修改优先级后Hadoop也会优先加载这些类，所以可能会引起任务失败。

### 启动
启动app需要运行应用的driver，即`main()`方法所在的类。

```shell
% unset HADOOP_CLASSPATH  #不太懂为啥要 unset
% hadoop jar hadoop-examples.jar v2.MaxTemperatureDriver \
    -conf conf/hadoop-cluster.xml input/ncdc/all max-temp
```

### Job，Task，Task Attempt ID

- Job ID根据YARN生成，YARN application ID格式为：`application_*{YARN启动时间}*_*{任务序号}*`，如`application_1410450250506_0003`。Job ID则将`application`替换为`job`即可，如`job_1410450250506_0003`。
- Task ID将Job ID的`job`替换为`task`，并添加后缀，如：`task_1410450250506_0003_m_000003`。
- Task Attempt ID：一个task 可能会运行多次（失败重试），同一个task的不运行次数会分配一个task attempt ID，如：`attempt_1410450250506_0003_m_000003_0`。

### MapReduce Web UI
可以查看任务统计和日志，URL：[http://resource-manager-host:8088/]()。resource manager会默认在内存中保存10000条历史记录（`yarn.resourcemanager.max-completed-applications`可以修改）更早的记录会保存到job history中。Job history的文件由application master保存到HDFS上，路径为`mapreduce.jobhistory.done-dir`，并会保存一周的时间。这些历史记录可疑在Web上或通过`mapred job -history`命令查看。



### 获取结果
`-getmerge`参数可以将指定目录中的文件合并到一个文件中。`cat *`也可以在shell中查看。

### 日志

| Logs                       | Primary audience | Description                                                                                                             | Further information |
| :------------------------- | :--------------- | :---------------------------------------------------------------------------------------------------------------------- | :------------------ |
| System daemon logs         | Administrators   | 每个节点的守护进程都会使用log4j将日志写入`HADOOP_LOG_DIR`                                                                  |                     |
| HDFS audit logs            | Administrators   | 在namenode记录每个HDFS的请求                                                                                             |                     |
| MapReduce job history logs | Users            | 记录任务执行过程中发生的事件，存储在HDFS中                                                                                 |                     |
| MapReduce task logs        | Users            | 每个task子进程都会产生三种日志文件：syslog（使用log4j）、stdout、stderr，这三种日志都存储在`*{YARN_LOG_DIR}*/userlogs/`目录下 |                     |

YARN中提供了日志聚合的服务，所有task的日志都会被聚合并存储到HDFS中，设置`yarn.log-aggregation-enable`为`true`启用该服务。在服务启动情况下，可以在Web的*task attempt*界面或使用`mapred job -logs`命令查看。

用户代码可以将task的日志写入*syslog*，可以使用Apache Commons Logging API 或其他可以写入log4j的API。通过`mapreduce.map.log.level`和`mapreduce.reduce.log.level`改变日志输出级别，`yarn.nodemanager.log.retain-seconds`和`mapreduce.task.userlog.limit.kb`设置日志的滚动删除机制。

```shell
% hadoop jar hadoop-examples.jar LoggingDriver -conf conf/hadoop-cluster.xml \
    -D mapreduce.map.log.level=DEBUG input/ncdc/sample.txt logging-out
```

### Debugging

- Reproduce the failure locally  
将出问题的数据下载到本地复现
- Use JVM debugging options  
在启动任务时设置`mapred.child.java.opts`，` -XX:-HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/path/to/dumps`，在拿到dump文件之后使用jhat或其他工具分析。
- Use task profiling

`mapreduce.task.files.preserve.failedtasks`设置为 `true`保存挂掉task产生的文件，`mapreduce.task.files.preserve.filepattern`设置为`true`保存`filepattern`命中的文件（正则表达式）。

task attempt files保存在`mapreduce.cluster.local.dir property`目录下：`mapreduce.cluster.local.dir/usercache/user/appcache/application-ID/output/task-attempt-ID`

### Tuning
|                          |                          Best practice                          | Further information |
| ------------------------ | --------------------------------------------------------------- | ------------------- |
| Number of mappers        | 尽量让每个mapper的运行时长在一分钟                                 |                     |
| Number of reducers       | 每个reducer运行时长一般在五分钟才能产生足够一个区块的数据            |                     |
| Combiners                | 优化shuffle时需要传输的数据量                                     |                     |
| Intermediate compression | 优化数据传输量                                                   |                     |
| Custom serialization     | 如果使用了自定义的`Writable`对象，需要确认是否实现了`RawComparator` |                     |
| Shuffle tweaks           | 调整shuffle的参数                                                |                     |

## MapReduce Workflows
