---
{"dg-publish":true,"permalink":"/Code/7.Database/Database.1a.索引/","title":"索引","noteIcon":""}
---


# 索引

> [08 索引：排序的艺术.md (lianglianglee.com)](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%98%E5%AE%9D%E5%85%B8/08%20%20%E7%B4%A2%E5%BC%95%EF%BC%9A%E6%8E%92%E5%BA%8F%E7%9A%84%E8%89%BA%E6%9C%AF.md)
> [索引常见面试题 | 小林coding (xiaolincoding.com)](https://xiaolincoding.com/mysql/index/index_interview.html)

**索引是提升查询速度的一种数据结构**

缺点：
- 需要占用空间
- 创建和维护索引产生时间和性能开销

适用场景：
- 字段有唯一性限制
- 字段经常用于WHERE查询
- 字段经常用于GROUP BY及ORDER BY

不建议使用情况：
- 表数据少
- 索引字段存在大量重复数据
- 索引字段经常更新

## 索引分类

- 按「数据结构」分类：**B+tree索引**、**Hash索引**、**Full-text索引**
- 按「物理存储」分类：**聚簇索引(主键索引)**、**二级索引(辅助索引)**
- 按「字段特性」分类：**主键索引**、**唯一索引**、**普通索引**、**前缀索引**
- 按「字段个数」分类：**单列索引**、**联合索引**

InnoDB采用B+树索引结构

### 数据结构

#### B+树索引

> [为什么 MySQL 采用 B+ 树作为索引？ | 小林coding ](https://xiaolincoding.com/mysql/index/why_index_chose_bpuls_tree.html#%E4%BB%80%E4%B9%88%E6%98%AF-b-%E6%A0%91)

> ![7c635d682bd3cdc421bb9eea33a5a413.png (1080×509) (xiaolincoding.com)](https://cdn.xiaolincoding.com//mysql/other/7c635d682bd3cdc421bb9eea33a5a413.png)

特点：基于磁盘的多叉平衡树，通常只有3~4层，磁盘I/O次数少，**查询效率高**

B+树是一种多叉树，叶子节点存放实际数据(索引+记录)，非叶子节点只存放索引
- 每个节点里的数据**按主键顺序存放**
- 每一层父节点的索引值都会出现在下层子节点的索引值中，并且是子节点中所有索引的最大或最小
- 每个叶子节点都包含有前后叶子节点的指针，构成双向链表，有助于范围查找
- B+ 树在删除根节点的时候，由于存在冗余的节点，所以不会发生复杂的树的变形

所有B+树都从高度为1的树开始慢慢增加高度，Mysql中B+树非叶子节点的大小受页限制

### 物理存储

>![3104c8c3adf36e8931862fe8a0520f5d.png (1080×509) (xiaolincoding.com)](https://cdn.xiaolincoding.com//mysql/other/3104c8c3adf36e8931862fe8a0520f5d.png)

二级索引的B+树中存放二级索引和主键的键值对，在二级索引B+树中找到主键然后回到主键B+树中查找值的方式叫做**回表**
**覆盖索引**指仅通过二级索引的B+树就能够找到数据

### 字段类型

**主键索引**：建立在主键字段上的索引，一张表最多一个，不允许包含空值
**唯一索引**：建立在`UNIQUE`字段上的索引，一张表可以有多个，允许包含空值
**普通索引**：建立在普通字段上的索引
**前缀索引**：对字符类型字段前n个字符建立的索引，可建立在类型为char、varchar、binary、varbinary的字段上

```sql
CREATE TABLE table_name (    
  id BIGINT AUTO_INCREMENT,
  card_id CHAR(18) NOT NULL,
  name VARCHAR(128) NOT NULL,
  sex CHAR(6) NOT NULL,
  registerDate DATETIME NOT NULL,
  ...
  PRIMARY KEY(id) USING BTREE -- 主键索引
  UNIQUE KEY(card_id,name)    -- 唯一索引
  INDEX(sex, registerDate)    -- 普通索引
  INDEX(name(5))              -- 前缀索引
);
```

### 字段个数

**单列索引**建立在单个字段上
**联合索引**建立在多个字段上

```sql
CREATE INDEX index_a_b_c ON product(a, b, c); -- 联合索引
```

联合索引的非叶子节点用多个字段的值作为 B+树的 key 值
使用联合索引时存在**最左匹配原则**，如果查询时未使用最左端字段则无法匹配上联合索引，原理为联合索引的B+树中a是全局有序的，b、c是全局无序，局部相对有序的
最左匹配原则会一直向右匹配，遇到范围查询则停止

V5.6 引入的**索引下推优化**， 可以在联合索引遍历过程中，对联合索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数

实际开发工作中建立联合索引时，**把区分度大的字段排在前面**，这样区分度大的字段越有可能被更多的 SQL 使用到

**联合索引范围查询**：
- `select * from t_table where a > 1 and b = 2`
  - 无法使用联合索引
- `select * from t_table where a >= 1 and b = 2`
  - 对于`a=1, b=2`使用了联合索引
- `SELECT * FROM t_table WHERE a BETWEEN 2 AND 8 AND b = 2`
  - 在Mysql中BETWEEN包含边界值，对于`a=2, b=2`使用联合索引

## 索引优化

### 前缀索引优化

使用前缀索引以减小索引字段大小，进而增加单个索引页中存储的索引值，提高索引查询速度

局限性：
- ORDER BY无法使用前缀索引
- 无法把前缀索引用作覆盖索引

### 覆盖索引优化

可通过建立联合索引的方式进行覆盖索引，避免回表

## 索引选择

优化器计算每条SQL语句的成本，基于成本选择索引，优先使用成本最低的索引

一条SQL的成本 = CPU开销 + IO开销

## 索引失效

- 当使用左模糊匹配`like %xx`或左右模糊匹配`like %xx%`时
- 在WHERE中对索引进行了计算，函数，类型转换等操作
- 在WHERE中OR 前的条件列是索引列，而在 OR 后的条件列不是索引列
- 联合索引中未遵循最左匹配原则

## 索引冲突

如果在插入数据的时候发生了主键索引冲突，插入新记录的事务会给已存在的主键值重复的聚簇索引记录添加*S 型记录锁*
如果在插入数据的时候发生了唯一二级索引冲突，插入新记录的事务会给已存在的二级索引记录添加*S 型临键锁*
