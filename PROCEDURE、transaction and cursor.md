# 📘 SQL 学习日报 - 核心概念精要 | 2025.8.31

> 学习内容：《SQL必知必会》第五版 - 存储过程、事务处理、使用游标

## 1. 🧩 存储过程 (Stored Procedure)

### 核心直觉
**数据库中的可重用脚本**。像定义函数一样封装一组SQL语句，通过名称调用，实现代码复用、简化操作和增强安全。

### 关键点
- **创建**: `CREATE PROCEDURE proc_name() BEGIN ... END`
- **调用**: `CALL proc_name();`
- **优势**: 代码封装、预编译性能好、权限控制（可授予执行过程权限而非底层表权限）。
- **劣势**: 逻辑隐藏于数据库，不利于调试和版本控制。

### 代码示例
```sql
DELIMITER //
CREATE PROCEDURE GetCustomer(IN cityName VARCHAR(255))
BEGIN
    SELECT * FROM customers WHERE city = cityName;
END //
DELIMITER ;

-- 调用
CALL GetCustomer('London');
```

---

## 2. ⚖️ 事务 (Transaction)

### 核心直觉
**保证一系列操作“全部成功”或“全部失败”的机制**。是维护数据一致性和完整性的基石。

### 关键点 (ACID)
- **原子性 (Atomicity)**: 事务内的操作是一个不可分割的整体。
- **一致性 (Consistency)**: 事务使数据库从一个一致状态转变为另一个一致状态。
- **隔离性 (Isolation)**: 并发事务之间互不干扰。
- **持久性 (Durability)**: 事务一旦提交，其结果就是永久性的。

### 代码示例 (自动化流程)
```python
# Python 伪代码示例
try:
    start_transaction()
    update_account_balance('A', -100) # 操作1
    update_account_balance('B', +100) # 操作2
    commit() # 全部成功则提交
except Exception as e:
    rollback() # 任一失败则回滚
    print("Transaction failed:", e)
```

### 保留点 (SAVEPOINT)
允许部分回滚，是事务内的“存档点”。
```sql
START TRANSACTION;
-- 一些操作...
SAVEPOINT before_critical_op;
-- 关键操作...
ROLLBACK TO before_critical_op; -- 仅回滚到保留点
-- 其他操作...
COMMIT;
```

---

## 3. 🧭 游标 (Cursor)

### 核心直觉
**用于在存储过程中逐行遍历查询结果的迭代器**。是对结果集进行逐行处理的工具。

### 关键点
- **使用场景**: 几乎仅限于在存储过程中需要对查询结果进行逐行复杂处理时。
- **性能警告**: **慎用！** 集合操作（一条SQL）远快于游标的逐行操作。游标是最后的手段。
- **使用流程**: `DECLARE` -> `OPEN` -> `FETCH` (循环) -> `CLOSE`。

### 代码示例 (标准四步法)
```sql
DECLARE done BOOLEAN DEFAULT FALSE;
DECLARE cur CURSOR FOR SELECT id, name FROM products;
DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;

OPEN cur;
read_loop: LOOP
    FETCH cur INTO v_id, v_name;
    IF done THEN LEAVE read_loop; END IF;
    -- 对当前行(v_id, v_name)进行处理...
END LOOP;
CLOSE cur;
```

---

## 💎 今日核心总结

| 概念 | 是什么 | 何时使用 | 核心命令 |
| :--- | :--- | :--- | :--- |
| **存储过程** | 预存的SQL脚本集合 | 封装复杂、可重用的数据库操作 | `CREATE PROCEDURE`, `CALL` |
| **事务** | 原子性操作单元 | 保证多个关联操作的一致性（如转账） | `START TRANSACTION`, `COMMIT`, `ROLLBACK` |
| **游标** | 结果集迭代器 | 在存储过程中**必须**对结果集逐行处理时 | `DECLARE CURSOR`, `OPEN`, `FETCH`, `CLOSE` |

---

## 🤔 互动与思考

1.  **权衡与选择**: 存储过程将业务逻辑放入数据库，应用程序只负责调用。你认为这种设计的**优点**（如性能、安全）和**缺点**（如可维护性、调试难度）在微服务架构下是否依然成立？为什么？
2.  **实践反思**: 游标的性能远低于集合操作。你能想到一个具体的业务场景，是**必须使用游标**而无法用一条复杂的SQL语句实现的吗？反之，你是否见过本该用集合操作却误用了游标的例子？
3.  **深入探索**: 事务的隔离级别（如读已提交、可重复读）对并发性能和数据一致性有巨大影响。在你们当前的项目中，数据库默认的隔离级别是什么？是否有遇到过因为隔离级别设置不当而导致的bug或性能问题？

---

Welcome to follow My GitHub
