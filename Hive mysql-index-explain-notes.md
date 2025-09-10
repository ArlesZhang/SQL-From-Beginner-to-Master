# Hive Explain 执行计划 & Index 解析精要笔记 | By Arles Zhang

> 2025.9.10

## 1. 逻辑执行顺序 vs 物理执行顺序
- **逻辑执行顺序**（SQL 标准）：
  ```
  FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
  ```
- **物理执行顺序**（优化器改写后）：
  - 谓词下推（Predicate Pushdown）：将可过滤条件提前到扫描阶段执行
  - 列裁剪（Column Pruning）：只读取需要的列
  - 本地预聚合（Local Aggregation）：先在本地节点聚合，减少 shuffle 数据量
  - 数据重分布（Shuffle / Exchange）：按 key 分区，保证相同 key 在同一节点
  - 全局计算（Global Aggregation / Join / Window）
  - 排序 / Limit

**核心原则**：
> 先减少数据量 → 再做数据重分布 → 最后做全局计算

---

## 2. Hive 中的三种“分区”概念

| 类型 | 层面 | 作用 | 是否减少行数 | 是否减少扫描量 |
|------|------|------|--------------|----------------|
| **存储分区**（表分区） | 存储层 | 按分区列存储数据（如 dt、region） | ❌ | ✅（分区裁剪） |
| **执行分区**（PartitionCols） | 执行层 | 分布式执行时按 key 分发数据（Shuffle） | ❌ | ❌ |
| **SQL 分区**（PARTITION BY） | SQL 逻辑层 | 窗口函数的计算范围 | ❌ | ❌ |

**口诀**：
> 存储分区：减少扫描量  
> 执行分区：数据搬家到同节点  
> SQL 分区：逻辑分组计算

---

## 3. Hive `EXPLAIN` 结构解读

### 3.1 前置信息（舞台说明）
- `Plan optimized by CBO.` → 使用基于代价的优化器
- `Vertex dependency in root stage` → Stage 依赖关系（Map → Reduce）
- `Reducer 2 <- Map 1 (SIMPLE_EDGE)` → Map 输出传给 Reducer（一次 shuffle）
- `Stage-0`、`Stage-1` → 每个 Stage 是一个独立任务

### 3.2 算子树（Operator Tree）
- **File Output Operator (FS_x)** → 最终输出（写文件/返回结果）
- **Select Operator (SEL_x)** → 选择输出列
- **PTF Operator (PTF_x)** → 窗口函数执行
- **Filter Operator (FIL_x)** → 过滤数据
- **TableScan (TS_x)** → 扫描表数据（可能带分区裁剪、列裁剪）

**注意**：
- `EXPLAIN` 打印顺序是**从上往下**（输出 → 输入），  
  **执行顺序是从下往上**（TableScan → Filter → Shuffle → Output）
- “提前到扫描阶段” = 执行时提前，“下推” = 从计划结构上推到更底层

---

## 4. 看 Hive 执行计划的通用方法
1. **找 TableScan** → 看是否有分区裁剪、列裁剪
2. **找 Filter** → 看过滤条件是否提前执行
3. **找 Shuffle / PartitionCols** → 看数据是按什么列分区的
4. **找全局计算** → 聚合 / join / 窗口函数
5. **找排序 / limit** → 是否推迟到最后
```sql
1. TableScan（扫描数据）
   ↓
2. Filter（过滤）
   ↓
3. Projection（列裁剪）
   ↓
4. Local Aggregation / Local Processing（本地预聚合、预处理）
   ↓
5. Shuffle / Exchange（数据重分布）
   ↓
6. Global Aggregation / Join / Window（全局计算）
   ↓
7. Sort / Limit（排序、取前 N）
   ↓
8. Output（返回结果）
```
**口诀**：
> 扫（Scan） → 砍（Filter） → 剪（Projection） → 先算（Local） → 搬（Shuffle） → 总算（Global） → 排（Sort） → 出（Output）

---

## 5. 今日收获
- 理解了 Hive `EXPLAIN` 的**打印顺序 ≠ 执行顺序**
- 搞清楚了“提前”和“下推”是同一件事的不同视角
- 区分了 Hive 中三种“分区”的本质区别
- 掌握了看执行计划的通用锚点和分析步骤

---

## 💡 思考 / 互动问题
1. 你在实际工作中，有没有遇到过 **WHERE 条件写在子查询里** 导致性能变差的情况？优化器是怎么处理的？
2. 如果一个窗口函数的 `PARTITION BY` 列基数非常大（几乎每行一个分区），会对性能造成什么影响？如何优化？
3. 在 Hive 中，**分区裁剪** 和 **列裁剪** 哪个对性能提升更大？为什么？

---

# 📚 MySQL 索引与执行计划精要笔记

## 1. 执行计划中的关键字段
在 `EXPLAIN` 结果中，常用字段含义：
- **type**：访问类型（性能从差到好）
  ```
  ALL < index < range < index_merge < ref_or_null < ref < eq_ref < const < system
  ```
  - `ALL`：全表扫描（最差）
  - `ref`：非唯一索引等值匹配（常见且性能不错）
  - `eq_ref`：唯一索引等值匹配（更好）
  - `const/system`：常量匹配（最好）
- **key**：实际使用的索引名
- **rows**：预估扫描行数（越少越好）

---

## 2. `type=ALL` 与索引优化

### 2.1 问题
- `type=ALL` 表示全表扫描，数据库会一行一行检查，时间复杂度 **O(N)**。
- 原因：查询条件字段没有可用索引。

### 2.2 例子
```sql
-- 表结构
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    age INT,
    city VARCHAR(50)
);

-- 查询
EXPLAIN SELECT * FROM users WHERE city = 'Beijing';
```
结果：
```
type: ALL
key: NULL
rows: 10000000
```
- 全表扫描 1000 万行
- 没有用到索引（key=NULL）

---

## 3. 为条件字段添加索引

### 3.1 创建索引
```sql
CREATE INDEX idx_city ON users(city);
```

### 3.2 再次查询
```sql
EXPLAIN SELECT * FROM users WHERE city = 'Beijing';
```
结果：
```
type: ref
key: idx_city
rows: 5000
```
- **type=ref**：使用了非唯一索引等值匹配
- **key=idx_city**：用到了我们创建的索引
- **rows=5000**：只扫描 5000 行（比 1000 万行快很多）

---

## 4. `type=ref` 的含义
- 使用**非唯一索引**进行等值匹配（`=` 或 `IN`）
- 可能返回多行（因为索引列的值可能重复）
- 性能比 `ALL` 好很多，但不如 `eq_ref`（唯一索引匹配单行）

---

## 5. 为什么索引能优化时间复杂度？

### 5.1 没有索引
- 全表扫描：O(N)
- 类比：找书里一个词，从第一页翻到最后一页

### 5.2 有索引
- 索引通常是 **B+ 树**（或哈希表）
- 查找时间复杂度：O(log N) + O(M)（M=匹配行数）
- 类比：先查目录（索引） → 直接跳到对应页码

---

## 6. 索引优化的逻辑
1. **WHERE 条件字段**经常用来过滤 → 建索引
2. **JOIN 连接字段** → 建索引
3. **ORDER BY / GROUP BY 字段** → 有时可用索引优化
4. **注意**：
   - 索引会占用额外存储空间
   - 索引会降低写入性能（INSERT/UPDATE/DELETE 需要维护索引）
   - 低选择性字段（如性别）加索引意义不大

---

## 7. 优化口诀
> **type=ALL** → 全表扫描 → 看 `key` 是否为 NULL → 如果是 → 考虑给 WHERE 条件字段加索引  
> 索引 = 数据的“目录”，让查找从 O(N) 变成 O(log N)

---

## 💡 思考题
1. 如果一个字段的取值只有 `M` 和 `F`，加索引会有用吗？为什么？
2. 在组合索引 `(col1, col2)` 中，`WHERE col2 = 'x'` 能用到索引吗？
3. 为什么索引会影响写入性能？在什么场景下你会选择不加索引？

---


