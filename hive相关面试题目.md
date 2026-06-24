
## spark为啥比mr块
DAG减少落盘、线程复用、内存缓存

## hive on spark 和 spark on hive

**hive on spark** Hive既作为存储⼜负责sql的解析优化，Spark负责执⾏
**Sparkon Hive** Hive只作为存储⻆⾊，Spark负责sql解析优化，执⾏

## Hive哪些操作会有shuffle？谈⼀谈Hive Shuffle 的具体过程

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

## **介绍下知道的Hive窗口函数，举一些例子(必背）**

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

## **窗口函数中加Order By和不加Order By的区别？（必背）**

**“加 Order By 本质上改变了窗口的‘视野范围（Window Frame）’。”**

- 如果窗口函数是**排序类**（`row_number` / `rank`），Order By 只负责**排列顺序**。
- 如果窗口函数是**聚合类**（`sum` / `avg` / `max`），Order By 会强制开启**累积窗口（从第一行到当前行）”**。

**第一类是排序函数，比如 `row_number()` 和 `rank()`。**  
这种情况下，`Order By` 是**强制必要**的，它定义了“第一行”的基准。如果不加，Hive 无法确定数据顺序，序号会随机生成，没有任何业务意义，所以这类函数极少单独脱离 `Order By` 使用。

**第二类是聚合函数，比如 `sum()`、`avg()` 和 `count()`，这是面试官最常关注的差异点。**  
不加 `Order By` 时，窗口范围默认是**整个分区（`PARTITION BY`）**。此时每一行计算出的聚合值都是相同的，那就是分区的总和或平均值。  
而一旦加了 `Order By`，窗口的默认范围会**自动变为“从分区第一行到当前行”**，也就是开启了一个**累积窗口**。结果会呈现逐行递增或递减的滚动状态，比如计算截止到每个月的累计销售额。

## **说说row_number()、rank() 、dense_rank()的区别。**

- ROW_NUMBER() 为查询结果中的每一行分配唯一的行号, 行号是连续的,没有重复值
- RANK() 为查询结果中的每一行分配排名值, 如果有相同的排序值, 则会跳级排名 . 
- DENSE_RANK()  为查询结果中的每一行分配密集排名值, 如果有相同的排序值, 则会连续排名, 不跳级.
## **请讲一下order by、sort by、distribute by、Cluster by的区别**

**order by：**
(1)全局排序；
(2)对输入的数据做排序，故此只有一个reducer(多个reducer无法保证全局有序)；会当输入规模较大时，消耗较长的计算时间。

**sort by：**
(1)非全局排序，局部有序，对单个reduce进行排序，reduce内有序；
(2)当设置mapred.reduce.tasks>1时，只能保证每个reducer的输出有序，不保证全局有序；（如果reduce=1，其实既是局部有序也是全局有序）

**distribute by：**
(1)按照指定的字段对数据进行划分输出到不同的reduce中；后面不能跟desc、asc排序；
(2)常和sort by一起使用，并且distribute by必须在sort by前面；（但是distribute by可以单独使用，比如一般公司中我们对指定的重复度高的列使用distribute by  c1,c2,进而提高列的压缩率，降低存储）

**cluster by:**
相当于distribute by+sort by，只能默认升序，不能使用倒序。

## Hive UDF 函数

Udf 函数单⾏操作，⼀对⼀输出 继承 UDF 类 , 重写 evalue 函数 
UDAF 函数是聚合函数 多对⼀，类继承 UDAF 类，似于 map 和 reduce 功能，先处理每⾏输出，然后局部聚合函数，再写全局聚合函数。 
UDTF 函数是爆炸函数，⼀对多，继承 UDTF 类，重写 process ⽅法 

## 小文件治理

**首先，小文件产生的根源主要有三个：**  
一是**流式数据写入**，比如Kafka或Flink高频微批处理写入Hive，几分钟生成一个文件；二是**任务并行度过高**，比如Reduce个数设置过大，或者Spark分区数远大于数据量，导致每个Task只输出极少数据；三是**动态分区**使用不当，比如按“小时”甚至“分钟”分区，每个分区下只有零星数据。

**其次，小文件的危害集中在两点：**  
一是**NameNode层面**，每个文件、每个Block都要占用NameNode约150字节的元数据内存，大量小文件会撑爆堆内存，导致集群重启缓慢甚至RPC超时；二是**计算层面**，HDFS的设计初衷是让寻址时间仅占传输时间的1%，但小文件会导致大量磁盘寻址开销，同时启动Map任务时也会产生巨大的调度损耗。


**接下来是治理手段，我分为“实时写入治理”和“历史存量治理”两部分：**

**对于实时写入任务**，我会在数据写入前做微批合并，使用Hive的`hive.merge.mapredfiles=true`参数，在写入结束后自动触发合并。

**对于历史存量文件**，我会直接使用`INSERT OVERWRITE`重写对应分区，并配合开启一系列合并参数。这里有个核心参数组合：
- 开启`hive.merge.mapfiles`和`hive.merge.mapredfiles`，让Hive在Map-only任务和MapReduce任务结束后都尝试合并；
- 设置`hive.merge.smallfiles.avgsize=32000000`，**这是触发合并的开关阈值**；
- 设置`hive.merge.size.per.task=256000000`，**这是合并后每个目标文件的大小**。

为什么配置了这些参数，任务跑完就会自动合并？因为Hive在**最后一个Job执行完成后**，会检查本次输出的**平均文件大小**。如果这个平均值小于`hive.merge.smallfiles.avgsize`，Hive就会**再启动一个仅含Map阶段的任务**，将输出文件重新读取并写入，写入时的Map切片大小由`mapred.max.split.size`控制。所以，如果我们想让合并生效，就必须让**Map的切片最小值大于平均文件阈值**，否则切出来的文件还是很小，合并就形同虚设。


