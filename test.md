
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
