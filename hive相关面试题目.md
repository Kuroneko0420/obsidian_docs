
# spark为啥比mr块
DAG减少落盘、线程复用、内存缓存

# hive on spark 和 spark on hive

**hive on spark** Hive既作为存储⼜负责sql的解析优化，Spark负责执⾏
**Sparkon Hive** Hive只作为存储⻆⾊，Spark负责sql解析优化，执⾏

# Hive哪些操作会有shuffle？谈⼀谈Hive Shuffle 的具体过程

join 、Group By、 ORDER BY（全局排序） 、COUNT(DISTINCT)、Partition by、Distribute by 把相同的key shuffle到同一个reduce中、count,avg,sum这类统计聚合函数也会产生shuffle的，但是不会出现数据数据倾斜而已

 **好的，Shuffle是Map端输出到Reduce端输入的中间过程，核心是**分区、排序、合并**。我分为两端来说：**

**第一，Map端的Shuffle（输出阶段）：**

1. **写入环形缓冲区**：Map方法输出的是`<Key,Value>`，先写入内存中的**环形缓冲区**（默认100MB）。     
2. **溢写（Spill）并排序分区**：当缓冲区达到阈值（默认80%）时，后台线程开始溢写到磁盘。在溢写前，会先**按分区（Partition，即Reduce个数）** 进行划分，并在每个分区内部**按Key进行快速排序**。
3. **归并（Merge）**：一个Map可能会溢写出多个小文件，最终这些文件会**归并成大文件**。归并过程中依然会进行排序和分区，且如果设置了Combiner，会在此时提前进行一次本地聚合，减少网络传输量。
**第二，Reduce端的Shuffle（拉取与合并阶段）：**
4. **拉取（Copy）**：Reduce启动后，会主动向各个Map节点**拉取**属于自己分区的那部分数据（通过HTTP方式）。
5. **归并排序（Merge Sort）**：拉取过来的数据先放内存，内存不够则溢写磁盘。Reduce会把来自不同Map的、同一分区的**所有文件进行归并排序**，形成一个全局有序的大文件（或内存数据块）。 
6. **分组（Group）**：归并完成后，Shuffle结束。Reduce进入`reduce()`方法，将**相同Key的Value聚合在一起（形成迭代器）**，进行业务逻辑处理。

# **介绍下知道的Hive窗口函数，举一些例子(必背）**

分析函数/**专用窗口函数 over(partition by** 列名 **order** **by** 列名 **rows** **between** 开始位置 **and** 结束位置)
**常见Hive窗口函数分类：****聚合开窗函数**、**排序开窗函数、取数函数**。

1**聚合开窗函数**
- count()over();   -- 窗口内总条数
- sum()over();   -- 窗口内数据的和
- min()over();   -- 窗口内最小值
- max()over();   -- 窗口内最大值
- avg()over();   -- 窗口内的平均值

**2. 排序开窗函数**
- row_number();     -- 从1开始，按照顺序，生成分组内记录的序列
- rank();         -- 生成数据项在分组中的排名，排名相等会在名次中留下空位
- dense_rank();    -- 生成数据项在分组中的排名，排名相等会在名次中不会留下空位
- 
**3.位移窗口函数**
- lag() over()：取向上第n行的数据
- lead()over()：取向下第n行的数据

**4.极值窗口函数**
-  first_value(col,true/false) over()：取分组内排序后，截止到当前行，第一个值。
-  last_value(col,true/false) over()：取分组内排序后，截止到当前行，最后一个值

**5.分箱窗口函数**
-  ntile() over() 分箱窗口函数，用于将分组数据按照顺序切分成 n 片，返回当前切片值，如果切片不均匀，默认增加到第一个切片中。