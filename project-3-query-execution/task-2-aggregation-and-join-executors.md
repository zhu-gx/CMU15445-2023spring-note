---
description: 聚合和联接执行程序
---

# task#2 Aggregation & Join Executors

## 要求

在此任务中，你将添加一个聚合执行器和多个连接执行器，并使优化器能够在规划查询时在嵌套循环联接和哈希联接之间进行选择。你将在以下文件中完成实现：

* `src/include/execution/aggregation_executor.h`
* `src/execution/aggregation_executor.cpp`
* `src/include/execution/nested_loop_join_executor.h`
* `src/execution/nested_loop_join_executor.cpp`
* `src/include/execution/hash_join_executor.h`
* `src/execution/hash_join_executor.cpp`
* `src/optimizer/nlj_as_hash_join.cpp`

下面介绍每项任务。

## Aggregation 聚集

`AggregationPlanNode`支持如下的查询：

```sql
EXPLAIN SELECT colA, MIN(colB) FROM __mock_table_1 GROUP BY colA;
EXPLAIN SELECT COUNT(colA), min(colB) FROM __mock_table_1;
EXPLAIN SELECT colA, MIN(colB) FROM __mock_table_1 GROUP BY colA HAVING MAX(colB) > 10;
EXPLAIN SELECT DISTINCT colA, colB FROM __mock_table_1;
```

请注意，聚合执行器本身不需要处理`having`谓词。计划其将`having`是为`FilterPlanNode`。聚合执行器只需要为输入的每个组执行聚合。它只有一个子节点。

聚合的模式是先分组列，然后是聚合列。

如[lecture#10](https://15445.courses.cs.cmu.edu/fall2022/slides/)所述，实现聚合的常见策略是使用哈希表。在本项目中，你将使用这种方法，但是我们做出了简化假设，即聚合哈希表完全适合内存。这意味着你不需要担心为哈希聚合实现两阶段策略（分区，重新哈希）。你还可以假设所有聚合结果都可以驻留在内存中的哈希表中（即哈希表不需要由缓冲池页面支持）。

我们提供了`SimpleAggregationHashTable`数据结构，它公开了一个内存中的哈希表（`std::unordered_map`），但其接口专门设计用于计算聚合。该类还公开了`SimpleAggregationHashTable::Iterator`类型，可用于遍历哈希表。你需要填写该类的`CombineAggregateValues`函数。

提示：

* 请记住，在查询计划的上下文中，聚合是管道breakers。这可能会影响你在实现中使用`AggregationExecutor::Init()`和`AggregationExecutor::Next()`函数的方式。特别是，请考虑聚合的构建阶段是应该在`AggregationExecutor::Init()`还是`AggregationExecutor::Next()`中执行。
* 你必须考虑如何处理聚合函数输入中的NULL值（即：元组的属性值可能为NULL）。请参阅测试用例以了解期望的行为。分组列永远不会为NULL。
* 在空表上执行聚合时，`CountStarAggregate`应返回零，而所有其他聚合类型应返回`integer_null`。这就是为什么`GenerateInitialAggregateValue`将大多数聚合值初始化为`NULL`的原因。

### 目标文件

* `src/include/execution/aggregation_executor.h`
* `src/execution/aggregation_executor.cpp`

### 思路

* 先实现group by（分组），再实现聚集(5个算子)：
  * min()
  * max()
  * sum()
  * count(列)
  * count(\*)

### 代码

* 成员变量包括：
  * plan node
  * 孩子执行器
  * 哈希表
  * 迭代器
  * 针对是否空表的bool
* 构造函数对成员变量初始化
  * 哈希表
  * 迭代器：哈希表的begin
* Init
  * 把哈希表的分组分开：获取子节点的tuple\&rid（Next方法）
* Next
  * 只要迭代器没有遍历到末尾，每次获取一个哈希表中的key

## NestedLoopJoin 嵌套循环连接

默认情况下，DBMS将用于NestedLoopJoinPlanNode所有联接操作。以下是示例查询：

```sql
EXPLAIN SELECT * FROM __mock_table_1, __mock_table_3 WHERE colA = colE;
EXPLAIN SELECT * FROM __mock_table_1 INNER JOIN __mock_table_3 ON colA = colE;
EXPLAIN SELECT * FROM __mock_table_1 LEFT OUTER JOIN __mock_table_3 ON colA = colE;
```

你需要使用lecture中提到的基本嵌套循环连接算法来为NestedLoopExecutor实现inner join和left join。此操作符的输出模式是左表中的所有列，后跟右表中的所有列。

该执行器应该实现lecture#11中介绍的简单嵌套循环连接算法。也就是说，对于连接的外部表中的每个元组，应该考虑连接的内部表中的每个元组，并在满足连接谓词时输出一个元组。

提示：

* 你将需要使用`NestedLoopJoinPlanNode`中的谓词。特别地，请查看`AbstractExpression::EvaluateJoin`，它处理左元组和右元组及其各自的模式。请注意，此函数返回一个值，可能为false，true或者NULL。请参阅FilterExecutor以了解如何在元组上应用谓词。
* [SQL中左联接、右联接、等值联接的区别](https://www.jianshu.com/p/e7e6ce1200a4)
* [火山模型概念简介](https://zhuanlan.zhihu.com/p/478851521)
* [执行流程动画](https://cs186berkeley.net/resources/join-animations/)

### 思路

* 先在外层的Next函数调用tuple，然后在内层用for循环遍历缓冲到内存中的数组。匹配成功返回，用变量记录for循环遍历到的位置，接着往下继续遍历，防止漏匹配。

