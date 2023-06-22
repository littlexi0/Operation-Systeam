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








概述
在本次实验中，一个连接对应一个后端进程。该进程的工作就是接受 client 发来的 SQL statement
并返回查询结果给 client 。
SQL 进入后端进程之后，
通过分析器生成语法解析树 parse tree 和查询树 query tree ，这里将进行语法检查
然后进入 rewriter ，按照某些规则进行树的改写，例如将含视图的查询重写为对表的查询
优化器生成计划树 plan tree ，这里的程序负责计算指令开销，并将最优秀的那种执行方法导出
到 plan tree
最后，执行器 executor 按照规划树指定的方式进行表（和索引）的访问，执行查询（包括计算约
束条件），返回一行行结果给 client
值得一提的是，规划器检查不同的、可能的连接方式，找出开销最小的。这里，连接就包括嵌套循环连
接 Block Nested Loop Join 。
具体分析与例子
以 levenshtein_distance ，即
的 无表查询 为例。多表查询的路径优化甚至会涉及遗传算法，本次限于篇幅就略过了。
PostgresMain
首先调用的是 PostgresMain ，在完成时区等初始化之后进入 CommandRead 状态。 ReadCommand 函
数会将以字符串形式储存的 query 进行解析，返回 'Q' 'P' 'B' 等标识符。就本例而言，返
回 'Q' 。
随后经过 switch case 筛选， query 将被进行长度合法性判断，更改编码方式
( pg_client_to_server )，最后将数据传入 exec_simple_query 函数进行执行。
exec_simple_query
select levenshtein_distance('apply', 'apple');
pg_parse_query
首先进入 pg_parse_query 进行查询语句的处理，该函数返回 parsetree_list (语法分析树的初步形
态)。该函数内部其主要作用的又是 raw_parser 函数，这里再通过 scanner 使用 gram.y 中定义的
POSTGRESQL BISON rules/actions 进行语法解析。
gram.y 的路径是 src/backend/parser/gram.y，需要注意。
具体的树在 src/include/nodes/parsenodes.h 中定义。树的根节点有不同的数据结构，select语句
对应 SelectStmt ，update语句对应 UpdateStmt ，以此类推。
观察 SelectStmt 结构体，部分代码其实相当直白
除此之外还有指向左右节点的指针等成员，限于篇幅不再详述。
另外需要注意到的是，pg_parse_query() 返回的是一个语法解析树的列表，之后代码将进入一个 for 循
环，把列表里的每一棵树拿出来进行分析、重写和执行。
pg_analyze_and_rewrite
该函数接受一个 parsetree ，返回一个 Qurey 结构体的列表。
Qurey 结构体本身也是查询树根节点的数据结构，这有助于理解parse_analyze() 返回类型为什么
是 Query* 。
查询树的根节点包含了：指令类型 commandType (select、insert、update、delete、utility)、
hasSubLinks 是否包含子查询、有无聚合函数 hasAggs 、返回值列表头指针 targetList 等等
诸多信息。
根据我的分析， rtable 指针指向的就是该查询中用到的表的列表； jointree 指针指向的是
fromlist 和已经表达式树化的 where clause 的信息。
// scanner, src/backend/parser/parser.c 52行
yyresult = base_yyparse(yyscanner);
List *distinctClause; /* (SELECT DISTINCT) 表达式列表或 NULL */
IntoClause *intoClause; /* SELECT INTO / CREATE TABLE AS 的目标表*/
List *targetList; /* 指向查询结果列表的指针 */
// 下面是指向查询子句的指针
List *fromClause; /* FROM */
Node *whereClause; /* WHERE */
List *groupClause; /* GROUP BY */
Node *havingClause; /* HAVING */
List *windowClause; /* WINDOW */
WithClause *withClause; /* WITH */
parse_analyze() 中，根据 SQL 语句的不同，会调用 transform$Stmt 函数，这里 $ 指代select和update
等语句标识。该类函数返回值也是 Query* 类型，经简单处理就将作为 parse_analyze() 的返回值，表示
一棵查询树已经构建完毕。
pg_rewrite_query() 接受一个查询树，返回一个查询树列表。
这个操作暗含了 rewrite 的功能之一，即将视图转写成基于基本表的查询。这样，一个 Query
便会又在生成其他 Query ，而不得不使用列表了。
pg_plan_queries
该函数接受重写后的查询树列表，返回 plantree_list （也是列表）。在函数内部，
querytree_list 经过循环，其中的元素被pg_plan_query() 逐个处理。该函数的返回值数据结构
PlannedStmt* 就是指向 plantree 根节点的指针。这是指向结构体 PlannedStmt 的指针，其中存储
着执行器所需的大量信息。
处理过程大体如下
一些指针的设置
预处理（ preprocess_expression 以及 preprocess_qual_conditions 、
reduce_outer_joins 函数和将 having 转为 where 的代码块等）。这一阶段相当明显，从
planner.c 的422行开始 preprocess_expression() 大量被调用，针对 targetList offset 等进行
提前计算和优化，将外连接转为自然连接等等。
正式处理 main planning 。创建一条到表的访问路径时，将顺序扫描、索引扫描、位图扫描的代
价计算出来。对应扫描方式的访问路径和代价都会被保存，最后比较大小。最小的那个结果将新建
一个 RelOptInfo 结构体保存。此外， order by 子句的耗时也会被计算。最后，利用存下来的最
小代价的路径，生成 planntree 。
以上是基于 standard_planner 函数（其被planner()调用，是没有指定planner时的默认选项）中调用
的 subquery_planner 函数内部结构的分析。
由于例子并不涉及表的访问，因而在预处理阶段对 targetList 进行优化的时候即访问例子中的函数，
完成值的计算。具体的调用路径如下：
// 分析和重写的代码框架
// pg_analyze_and_rewrite()
// (1) Perform parse analysis.
if (log_parser_stats)
ResetUsage();
query = parse_analyze(parsetree, query_string, paramTypes, numParams);
if (log_parser_stats)
ShowUsage("PARSE ANALYSIS STATISTICS");
// (2) Rewrite the queries, as necessary
querytree_list = pg_rewrite_query(query);
preprocess_expression() => eval_const_expressions() =>
eval_const_expressions_mutator() =>
// 进入迭代，单个处理。我们可以明显地看到函数名已经不带复数了
expression_tree_mutator() => simply_function() => evaluate_function() =>
evaluate_expr() =>
ExecEvalExprSwitchContext() => ExecEvalFunc() => ExecMakeFuncResult() =>
// 下面是一个宏
FunctionCallInvoke(fcinfo) => 结束，已经到达levenshtein_distance()
预处理执行结束之后，函数的返回值已经被计算出来了，他将被放入 expression_tree ，运送到正式
处理阶段。
至于代价的估计，则是 subquery_planner 调用的 query_planner 函数进行的。从调用
make_one_rel() 起，一个正式的 RelOptInfo 结构体就被创建起来存储代价和访问路径数据，其中被
query_planner() 中着重关注的元素如下：
设置具体开销常数的函数的一些例子是：
这些函数有利于人工帮助 postgresql 更加正确地估计查询开销，我个人认为还是比较重要的。
多表查询将使用动态规划或遗传算法（表数目过多时）进行查询代价的优化。
执行
主要的函数是 PortalDefineQuery PortalStart PortalRun 。在启动 portal 之后， executor
从 plantree 中自底向上地处理节点（调用对应的处理函数）。
处理函数的一个例子是执行索引扫描的函数 ExecIndexScan ，其输入参数只有一个 IndexScanState
* ，指向一个节点。在该函数内部，ExecScan()被调用来完成扫描工作。

