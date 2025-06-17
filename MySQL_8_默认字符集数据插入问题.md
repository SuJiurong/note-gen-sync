# EasyExcel插入数据库唯一索引冲突问题排查报告

## 问题描述

在通过 EasyExcel 批量导入数据至数据库表 `unit` 时，`unit_name` 字段有唯一索引（`idx_unit_name`）。插入 `㎡` 后，再插入 `m2` 时，数据库提示唯一索引冲突：

```
Duplicate entry '㎡' for key 'unit.idx_unit_name'
```

实际业务需求中，`㎡`（平方米符号）和 `m2`（字母+数字）应允许同时存在。

---

## 排查过程

### 1. 数据表结构分析

```sql
CREATE TABLE unit (
  ...
  unit_name varchar(50) NOT NULL COMMENT '单位名称',
  ...
  UNIQUE KEY idx_unit_name (unit_name),
  ...
) ENGINE=InnoDB AUTO_INCREMENT=247 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='计量单位表'
```

**关键点**：

- `unit_name` 列唯一索引
- 表默认字符集：`utf8mb4`
- 排序规则（Collation）：`utf8mb4_0900_ai_ci`

---

### 2. 数据内容检查

查询实际存储的 HEX 值：

```sql
SELECT unit_name, HEX(unit_name) FROM unit WHERE unit_name IN ('㎡', 'm2');
```

**结果：**

- `m2` 的 HEX 值为 `6D32`
- `㎡` 的 HEX 值为 `E38EA1`

表明两者在存储层面完全不同。

---

### 3. 唯一索引冲突复现

手动插入数据：

```sql
INSERT INTO unit (unit_name, unit_en_name, ...) VALUES ('㎡', 'sqm', ...); -- 成功
INSERT INTO unit (unit_name, unit_en_name, ...) VALUES ('m2', 'sqm', ...); -- 报错
```

**报错信息：**

```
Duplicate entry '㎡' for key 'unit.idx_unit_name'
```

---

### 4. 问题根因分析

#### 4.1 排序规则（Collation）影响

- `utf8mb4_0900_ai_ci` 是 MySQL 8.0 默认排序规则
- `ci` = case-insensitive（不区分大小写）
- 此排序规则对部分**形近字符/单位符号**进行了“归一化”处理
  - **例如：`㎡`、`m2`、`m²` 在此 Collation 下视为等价字符**

**导致现象：**
即使 HEX 不同，只要 Collation 判定等价，唯一索引也视为重复。

---

## 解决方案

### 修改唯一索引字段的 Collation 为严格区分字符的二进制方式

**执行如下 SQL：**

```sql
ALTER TABLE unit MODIFY unit_name VARCHAR(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL COMMENT '单位名称';
```

- `utf8mb4_bin` 以字节为单位区分字符
- 只要存储内容不同，唯一索引不会冲突

### 验证结果

更改 Collation 后，再次插入 `㎡` 和 `m2` 均可成功，唯一索引冲突问题消失。

---

## 结论与建议

- 本次问题根因在于 MySQL 8.0 新排序规则 `utf8mb4_0900_ai_ci` 对部分字符做了等价归一化处理，影响了唯一性判断。
- 对于**业务需要严格区分字符**的唯一索引字段，建议 Collation 使用 `utf8mb4_bin`。
- 不建议全表都用 `bin`，只需用于相关唯一性字段即可。

---

## 参考资料

- [MySQL 8.0 排序规则差异](https://dev.mysql.com/doc/refman/8.0/en/charset-unicode-sets.html#charset-unicode-sets-collations)
- [MySQL升级8.0的新故障，utf8mb4_0900_ai_ci是啥？](https://www.lifesailor.me/archives/2676.html)
