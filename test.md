# OpenTSDB平台SQL设计与实现

------

OpenTSDB的RowKey由如下几个部分组成：

|rowkey组成字段|
|--------|
|RowKey  ::= uid hour_timestamp [tagk, tagv] {tagk, tagv}

其中uid为3Bytes整型，hour_timestamp是4Bytes整型，为UNIX Epoch时间的小时部分值（即等于timesec-timesec%3600），tagk和tagv均为3Bytes整型。其中uid、tagk、tagv的值都从元数据表中映射得到的。
实际上我们使用的OpentTSDB和开源版本会有少许差异，主要区别在RowKey的设计，我们的OpenTSDB的RowKey是固定长度的，有一个元数据表标明RowKey的组成成分及顺序，表名为RowKeyMeta，各字段如下：

|列名|	数据类型|说明
|--------|-----:|:----:|
|id	|INT	|自增主键，无意义
|name	|VARCHAR|	字段名称
|kindex	|INT	|字段在rowkey的序号
|length	|INT	|字段在rowkey占用的字节数
|tname	|VARCHAR	|字段对应的hbase表名，同属于一个表名下的所有字段组成一个rowkey
|mtable	|VARCHAR	|字段的映射表名，如果该值为null表示不需要映射
|info	|VARCHAR	|该字段的说明

查询该表的语句如下：
```sql
select * from RowkeyMeta where tname = 'hbase table name' order by index asc
```
以电表采集数据存储在tsdb表为例，在MySQL的RowKeyMeta表中，RowKey字段组成如下：

|名称|类型 |序号 |长度 | HBase表名  |映射表名 |说明 |
| --------   | -----:  | :----:  | :----:  | :----:  | :----:  | :----:  |
| eid	| INT	| 1	| 3	| tsdb	| eid	| 电表ID
|address|INT	|2	|2	|tsdb	|address	|地址(乡站)归属
|type	|INT	|3	|1	|tsdb	|NULL	|（行业、业务）分类
|voltage	|INT	|4	|1	|tsdb	|voltage	|电压等级
|datetime	|INT	|5	|4 |NULL|NULL|Unix epoch 小时时间


**注意：当前支持的映射关系都是字符串映射到整型，即映射表为两列[id: INT, name: String]，如果映射表名为NULL，那么该类型整型值直接嵌入到rowkey中**

其中datetime默认放置在最后，和OpenTSDB一样为UNIX Epoch时间的小时部分，以上每一个字段都有一个元数据表对应（元数据表名列），如地址归属项目，元数据表address中有"长沙市麓谷高新区芯城科技园"对应address值为1。
	此外，查询出来的数据字段处理RowKey外，还包括timestamp（用RowKey中hour值加上CQ中的偏移得到）、value，也就是说实际查询出来的数据解开后格式为：
	
|结果组成字段|
|--------|
|Result ::= eid address type voltage timestamp value


------

## 需求描述

常规分析查询通过Spark实现，而一些即席查询Spark无法涵盖到，所以这里实现主要处理查询结果不大、查询频次少、查询语句不确定的查询，因当前查询需求不明确，所有暂时考虑支持的查询语句如下：
>* 仅支持DML操作且只支持select，不支持insert/update/delete等
>* 所有语句SQL关键字均要求为小写
>* 不支持子查询、JOIN查询、IN查询、BETWEEN查询、EXIST查询、NOT查询、LIKE查询
>* 不支持函数和存储过程
>* 仅支持count、sum、min、max、avg五个函数
>* 有限支持group by、order by（DESC&ASC支持？）
>* 支持Projection中的别名
>* where从句中必须有时间字段（名称$datetime）且最好是：$datetime >from and $datetime < to 这种方式（如果to不存在取当前时间戳？）
>* 要求所有的条件写右边，即column > 10而不是10 < column
>* 为保证内存使用在较低范围，limit会自动带上，默认为1000（TODO）

|查询示范语句|
|-------|
|select eid, sum(value) as TOTAL from tsdb where eid = 10 and (address = "长沙" or type != "农网") and datetime > "2017/10/26 09:10:59" group by address order by eid limit 100|

## 设计

### 1. 流程

```flow
st=>start: Start
op=>operation: SQL解析
parser_cond=>condition: 语法是否正确?
field_cond=>condition: 要求的字段是否存在?
meta_op=>operation: 查询元数据表组装rowkey
hbase_scan=>operation: 根据RowKey查询HBase
compute_op=>operation: 对查询结果进行计算
e=>end

st->op->parser_cond->field_cond->meta_op->hbase_scan->compute_op
parser_cond(yes)->field_cond
parser_cond(no)->e
field_cond(yes)->meta_op
field_cond(no)->e
```


## 实现

### 1. AST解析

SQL解析的BNF语法如下：
```bnf
selectStmt ::= "select" projectStmt fromStmt [whereStmt] [groupbyStmt] [orderbyStmt] [limitStmt] [";"]

projectStmt ::= projection {"," projection}
projection ::= "*" | primarySelect ["as" ident]
primarySelect ::= knowFunction | selectLiteral | selectIdent
knowFunction ::= "count" "(" "*"| ["distinct"] singeSelect ")" | "min" "(" singeSelect ")" |
               "max" "(" singeSelect ")" | "sum" "(" ["distinct"] singeSelect ")" |
               "avg" "("  ["distinct"] singeSelect ")"
singeSelect ::= literal | fieldIdent

fromStmt ::= "from" (ident ["as" indent] | { ident ["as" indent]})
whereStmt ::= "where" primaryWhere { [ ("and" | "or") primaryWhere] }
primaryWhere ::= (literal | fieldIdent) ("=" | "<>" | "!=" | "<" | "<=" | ">" | ">=") (literal | fieldIdent)
groupbyStmt ::= "group" "by" fieldIdent {, fieldIdent }
orderbyStmt ::= "order" "by" (fieldIdent ["asc" | "desc"]) {, (fieldIdent ["asc" | "desc"])}
limitStmt ::= "limit" numericLiteral [, numericLiteral]

literal::= numericLiteral | stringLiteral | floatLiteral | "null"
fieldIdent::= ident ["." ident]

```

语法树定义（用scala描述，仅列出部分及示范，不代表实际代码）：
```scala
trait Node
trait SqlExpr extends Node
trait BinOp extends SqlExpr {
  val lhs: SqlExpr
  val rhs: SqlExpr
}
case class Literal(value: MetaData[_]) extends SqlExpr with SqlProj
case class FieldIdent(qualify: Option[String], name: String) extends SqlExpr with SqlProj
case class Or(lhs: SqlExpr, rhs: SqlExpr) extends BinOp
case class And(lhs: SqlExpr, rhs: SqlExpr) extends BinOp

trait EqualityLike extends BinOp
case class Eq(lhs: SqlExpr, rhs: SqlExpr) extends EqualityLike
case class Neq(lhs: SqlExpr, rhs: SqlExpr) extends EqualityLike
case class Ls(lhs: SqlExpr, rhs: SqlExpr) extends EqualityLike
case class Gt(lhs: SqlExpr, rhs: SqlExpr) extends EqualityLike
case class LsEq(lhs: SqlExpr, rhs: SqlExpr) extends EqualityLike
case class GtEq(lhs: SqlExpr, rhs: SqlExpr) extends EqualityLike
```
**注意：解析为AST后，一般还要做有效性判断，如判断where从句的列是否存在，类型是否正确等，但在该版本中暂时不实现，后续增加这部分内容不会影响整个流程**


SQL解析当前存在如下几种实现方案：

 1. FLEX & YACC
 2. ANTLR
 3. scala-parser-combinators

本项目只需要实现简单的SQL处理，flex&yacc以及antlr学习复杂度高，需要较长时间去学习和问题定位，scala-parser相对简单，使用case class可以很轻松的实现这些SQL的解析，此外，scala也可以调用java类库，所以本项目选择scala-parser。

### 2. HBase查询

第一步得到Where从句信息后，操作如下：

 1. 是否存在datetime，否抛出SqlNotExistDateException
 2. datetime的操作是否为EQ或GT或GE（大于、等于或大于等于操作），否抛出ScanTooMuchDataException
 3. 查询RowKeyMeta表获取相关元数据，根据元数据指向的映射表查出相关数据（在数据不大情况下缓存到本地）
 4. 根据RowKey拼接所需要的字段获取的映射表数据集大小，预判大约要读取的HBase行数（等于每一个字段的映射结果集大小相乘之后再乘以datetime小时数），如行数大于约定值（如1000）抛出异常ScanTooMuchDataException
 5. 如果rowkey所需字段在where从句中全部存在，取每一个映射结果集的最大最小值组装startKey和endKey，再调用Table.getScanner
 6. 查询得到的数据为List(rowkey, datetime,value)，然后将rowkey按RowKeyMeta中的项目进行分解，结果类型为List[Row]，其中Row为Map
 7. 对查询结果进行计算，即实现where从句的计算部分





