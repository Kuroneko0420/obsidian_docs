

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