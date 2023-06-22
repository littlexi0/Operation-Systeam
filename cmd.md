cd ~/postgresql-9.1.3 

make -j

make install -j

$HOME/pgsql/bin/initdb -D $HOME/pgsql/data --locale=C

$HOME/pgsql/bin/postgres -D $HOME/pgsql/data 

$HOME/pgsql/bin/psql postgres -c 'CREATE DATABASE similarity;'

$HOME/pgsql/bin/psql -d similarity -f ./similarity_data.sql

$HOME/pgsql/bin/psql similarity -c "SELECT ra.address, ap.address, ra.name, ap.phone FROM restaurantaddress ra, addressphone ap WHERE levenshtein_distance(ra.address, ap.address) < 4 AND (ap.address LIKE '%Berkeley%' OR ap.address LIKE '%Oakland%') ORDER BY 1, 2, 3, 4;" > ../levenshtein.txt

$HOME/pgsql/bin/psql similarity -c "SELECT rp.phone, ap.phone, rp.name, ap.address FROM restaurantphone rp, addressphone ap WHERE jaccard_index(rp.phone, ap.phone) > .6 AND (ap.address LIKE '%Berkeley%' OR ap.address LIKE '%Oakland%') ORDER BY 1, 2, 3, 4; " > ../jaccard.txt

$HOME/pgsql/bin/psql similarity -c "SELECT ra.name, rp.name, ra.address, ap.address, rp.phone, ap.phone FROM restaurantphone rp, restaurantaddress ra, addressphone ap WHERE jaccard_index(rp.phone, ap.phone) >= .55 AND levenshtein_distance(rp.name, ra.name) <= 5 AND jaccard_index(ra.address, ap.address) >= .6 AND (ap.address LIKE '%Berkeley%' OR ap.address LIKE '%Oakland%')ORDER BY 1, 2, 3, 4, 5, 6;" > ../combined.txt

$HOME/pgsql/bin/pg_ctl -D $HOME/pgsql/data stop
1条查询语句通过`postgres.c`的`exec_simple_query()`进行，可以分为4个部分。



1. 将用户输入的SQL查询语句转化为原始语法树`raw_parsetree_list`。通过调用函数`pg_parse_query()`

```c
parsetree_list = pg_parse_query(query_string);
```

返回原始语法树列表。

2. 语义分析与查询重写，生成`querytree_list`。

```c
List *
pg_analyze_and_rewrite(Node *parsetree, const char *query_string,
					   Oid *paramTypes, int numParams)

stmt_list = pg_analyze_and_rewrite(parsetree,
										   sql,
										   NULL,
										   0);
```

将一个`select `语句拆分成多个部分，将parse tree转换成query tree

3. 生成并优化查询计划`plan_query`。

```c
List *
pg_plan_queries(List *querytrees, int cursorOptions, ParamListInfo boundParams)
    
stmt_list = pg_plan_queries(stmt_list, 0, NULL);
```

根据query tree产生查询计划。其中，调用了优化函数，

```c
/* call the optimizer */
	plan = planner(querytree, cursorOptions, boundParams);
```

根据表和索引的统计信息去计算不同路径的可能代价值，最后选出最优者。

4. 执行查询。

```c
if (IsA(stmt, PlannedStmt) &&
	((PlannedStmt *) stmt)->utilityStmt == NULL)
{
    QueryDesc  *qdesc;

    qdesc = CreateQueryDesc((PlannedStmt *) stmt,
                                            sql,
                                            GetActiveSnapshot(), NULL,
                                            dest, NULL, 0);

    ExecutorStart(qdesc, 0);
    ExecutorRun(qdesc, ForwardScanDirection, 0);
    ExecutorFinish(qdesc);
    ExecutorEnd(qdesc);

    FreeQueryDesc(qdesc);
}
else
{
    ProcessUtility(stmt,
                    sql,
                    NULL,
                    false,	/* not top level */
                    dest,
                    NULL);
}
```

### 2.2 源码分析

1. `src/backend/utils/fmgr/funcapi.c`

添加目标函数的实现。

2. `src/include/catalog/pg_proc.h `

目标函数的注册。

3. `src/backend/executor/execMain.c `

实现2.1.4中的查询执行过程中的函数

```c
ExecutorStart(qdesc, 0);
ExecutorRun(qdesc, ForwardScanDirection, 0);
ExecutorFinish(qdesc);
ExecutorEnd(qdesc);
```

4. `src/backend/executor/execScan.c`

对关系的元组进行扫描，返回正确元组。

5. `src/backend/executor/execProcnode.c `

 对节点类型进行初始化、获取元组、清理。

6. `src/backend/executor/execTuples.c`

对元组槽进行处理，用于与元组相关的资源管理，是否临时元组的内存。

