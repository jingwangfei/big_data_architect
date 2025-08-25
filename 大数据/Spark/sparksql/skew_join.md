# Spark Skew Join 优化原理深度解析
  

### **一、问题背景：为什么会有 Skew Join？**
当 Join 的两个数据集（左表、右表）的分区数据分布极不均衡时，就会出现 **数据倾斜**：  
- 比如左表的某个分区包含 10GB 数据，而其他分区只有 100MB；  
- 该分区对应的 Join 任务会成为“长尾任务”，拖慢整个作业。  

Spark AQE 的 `OptimizeSkewedJoin` 会在运行时检测这种倾斜，并通过 **拆分倾斜分区** 来平衡任务负载。


### **二、OptimizeSkewedJoin 核心流程**
从源码来看，`OptimizeSkewedJoin` 的优化分为 **4 个核心步骤**，我们逐一拆解：  

#### **步骤 1：判断是否允许拆分分区**  

并非所有 Join 类型都能拆分分区，需根据 **Join 类型** 判断左右表是否可拆分：  
- **内连接（Inner）、交叉连接（Cross）**：左右表均可拆分（保留匹配数据即可）；  
- **左外连接（Left Outer）**：右表不能拆分（需保留所有右表数据，拆分可能导致空匹配丢失）；  
- **右外连接（Right Outer）**：左表不能拆分（类似左外连接逻辑）。  

代码中通过 `canSplitLeft` 和 `canSplitRight` 方法判断：  
```scala
val canSplitLeft = canSplitLeftSide(joinType)
val canSplitRight = canSplitRightSide(joinType)
if (!canSplitLeft && !canSplitRight) return None // 无法拆分，直接返回
```  


#### **步骤 2：检测数据倾斜**  
<img width="2834" height="1718" alt="image" src="https://github.com/user-attachments/assets/f1d8881c-5d96-4850-820a-31d559ea5e5e" />

要判断分区是否倾斜，需先获取 **分区大小** 并计算 **基准值（中位数）** 和 **倾斜阈值**：  

1. **获取分区大小**：  
   通过 `mapStats.bytesByPartitionId` 获取左、右表每个分区的大小（字节数）：  
   ```scala
   val leftSizes = left.mapStats.get.bytesByPartitionId
   val rightSizes = right.mapStats.get.bytesByPartitionId
   ```  

2. **计算中位数（medianSize）**：  
   中位数比平均值更稳定（避免极值干扰），作为判断倾斜的基准：  
   ```scala
   val leftMedSize = medianSize(leftSizes)  // 左表分区大小的中位数
   val rightMedSize = medianSize(rightSizes) // 右表分区大小的中位数
   ```  

3. **计算倾斜阈值（Skew Threshold）**：  
   通常为 **中位数的 N 倍（如 5 倍）**，超过该值的分区被判定为“倾斜分区”：  
   ```scala
   val leftSkewThreshold = getSkewThreshold(leftMedSize)
   val rightSkewThreshold = getSkewThreshold(rightMedSize)
   ```  


#### **步骤 3：计算目标分区大小（Target Size）**  
<img width="2732" height="1660" alt="image" src="https://github.com/user-attachments/assets/64feadaa-5128-4822-8178-3d3a0d2346df" />

拆分倾斜分区时，需确定 **每个子分区的理想大小**（即 `targetSize`），让拆分后的子分区数据量更均衡。计算逻辑结合 **原始分区大小** 和 **倾斜阈值**，确保子分区既不太小（避免任务过多）也不太大（避免长尾）。  


#### **步骤 4：遍历分区，处理倾斜**  
<img width="2682" height="1490" alt="image" src="https://github.com/user-attachments/assets/deb03c7c-3881-4833-aded-f64455668b79" />

对每个分区索引，检查是否倾斜，并决定是否拆分：  

1. **判断是否倾斜**：  
   结合“是否允许拆分”和“分区大小是否超过阈值”：  
   ```scala
   val isLeftSkew = canSplitLeft && leftSize > leftSkewThreshold
   val isRightSkew = canSplitRight && rightSize > rightSkewThreshold
   ```  

2. **非倾斜分区**：  
   使用 `CoalescedPartitionSpec`，表示“单个分区范围”（从自身到自身+1）：  
   ```scala
   val leftNoSkewPartitionSpec = Seq(CoalescedPartitionSpec(partitionIndex, partitionIndex + 1, leftSize))
   ```  

3. **倾斜分区：拆分逻辑**  
   调用 `ShufflePartitionsUtil.createSkewPartitionSpecs` 拆分，核心是 **`splitSizeListByTargetSize` 方法**：  
   - 输入：分区内的元素大小列表 `sizes`、目标大小 `targetSize`；  
   - 输出：分区起始索引数组（如 `[0,3,5]` 表示子分区为 `[0-2]`、`[3-4]`、`[5-末尾]`）。  

   **拆分逻辑**（简化版）：  
   ```scala
   while (i < sizes.length) {
     if (当前分区+下一个元素会超过 targetSize) {
       尝试合并相邻小分区（避免过多子分区）
       记录新分区的起始索引
       currentPartitionSize = 下一个元素大小
     } else {
       currentPartitionSize += 下一个元素大小
     }
     i += 1
   }
   ```  
   该逻辑确保每个子分区的总大小 **尽可能接近 targetSize**，同时避免产生过多极小分区（通过 `tryMergePartitions` 合并小分区）。  




### **三、关键工具类：ShufflePartitionsUtil**  

<img width="2806" height="1662" alt="image" src="https://github.com/user-attachments/assets/01ed162c-e30c-4280-a806-7acc7e2d480f" />

`ShufflePartitionsUtil` 是拆分倾斜分区的核心工具，其中 **`splitSizeListByTargetSize` 方法** 实现了“按目标大小拆分分区”的逻辑：  

1. **输入**：分区内各元素的大小列表 `sizes`、目标大小 `targetSize`；  
2. **输出**：子分区的起始索引数组（标记每个子分区从哪里开始）；  
3. **核心逻辑**：遍历 `sizes`，累加元素大小，当接近 `targetSize` 时分割，同时处理小分区合并（避免碎片化）。  
<img width="2750" height="1570" alt="image" src="https://github.com/user-attachments/assets/8735df97-96c4-463d-bbc5-28e48b598777" />




### **四、总结：Skew Join 优化的本质**  
Spark 通过 **动态检测、智能拆分** 倾斜分区，将大任务拆分为多个小任务，从根本上解决 Join 的长尾问题。核心步骤可归纳为：  
1. **判断可行性**：根据 Join 类型决定是否可拆分左右表；  
2. **检测倾斜**：通过中位数和阈值识别倾斜分区；  
3. **计算目标**：确定拆分后子分区的理想大小；  
4. **拆分执行**：用工具类将大分区拆分为多个均衡的子分区。  

这一机制让 Spark 能自适应处理数据倾斜，大幅提升 Join 性能。理解其原理，也能帮助我们在业务中更高效地优化 Spark 作业！  


**参考资料**：Spark 3.2 源码 `OptimizeSkewedJoin.scala`、`ShufflePartitionsUtil.scala`
