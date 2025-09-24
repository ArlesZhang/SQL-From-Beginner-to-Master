# SQL 查询与关系模型的核心理解 
> 2025.9.24 学习总结 | Ales & ChatGPT

## 1. **SQL 查询的哲学**
- **一条查询 = 一张表**  
  SQL 基于关系模型（Relational Model），一条 `SELECT` 查询的最终结果是一个**单一的结果集**（二维表）。
- **结果集的结构一致性**  
  表的所有行必须有相同的列结构（列数、列名、数据类型一致），这是关系模型的根本约束。
- **多结果集的处理**  
  如果需要返回多张表：
  - 用多条查询分别执行
  - 或用存储过程（Procedure）返回多个结果集
  - 或在一个查询中用 CTE / 子查询定义多个临时表，但最终只能选择一个结果集输出

---

## 2. **GROUP BY 与 DISTINCT 的区别**
- **GROUP BY**：按指定列组合分组，每组只输出一行，并可在组内执行聚合函数（`COUNT`、`SUM`、`AVG` 等）。
- **DISTINCT**：去掉重复行，不做聚合，只保留唯一行。
- **视觉上的“去重”**：`GROUP BY` 因为每组只输出一行，看起来像去重，但本质是分组聚合。

---

## 3. **条件统计的 CASE WHEN 用法**
```sql
SUM(CASE WHEN score = 100 THEN 1 ELSE 0 END) AS pass_count
```
- 如果 `score = 100` → 返回 1，否则返回 0
- 配合 `SUM` 可以统计满足条件的行数
- 类似 Excel 的 `IF(condition, 1, 0)`

---

## 4. **Hive 中 ORDER BY vs SORT BY**
| 特性 | ORDER BY | SORT BY |
|------|----------|---------|
| 排序范围 | 全局排序（单 reducer） | 每个 reducer 内部排序 |
| 性能 | 慢，单点瓶颈 | 快，可并行 |
| 安全限制 | 无 LIMIT 会报错（strict 模式） | 无限制 |
| 适用场景 | 小数据量全局排序 | 大数据量局部排序 |

---

## 5. **合格率统计的高效写法**
一次扫描表即可得到总量 + 合格数 + 合格率：
```sql
SELECT 
    scene_id,
    COUNT(*) AS total_count,
    SUM(CASE WHEN score = 100 THEN 1 ELSE 0 END) AS pass_count,
    ROUND(SUM(CASE WHEN score = 100 THEN 1 ELSE 0 END) / COUNT(*), 4) AS pass_rate
FROM table
WHERE ...
GROUP BY scene_id
SORT BY scene_id ASC;
```
- 高效：一次扫描完成统计
- 可扩展：在 `GROUP BY` 中加更多维度（部门、坐席等）

---

## 6. **结构不一致的结果集不要强行合并**
- 如果两个结果集的列结构不同，强行 `UNION` 会破坏数据语义，导致逻辑混乱。
- 正确做法：
  - 保持结构一致再合并
  - 或分开返回，分别处理
  - 或增加标识列区分来源

---

## 今日核心收获
1. SQL 查询的世界里，一条查询只能返回一张结构一致的表，这是关系模型的根本约束。
2. GROUP BY 是分组聚合，不是去重；DISTINCT 是去重，不做聚合。
3. 在 Hive 中，ORDER BY 和 SORT BY 的排序范围和性能差异很大，需按场景选择。
4. 高效的合格率统计可以用一次扫描 + CASE WHEN 完成。
5. 不同结构的结果集不要硬合并，否则会破坏数据语义。

---

## Thinking for you
1. 如果你必须在一次查询中返回两个结构完全不同的结果集，你会怎么设计？是用存储过程、还是拆成两条查询？
2. 在你的工作中，哪些场景必须用 **全局排序（ORDER BY）**，而哪些场景可以用 **局部排序（SORT BY）** 来提升性能？
3. 你有没有遇到过因为强行合并不同结构的表而导致分析结果出错的案例？当时是怎么解决的？

---
🎗️Welcome to follow My GitHub
