# Hive Explain 执行计划解析精要笔记 | By Arles Zhang

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

要不要我帮你把这个笔记配一张 **“Hive 执行计划流程图”**，  
这样你上传到 GitHub 时既有文字精华，又有可视化图，粉丝一眼就能看懂？这样互动效果会更好。
