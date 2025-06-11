# 索引

## MySQL 中都有哪些索引？

### 按物理存储分类 (主要针对 InnoDB 存储引擎)

* **聚簇索引 （Clustered Index）**：
    * InnoDB 存储引擎特有。
    * 将数据行和索引存储在一起，**数据是根据主键的顺序物理存储的**。
    * 每个 InnoDB 表只有一个聚簇索引，通常就是**主键**。
    * 查询效率高，因为找到索引就找到了数据。
* **二级索引 （Secondary Index / Non-Clustered Index）**：
    * 也称为辅助索引或非聚簇索引。
    * 存储索引列的值和**主键的值**。
    * 通过二级索引查询时，需要先获取主键值，**再通过主键值去聚簇索引中查找完整数据**（这个过程称为“回表”）。

### 按字段特性分类

* **主键索引 （Primary Key Index）**：
    * 一种特殊的**唯一索引**，一个表只能有一个。
    * 主键列的值必须**唯一且非空**。
    * 在 InnoDB 中，主键索引就是**聚簇索引**。
* **唯一索引 （Unique Index）**：
    * 保证索引列（或多列组合）的值是唯一的。
    * 允许包含 `NULL` 值（多个 `NULL` 值也是允许的）。
    * 一个表可以有多个唯一索引。
* **普通索引 （Normal Index / Simple Index）**：
    * 最基本的索引类型，对值没有唯一性或非空限制。
    * 可以包含重复值和 `NULL` 值，主要用于加速查询。
* **前缀索引 （Prefix Index）**：
    * 对字符串类型列，只索引列的**前 N 个字符**。
    * 减少索引大小，提高查询效率，但可能降低选择性。

### 按字段个数分类

* **单列索引 （Single-Column Index）**：
    * 一个索引只包含**一个列**。
    * 一个表可以有多个单列索引。
* **联合索引 （Composite Index / Compound Index）**：
    * 也称为复合索引或组合索引。
    * 一个索引包含**多个列**。
    * 遵循“**最左前缀原则**”：只有查询条件中包含了联合索引的**最左边的列**，索引才能被有效利用。

---

## 二级索引存放的有哪些数据？

<style>
    /* 引入 Inter 字体，如果你的 Docsify 环境已经全局引入则可以移除此行 */
    @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap');
   
    .index-container {
        display: flex;
        flex-direction: row; /* 改回水平排列 */
        align-items: center;
        gap: 30px; /* 调整水平间距 */
        margin-bottom: 30px;
    }

    .index-box {
        background-color: #ffffff;
        border: 2px solid #3498db; /* Blue border */
        border-radius: 12px; /* More rounded corners */
        box-shadow: 0 8px 16px rgba(0, 0, 0, 0.1); /* Softer shadow */
        padding: 30px;
        width: 380px; /* Slightly wider */
        min-height: 250px; /* Consistent height */
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
        position: relative;
    }

    .index-box.clustered {
        border-color: #27ae60; /* Green for clustered index */
    }

    .index-box h10 {
        color: #3498db;
        font-size: 1.8em;
        margin-top: 0;
        margin-bottom: 20px;
    }

    .index-box.clustered h10 {
        color: #27ae60;
    }

    .index-content {
        text-align: left;
        width: 100%;
    }

    .index-content p {
        margin-bottom: 10px;
        line-height: 1.6;
    }

    .index-content ul {
        list-style: none; /* Remove default list style */
        padding: 0;
        margin: 15px 0 0 0;
    }

    .index-content ul li {
        background-color: #ecf0f1; /* Light grey background for items */
        padding: 10px 15px;
        margin-bottom: 8px;
        border-radius: 6px;
        display: flex;
        align-items: center;
        box-shadow: inset 0 1px 3px rgba(0,0,0,0.05); /* Inner shadow */
    }

    .index-content ul li strong {
        color: #2c3e50; /* Dark text for emphasis */
        margin-right: 8px;
    }

    .index-content ul li.highlight {
        background-color: #ffeaa7; /* Yellow highlight for primary key */
        border: 1px solid #fdcb6e;
    }

    /* Arrows and Labels for horizontal layout */
    .path {
        display: flex;
        flex-direction: column; /* 保持垂直，因为箭头和标签是垂直关系的 */
        align-items: center;
        position: relative;
    }

    .path .arrow-line {
        width: 100px; /* 水平箭头的宽度 */
        height: 3px; /* 水平箭头的高度 */
        background-color: #e67e22; /* Orange for path */
        border-radius: 5px;
        position: relative;
    }

    .path .arrow-line::after {
        content: '';
        position: absolute;
        right: -10px; /* 箭头指向右方 */
        top: -6px; /* 垂直居中 */
        border: solid transparent;
        border-width: 10px 0 10px 15px; /* 箭头尺寸 */
        border-left-color: #e67e22;
    }

    .path .label {
        position: absolute;
        top: -30px; /* 标签在箭头上方 */
        font-size: 1.0em;
        font-weight: bold;
        color: #e67e22;
        white-space: nowrap;
        text-align: center;
    }
</style>

**MySQL 索引关系示意图 (InnoDB)**

<div class="index-container">
    <div class="index-box">
        <h10>二级索引 (Secondary Index)</h10>
        <div class="index-content">
            <p><strong>叶子节点存储内容：</strong></p>
            <ul>
                <li><strong>索引列值：：</strong> <small>(例如 `name`, `email` 等)</small></li>
                <li class="highlight"><strong>对应行的主键值：：</strong> <small>(例如 `id`)</small></li>
            </ul>
        </div>
    </div>
    <div class="path">
        <div class="arrow-line"></div>
        <span class="label">通过主键值</span>
    </div>
    <div class="index-box clustered">
        <h10>聚簇索引 (Clustered Index)</h10>
        <div class="index-content">
            <p><strong>叶子节点存储内容：：</strong></p>
            <ul>
                <li class="highlight"><strong>主键值：：</strong> <small>(例如 `id`)</small></li>
                <li class="highlight"><strong>完整的行数据：： </strong><small>(所有列的值，例如 `id`, `name`, `email`, `address` 等)</small></li>
            </ul>
        </div>
    </div>
</div>


在MySQL的InnoDB存储引擎中，二级索引（也称为辅助索引或非聚集索引）的叶子节点主要存放以下数据：

- **索引列的值 （Key Value）**：这是二级索引本身所基于的列的值。例如，如果你在 name 列上创建了一个二级索引，那么叶子节点就会存储 name 列的值。

- **主键的值 （Primary Key Value）**：除了索引列的值，二级索引的叶子节点还会存储对应行的主键值。

**为什么会存储主键值而不是行指针？**

InnoDB存储引擎的一个重要特性是它的**聚簇索引（Clustered Index）**。聚簇索引是表的主键，它的叶子节点存储了行的所有完整数据。

由于二级索引的叶子节点不存储完整的行数据，当通过二级索引查询到某个索引列的值后，如果需要获取该行的其他列数据，就需要通过叶子节点中存储的主键值，再回到**聚簇索引**中去查找对应的完整行数据。这个过程被称为**回表（Look-up/Back-to-table）**。

这种设计有以下几个优点：

- **减少维护成本**：如果二级索引存储的是物理行指针，那么当数据行移动或发生页分裂时，所有的二级索引都需要更新这些指针，维护成本很高。而存储主键值，即使行数据在物理存储上发生变化，只要主键值不变，二级索引就不需要更新。
- **保证数据一致性**：所有的数据最终都通过聚簇索引进行访问，确保了数据的一致性。

**总结来说**：

- **聚簇索引（主键索引）**的叶子节点存储的是完整的行数据。
- **二级索引（辅助索引）**的叶子节点存储的是索引列的值 + 对应行的主键值。
因此，当你使用二级索引查询数据时，通常需要两次B+树查找：一次在二级索引B+树中查找主键值，另一次在聚簇索引B+树中通过主键值查找完整的行数据。

## 索引失效的情况
在MySQL中，索引的创建是为了提高查询效率，但并非所有情况下索引都会被使用。以下是一些常见的MySQL索引失效的情况：

1. **不满足最左前缀原则 (Leftmost Prefix Rule)**
* **复合索引 (联合索引)** 的使用需要遵循最左前缀原则。如果查询条件不包含复合索引的最左侧列，或者跳过了中间的列，那么索引可能部分失效甚至完全失效。
* **示例：** `CREATE INDEX idx_name_age_gender ON users(name, age, gender);`
    * `SELECT * FROM users WHERE age = 25;` (失效，因为 `name` 不在查询条件中)
    * `SELECT * FROM users WHERE name = 'Alice' AND gender = 'Female';` (部分失效，`gender` 无法使用索引，因为跳过了 `age`)
    * `SELECT * FROM users WHERE name = 'Alice' AND age = 25 AND gender = 'Female';` (完全使用)
<br/>

2. **在索引列上进行计算、函数操作或类型转换**
* 如果对索引列进行数学运算、字符串函数（如 `LEFT()`, `SUBSTRING()`, `CONCAT()`）、日期函数（如 `DATE_FORMAT()`）或者隐式/显式类型转换，MySQL优化器将无法直接使用索引树进行查找。
* **示例：** `SELECT * FROM users WHERE age + 10 = 35;` (失效，对 `age` 进行了计算)
* **示例：** `SELECT * FROM users WHERE DATE_FORMAT(create_time, '%Y-%m-%d') = '2023-01-01';` (失效，对 `create_time` 使用了函数)
* **示例：** `SELECT * FROM users WHERE phone_number = 123456789;` (如果 `phone_number` 是字符串类型，而你用数字进行比较，可能导致隐式类型转换，索引失效)
<br/>

3. **使用 `OR` 连接条件**
* 当 `OR` 连接的条件中，其中一个条件列没有索引，或者每个条件都使用了索引但优化器认为全表扫描更高效时，索引可能会失效。如果 `OR` 的每个条件都使用了索引，并且查询优化器认为联合使用这些索引比全表扫描更高效，则可能会使用索引合并。
* **示例：** `SELECT * FROM users WHERE name = 'Alice' OR address = 'Beijing';` (如果 `address` 没有索引，则 `name` 索引也可能失效)
<br/>

4. **使用 `LIKE` 进行模糊查询，且模式以 `%` 开头**
* 当 `LIKE` 查询的模式以 `%` 开头时（例如 `LIKE '%keyword%'` 或 `LIKE '%keyword'`），索引无法被有效利用，因为无法确定起始点。
* **示例：** `SELECT * FROM products WHERE product_name LIKE '%phone%';` (失效)
* **示例：** `SELECT * FROM products WHERE product_name LIKE 'phone%';` (有效，可以利用索引)
<br/>

5. **使用 `NOT`、`!=`、`<>` (不等于)**
* 这些操作符通常会导致索引失效，因为它们会匹配索引中大量的非匹配值，优化器可能认为全表扫描更划算。
* **示例：** `SELECT * FROM users WHERE status != 'active';` (可能失效)
* **例外：** 在某些情况下，如果非匹配的比例非常小，优化器仍然可能选择使用索引。
<br/>

6. **使用 `IS NULL` 或 `IS NOT NULL`**
* 如果索引列允许存储 `NULL` 值，并且索引类型是B-tree索引，那么 `IS NULL` 和 `IS NOT NULL` 操作可能会导致索引失效。因为 `NULL` 值在B-tree索引中存储的特性，使得这类查询难以有效地利用索引。
* **示例：** `SELECT * FROM users WHERE email IS NULL;` (可能失效)
* **注意：** 对于某些存储引擎（如InnoDB），索引中可以存储 `NULL` 值，但这并不意味着查询 `IS NULL` 就能高效使用索引。通常情况下，如果一个列经常需要查询 `IS NULL` 或 `IS NOT NULL`，可以考虑在索引中包含一个额外的不为NULL的列，或者使用位图索引（MySQL不直接支持）。
<br/>


7. **数据分布不均匀 (低选择性)**
* 当索引列的区分度很低，即相同值的记录太多（例如，性别字段只有 '男' 和 '女'，各占50%），优化器可能会认为使用索引进行查找的成本甚至高于全表扫描，从而放弃使用索引。
* **示例：** `SELECT * FROM employees WHERE gender = 'Male';` (如果性别列数据非常均衡，索引可能不被使用)
<br/>


8. **`ORDER BY` 或 `GROUP BY` 不使用索引或跳过索引列**
* 如果 `ORDER BY` 或 `GROUP BY` 的列不在索引的最左前缀中，或者中间跳过了索引列，则无法利用索引进行排序或分组，可能导致文件排序 (filesort)。
* **示例：** `CREATE INDEX idx_city_age ON users(city, age);`
    * `SELECT * FROM users ORDER BY age;` (失效，因为 `city` 不在 `ORDER BY` 中)
    * `SELECT * FROM users WHERE city = 'Shanghai' ORDER BY age;` (有效，可以使用索引进行排序)
<br/>


9. **隐式类型转换导致的索引失效**
* 当查询条件中的数据类型与索引列的数据类型不匹配时，MySQL可能会进行隐式类型转换。如果转换发生在索引列上，就会导致索引失效。
* **示例：** `SELECT * FROM users WHERE id = '123';` (如果 `id` 是 `INT` 类型，`'123'` 是字符串，MySQL会尝试将 `id` 转换为字符串进行比较，导致索引失效)
<br/>

10. **优化器选择全表扫描**
* 即使存在可用索引，MySQL的查询优化器也会根据统计信息（如表大小、索引的选择性、缓存命中率等）评估使用索引和全表扫描的成本。在某些情况下，例如表数据量很小，或者要查询的数据占了表的大部分比例，优化器可能会认为全表扫描的成本更低，从而不使用索引。
