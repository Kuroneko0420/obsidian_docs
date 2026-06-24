

### 一段sql启动的job/stage 个数

**一段sql的Job 的划分主要看：**

1.  **正常一段SQL最终产生多少个job主要由a**ction算子、shuffle 边界、以及部分特殊算子** 共同决定****。**
2. **一个 Spark SQL 查询通常由一个 action 触发一个或多个 Job。*
3. **实际正常一段SQL具体产生多少个job其实是不明确的，主要是看****是否触发了 Action算子，以及****执行过程中是否有需要单独提交的阶段****：
    1. **首先广播join的时候broadcast exchange 可能触发额外 job，因为 Spark 要先把小表 `b` 算出来、收集到 Driver，再广播给各个 Executor。这个“先计算小表”的过程，往往会单独触发一个 Job，然后主表 join 再触发后续 Job。
    2. **其子查询可能触发额外 job，比如where in(select xxxxxx from xxxx)，子查询有时需要先执行，得到结果后，主查询才能继续，所以子查询可能单独生成 Job。
    3. **其次写入insert overwrtie这种往往会一个最终写入job；**
    4. **其次AQE开启以后就会优化很多job的执行，某些自适应执行/AQE 场景下，也可能看到额外的执行提交
4. **Stage的划分是比较明确的主要 Shuffle 边界划分

![[Pasted image 20260621164159.png]]



![[Pasted image 20260621170748.png]]

### **任务变慢除了数据倾斜还有原因导致？**

#### **1、YARN/对列资源不足导致并行度不足任务慢？**

**参数与并行度等配置都没有问题（不是瓶颈），主要是因为队列资源不足，只是因为队列资源不足，任务抢不到资源，导致起的excutor过少，task并行执行的task数过少，导致任务整体“串行执行”拖长执行时间；**

1. **任务执行中： 直接看队列的资源使用情况是否打满 + 当前这个任务占用资源情况（可以任务执行日志看下正在运行的stage的 task的并行度）；**
2. **任务执行完：**

	a. 我们看下任务的task的summary metrics信息，如果没有特别异常的task拖累整个stage，排出长尾问题导致的变慢。
	
	b. 这个时候我们看task的launchtime，如果task的launchtime间隔时间比较久，一般是stage 开始时资源少，后面逐渐拿到 executor，这时 task launch time 会拉得很开，但单个 task 真正被分配后 Scheduler Delay 仍可能很小
	
	c. 这个时候我们再看evet timeline看下task的Executors 数量是逐步增加的，Excutor的数量逐步增加；

#### **2、Executor个数不足设置导致并行度低变慢？**

**比如最maxExecutors个数使用时集群默认值比如500，那么对于绝大部分任务没问题，但是超级大任务，这个最大值往往会限制并行度；所以针对这种场景我们需要针对单个任务调整大这个值；**

**排查思路：直接查看Enviroment界面动态资源调整，Maxexcutors最大值，是否设置的过小；


#### **3、Task处理数据量或者内存设置不合理导致任务执行慢？**

**某个job/stage的所有task任务整体看所有的task的执行都很慢，那么基本都是单个task处理的数据量过多导致的，一般企业一个task处理一个block大小的文件，但是有些复杂业务场景，比如各种清洗加密复杂计算，这个时候task处理起来就很慢，这个时候可以适当增加task的内存和cpu，或者让每个task处理更少的数据。**



### **Spark某个Job/Stage执行很慢汇总案例排查？**
![[Pasted image 20260624180222.png]]


### 生产环境中导致task失败的场景汇总（熟悉）？

####  1.数据本身质量有问题？

- JSON/CSV/Parquet 格式损坏，文件损坏；
- 某一列数据类型不符合预期
- 反序列化失败，比如某一行某个字段长度太大，超过protocol序列化最大值；
- 某个 partition 里有脏数据，比如表情包符号乱码等；

####  2.用户代码异常

- `NullPointerException`
- `ArrayIndexOutOfBoundsException`
- `ClassCastException，数据类型转换异常；`
- `IllegalArgumentException`
- `ArithmeticException,比如计算的分母出现了0；`
- 比如自定义 UDF/UDTF/UDAF代码抛异常

#### 3.Shuffle / 依赖数据问题

- fetch拉 shuffle 数据时失败
- 比如读取数据block 丢失，不可用；
- 因为集群机器原因，网络风暴等，导致task处理网络超时
- executor 挂掉（因为资源争抢背kill，或者NM节点异常，机器宕机）

#### 4. 资源相关问题

- **task处理数据量过大，导致OOM（最常见），**`GC overhead limit exceeded等；`
- executor 内存不足导致进程被杀
- 本地磁盘满了
- 临时文件写入失败
- shuffle spill 失败

#### 5.依赖 / 环境问题

- 缺少 jar 包
- class not found
- 依赖版本冲突
- Python 环境异常（PySpark）
- native library 加载失败
- 特定机器上的环境变量不一致