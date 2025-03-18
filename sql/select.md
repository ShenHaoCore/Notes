# SQL查询

## 分页方法

### 方法一

使用`OFFSET`和`FETCH NEXT`实现分页：

```sql
SELECT * FROM table_name AS A WHERE 1 = 1 ORDER BY column DESC OFFSET 0 ROWS FETCH NEXT 20 ROWS ONLY
```

### 方法二

利用`ROW_NUMBER()`函数进行分页：

```sql
SELECT * FROM (
  SELECT ROW_NUMBER() OVER(ORDER BY column DESC) AS RowNo, * 
  FROM table_name AS A WHERE 1 = 1
) AS A WHERE A.RowNo BETWEEN 1 AND 20
```

分组方法
----

使用`ROW_NUMBER()`结合`PARTITION BY`实现分组取第一条数据：

```sql
SELECT * FROM (
  SELECT ROW_NUMBER() OVER(PARTITION BY column ORDER BY column DESC ) AS RowNo,
  FROM table_name AS A WHERE 1 = 1
) AS A WHERE A.RowNo = 1
```

SQL查询的执行顺序
----------

1. `FROM`: 执行笛卡尔积，生成虚拟表VT1。
2. `ON`: 应用ON过滤器，筛选满足条件的行，生成VT2。
3. `JOIN`: 添加外部行，生成VT3（LEFT/RIGHT/FULL OUTER JOIN）。
4. `WHERE`: 对VT3应用WHERE筛选器，生成VT4。
5. `GROUP BY`: 按指定列组合行值，生成VT5。
6. `AGG_FUNC`: 计算聚合函数（如AVG, COUNT等），生成VT6。
7. `WITH CUBE|ROLLUP`: 应用CUBE或ROLLUP选项，生成VT7。
8. `HAVING`: 对VT7应用HAVING筛选器，生成VT8。
9. `SELECT`: 选出指定列，并计算表达式，生成VT9。
10. `DISTINCT`: 去重，生成VT10。
11. `ORDER BY`: 排序结果集，生成游标VC11。
12. `LIMIT/OFFSET`: 返回指定数量的行。

注意：每个步骤都会为下一个步骤生成一个虚拟表作为输入。选择最小的数据源作为基础表可以提高查询效率。


