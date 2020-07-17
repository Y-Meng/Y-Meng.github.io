# Query DSL

Elasticsearch提供了一个基于JSON的完整查询DSL（领域特定语言）来定义查询。将查询DSL看作是查询的AST（抽象语法树），由两种类型的子句组成：

## 叶查询子句（叶查询子句）

叶查询子句在特定字段中查找特定值，例如match、term或range查询。这些查询可以自己使用。

## 复合查询子句（Compound query clauses）

复合查询子句包装其他叶查询或复合查询，用于以逻辑方式组合多个查询（例如bool或dis_max查询），或更改它们的行为（例如constanst_score查询）。

查询子句的行为不同，这取决于它们是在query上下文中使用还是在filter上下文中使用。