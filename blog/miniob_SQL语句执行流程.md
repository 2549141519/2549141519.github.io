点击返回[🔗我的博客文章目录](https://2549141519.github.io/#/toc)
* 目录
{:toc}
<div onclick="window.scrollTo({top:0,behavior:'smooth'});" style="background-color:white;position:fixed;bottom:20px;right:40px;padding:10px 10px 5px 10px;cursor:pointer;z-index:10;border-radius:13%;box-shadow:0.5px 3px 7px rgba(0,0,0,0.3);"><img src="https://2549141519.github.io/blogImg/backTop.png" alt="TOP" style="background-color:white;width:30px;"></div>

# 1. SQL语句执行流程
在MiniOB项目中，SQL语句的执行过程如下：   

# 1.1 SQL解析
当SQL语句被输入时，首先经过词法和语法分析器（如 Flex 和 Bison）解析成`SQLNode`。`SQLNode`是抽象语法树（AST）的组成部分，它表示SQL语句中的不同结构元素。  

# 1.2 SQLNode转换为Stmt
在` Stmt `模块中，`SQLNode` 会被简单地判断，并根据不同的 SQL 语句类型（如` SELECT`、`INSERT`、`UPDATE` 等）生成相应的`Stmt`对象。`Stmt` 是逻辑语义的更高级别表示，用于执行逻辑分析和进一步优化。   

# 1.3 生成逻辑算子
`Stmt`对象会被传递到逻辑分析阶段，生成对应的逻辑算子。逻辑算子表示操作的高层次语义（例如表扫描、连接、筛选等），但此时并不关心具体的执行方式。  

# 1.4 生成物理算子
在查询优化阶段，逻辑算子会被优化器转换为物理算子。物理算子代表查询的实际执行策略，例如顺序扫描、索引扫描、哈希连接等。优化器的作用是选择最优的执行方式，以提高查询效率。物理算子与存储模块之间有着直接的关联。物理算子负责执行具体的操作，而这些操作往往涉及对数据库中数据的实际读取、写入、更新和删除，这些操作需要与存储模块交互。  

# 1.5 执行
物理算子会由执行器执行，执行器根据物理算子生成的数据流，访问存储层并返回结果。  

# 2. 查询优化
查询优化 是在逻辑算子生成后和物理算子生成前执行的。这一步优化器根据逻辑算子生成多个执行方案（物理算子），并选择最优的执行方式。  
优化的目标是提高查询效率，可能包括：  
选择适当的索引：根据数据分布和查询条件选择最合适的索引。  
连接顺序优化：对于多表连接，优化器会根据表的大小、索引等选择合适的连接顺序。  
子查询优化：优化器可能将子查询重写为连接或其他形式以提高效率。   
优化完成后，优化器会将逻辑算子转换为物理算子，物理算子负责执行具体的操作（如表扫描、索引查找等）。  

# 3. 中间传递
`SQLStageEvent`类用于在SQL语句的不同阶段传递信息。在MiniOB项目的执行流程中,SQL语句的处理过程可以通过`SQLStageEvent`类将SQL相关的数据结构（如 `ParsedSqlNode`、`Stmt`、`PhysicalOperator`）在不同阶段之间传递。  

# 3.1 示例
1. 解析阶段：`ParsedSqlNode` 被创建并设置为 `sql_node_`。  
2. 语义分析阶段：将 `ParsedSqlNode` 转换为 `Stmt` 对象，并设置到 `stmt_` 中。  
3. 执行计划生成阶段：基于 `Stmt` 生成 `PhysicalOperator`，并设置到 `operator_` 中。  
4. 执行阶段：执行器通过 `PhysicalOperator` 来访问存储层，执行SQL语句。  