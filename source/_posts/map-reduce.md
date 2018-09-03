---
title: MapReduce 学习
date: 2018-08-22 20:18:35
tags: MapReduce,分布式计算
---

### 作者 Jeff Dean

!["Jeff dean"](http://pegw4tvmf.bkt.clouddn.com/image/jeff_dean.jpg)

#### 主要贡献

- [MapReduce](https://en.wikipedia.org/wiki/MapReduce)  	大规模数据处理系统，并行计算编程模型
- [Bigtable](https://en.wikipedia.org/wiki/Bigtable)  结构化的分布式数据库
- [Spanner](https://en.wikipedia.org/wiki/Spanner_(distributed_database_technology)) 第一个全球性的分布式数据库
- [Google Brain](https://en.wikipedia.org/wiki/Google_Brain) 
- [LevelDB](https://en.wikipedia.org/wiki/LevelDB) 
- [TensorFlow](https://en.wikipedia.org/wiki/TensorFlow) 
- Compilers don’t warn Jeff Dean.   Jeff Dean warns compilers.

------



### MapReduce 介绍

- 抽象的编程模型

- 并行计算

- 容错 & 分布可靠

- 自动负载均衡

- 很容易的进行分布式开发


##### 例：单词统计

伪代码

```c++
map(String key, String value):
    // key: document name
    // value: document contents
    for each word w in value:
    EmitIntermediate(w, "1");

reduce(String key, Iterator values):
    // key: a word
    // values: a list of counts
    int result = 0;
    for each v in values:
        result += ParseInt(v);
        Emit(AsString(result));
```

论文中的c++代码：

```c++
#include "mapreduce/mapreduce.h"

// User’s map function
class WordCounter : public Mapper {
    public:
        virtual void Map(const MapInput& input) {
            const string& text = input.value();
            const int n = text.size();
            for (int i = 0; i < n; ) {
                // Skip past leading whitespace
                while ((i < n) && isspace(text[i]))
                    i++;
                // Find word end
                int start = i;
                while ((i < n) && !isspace(text[i]))
                    i++;
                if (start < i)
                    Emit(text.substr(start,i-start),"1");
            }
        }
};
REGISTER_MAPPER(WordCounter);

// User’s reduce function
class Adder : public Reducer {
    virtual void Reduce(ReduceInput* input) {
        // Iterate over all entries with the
        // same key and add the values
        int64 value = 0;
        while (!input->done()) {
            value += StringToInt(input->value());
            input->NextValue();
        }
        // Emit sum for input->key()
        Emit(IntToString(value));
    }
};
REGISTER_REDUCER(Adder);
```



##### 其他应用例子：

分布式的Grep：

从日志统计url访问频率：

文档索引：

分布式排序

 

### 实现



#### 流程

原论文中插图：![image-20180823020032100](http://pegw4tvmf.bkt.clouddn.com/image/map_reduce_figure1.png)

盗用一张说明Hadoop的图 ![img](http://pegw4tvmf.bkt.clouddn.com/image/hadoop_map_reduce.png)

1. 将输入文件拆分成16到64M/块，用户程序拷贝到集群中的机器上

2. master 给一个空闲的worker分配任务，map task或reduce task

3. worker (map task) 解析输入数据，传递给用户自定义Map函数处理，中间结果缓存在内存中

4. 周期性地将内存中缓存的k/v对写到本地磁盘，分成R个区域里，并且告诉master保存的位置，master再将数据保存位置告诉reduce worker

5. 根据master的告知的中间结果文件位置，reduce worker通过RPC从map worker的磁盘读取数据，进行排序

6. 将排好序的中间数据，将每个唯一的中间key值和相关的value值传递给用户自定义reduce函数处理，输出追加到所属分区的输出文件（GFS文件系统）

7. Map和Reduce任务都完成之后，master再返回到用户程序



#### 容错

##### worker 故障

- master心跳监测

- 未完成的task分配给新的worker重新执行
- 已完成的map task需要重新执行，并通知相应的reduce worker
- 已完成的reduce task不需要重新执行

##### master故障

- 只有一台，出错概率小，可以重新执行整个MapReduce操作

- 设置checkpoint，将task 和 worker的状态保存到文件，重启master后可以从上个状态继续执行



#### 技巧

- 分配map任务尽量选保存了输入数据的机器，reduce任务尽量分配给相邻的机器
- 合适的任务粒度（M个map task，R个reduce task）利于提高集群的动态的负载均衡能力，加快故障恢复的速度
- 顺序保证，对于按 key 值随机存取，排序输出等很有帮助
- 快结束时执行备用任务，能有效防止某些worker的task拖慢整个操作
- 跳过损坏记录，worker向master汇报导致crash的输入数据和异常原因，segmentdefault 等，master不再分配这些任务
- 对于有一致性要求的任务，必须是确定性的任务。 原子性与幂等性保证：worker完成map task后向master提交，master只会记录一个成功提交，忽略其他提交。reduce 输出到文件完成后再rename成最终文件，结果是一致的



### 练习

[MIT 6.824 Lab 1](https://pdos.csail.mit.edu/6.824/labs/lab-1.html)   

实现框架中的部分流程、利用MapReduce完成计算任务，自动测试



### 总结

虽然思想看起来比较朴素，但是对于分布式大数据处理领域意义重大。

“一切都要从Google发的那篇论文说起”。

抽象出来的mapper和reducer很简单，大大降低了分布式开发的门槛。



下一代更优秀的大数据处理引擎：Spark、Flink等

------



### 相关参考资料:

> 1. Jeffrey Dean & Sanjay Ghemawat. 《MapReduce》(2004) [英文版](https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/mapreduce-osdi04.pdf)      [中文版](http://blog.bizcloudsoft.com/wp-content/uploads/Google-MapReduce%E4%B8%AD%E6%96%87%E7%89%88_1.0.pdf)
>
> 2. [MIT 6.824: Distributed Systems LEC 1](https://pdos.csail.mit.edu/6.824/schedule.html)
>
> 3. [MapReduce: The programming model and practice ](https://ai.google/research/pubs/pub36249)
>
> 4. [Wikipedia: Jeff Dean](https://en.wikipedia.org/wiki/Jeff_Dean_(computer_scientist))
>
> 5. [与 Hadoop 对比，如何看待 Spark 技术？](https://www.zhihu.com/question/26568496)
>
> 6. [大数据框架对比：Hadoop、Storm、Samza、Spark和Flink](http://www.infoq.com/cn/articles/hadoop-storm-samza-spark-flink)
>

