官网：https://abloz.com/hbase/book.html#architecture
<img width="1557" height="843" alt="image" src="https://github.com/user-attachments/assets/06d2ecfa-1fff-49cd-832a-4684c87c7abd" />
HBase 是一个基于 HDFS 的分布式数据库，数据的存储和管理通过多个组件来实现。以下是 HBase 中的主要组件及其关系的详细解释：

1. HLog (HBase Write-Ahead Log)
定义：HLog 是 HBase 的 Write-Ahead Log (WAL)，用于记录对数据的所有修改操作（写入、更新、删除），以确保数据的持久性和可靠性。
作用：当数据被写入 HBase 时，首先会被记录到 HLog 中，这样即使系统崩溃，HBase 也可以通过 HLog 恢复未成功持久化到 HDFS 的数据。
位置：HLog 文件通常保存在 HDFS 上的特定目录中，与 RegionServer 相关联。
2. HRegion
定义：HRegion 是 HBase 中数据存储的基本单元，每个表会被水平分割成多个 Region，每个 Region 存储一定范围的 Row Key 数据。
作用：HRegion 是数据操作的主要单元，负责处理读写请求。每个 HRegion 会管理多个 Store，每个 Store 对应一个列族的数据。
关系：一个表可以包含多个 HRegion，每个 HRegion 包含多个 Store。
3. Store
定义：Store 是 HRegion 中的一个组件，每个 Store 对应一个列族（Column Family）。
作用：Store 负责管理特定列族的数据，存储在 MemStore 中的最新数据，以及持久化在 HDFS 上的 StoreFile 中的数据。
关系：一个 HRegion 包含多个 Store，每个 Store 对应表的一个列族。
4. MemStore
定义：MemStore 是 HBase 中的数据写缓存，位于内存中。
作用：当数据写入 HBase 时，首先写入到 HLog，然后写入到 MemStore。MemStore 中的数据定期被刷新到磁盘，形成新的 StoreFile。
关系：每个 Store 都有一个 MemStore，负责暂存数据，直到数据被刷写到磁盘。
5. StoreFile
定义：StoreFile 是 HBase 中存储在 HDFS 上的物理文件，包含了持久化的数据。
作用：StoreFile 是由 MemStore 刷新到磁盘时生成的 HFile，存储实际的数据内容。多个 StoreFile 可以通过 Minor 或 Major Compaction 合并成一个新的 StoreFile。
关系：每个 Store 包含多个 StoreFile，存储着同一列族的数据。
6. HFile
定义：HFile 是 HBase 中用来存储实际数据的文件格式，底层使用 HDFS 进行存储。
作用：HFile 是存储在 StoreFile 中的物理文件，包含了经过编码和压缩的数据，支持快速随机读取和写入。
关系：每个 StoreFile 对应一个或多个 HFile，HFile 是存储在 HDFS 上的实际文件。
组件关系总结
数据写入流程：
数据写入 HBase 时，首先被写入 HLog (WAL)，然后写入 MemStore（位于内存中）。
当 MemStore 达到一定大小或达到时间阈值时，数据会被刷写到 HDFS 上，形成新的 StoreFile。
StoreFile 内部的数据以 HFile 格式存储。


数据读取流程：
读取请求首先在 MemStore 中查找，如果找到则返回最新数据。
如果 MemStore 中没有，则查找 StoreFile（对应 HFile）中的数据。


Compaction 操作：
Minor Compaction 会将多个小的 StoreFile 合并成一个较大的 StoreFile，删除无效的旧版本数据。
Major Compaction 会合并所有的 StoreFile，并且彻底清理删除标记和过期数据。


示例
假设你有一个表 my_table，它包含两个列族 cf1 和 cf2。
当你向 my_table 中插入数据时：
数据首先被写入到 HLog。
接着，数据会写入 cf1 和 cf2 各自的 MemStore。
当 MemStore 被刷写时，cf1 和 cf2 各自生成新的 StoreFile，每个 StoreFile 中的数据以 HFile 格式存储在 HDFS 上。
