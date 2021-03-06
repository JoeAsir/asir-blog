---
title: Catalyst Optimization in Spark SQL 
date: 2018-09-25 19:52:15
tags: spark
categories: spark
toc: true
---
![](https://github.com/JoeAsir/blog-image/raw/master/blog/background/app-applications-apps.jpg)
Spark SQL is one of the most important components of Apache Spark and has become the fundation of Structure Streaming, ML Pipeline, GraphFrames and so on since Spark 2.0. Also, Spark SQL provides SQL queries and DataFrame/Dataset API, both of which are optimized by Catalyst. Let's talk about Catalyst today.
<!--more-->
Catalyst Optimizer is a great optimization framework to improve Spark SQL performance, especially query capabilities of Spark SQL and it is written by functional programming construct in Scala. RBO(Rule Based Optimization) and CBO(Cost Based Optimization) are the optimization principles of Catalyst. The concept of Catalyst is familiar to many other query optimizers, so understanding Catalyst may help you learn the working pipeline of other query optimizers.
## Trees And Rules
We will have a quick review of trees and rules. You can learn more about them by the references. 
### Trees
The tree is the basic datatype in Catalyst, which is consisted of node objects. Each node has a node type, and some children, which could be none. Let's have an example, the tree for expression x+(1+2) could be translated in Scala as:

![](https://github.com/JoeAsir/blog-image/raw/master/blog/18/18-3.png)
```scala
Add(Attribute(x),Add(Literal(1),Literal(2)))
```
Actually, most of the SQL query optimization, like Catalyst, would transform the SQL to a huge tree structure as the first step of the optimization. Tree structure could be modified, tailored and optimized easily, also it represents the query plan briefly. What's more, the data can be thrown to every node of the tree by the query plan iteratively. That's why tree datatype is used and introduced firstly in Catalyst.

### Rules
Trees can be manipulated by the rules. The optimization can be treated as some transformations from one tree to another under some rules we provide. With the help of rules, we can run arbitrary code and many other operations on the input tree. And the Catalyst will know which part of the tree that rules could match or apply, and will automatically skip over the tree that does not match. The rules are all applied by Scala case class, in other words, one rule can match multiple patterns. Let's see an example.
```Scala
tree.transform {
  case Add(Literal(c1),Literal(c2)) => Literal(c1+c2)
  case Add(left, Literal(0)) => left
  case Add(Literal(0), right) => right
}
```

## Catalyst
From this section, we will know how Catalyst optimizes Spark SQL, which might improve your understanding of how Spark SQL works and how to manipulate it better. There are four plan through the Catalyst, which are Parsed(Unresolved) Logical Plan, Analyzed Logical Plan, Optimized Logical Plan and Physical Plan. You can find all the four plans in your Spark UI. The figure below presents the workflow of Spark SQL, and the Analysis, Logical Optimization and Physical Planning are all working under the Catalyst, and we will go over one by one.

![](https://github.com/JoeAsir/blog-image/raw/master/blog/18/18-2.png)
### Parser
The first stage in the figure above is called Parser. Despite not a part of Catalyst, Parser is also very crucial in the whole optimization. Spark SQL transform SQL queries to an [AST](http://ns.inria.fr/ast/sql/index.html)(Abstract Syntax Tree), also called Parsed Logical Plan in Catalyst,  by [ANTLR](https://www.antlr.org/), which is also used in the Hive, presto and so on. However, DataFrame/Dataset object can be transformed to Parsed Logical Plan by the API.
### Analysis
Returned by the Parser, Parsed Logical Plan contains many unresolved attribute reference of relation. For example, you have no idea whether the column name provided is correct or type of the column, either. Spark SQL use Catalyst and Catalog object that tracks the data all the time to resolve the attributes. Looking up relations by name from Catalog, mapping all the named attributes to the input, checking the attributes which match to the same value and giving the unique ID and pushing the type information of the attributes, the Parsed Logical Plan is transformed to Analyzed Logical Plan by Catalyst.
> Catalogs are named collections of schemas in an SQL-environment. An SQL-environment contains zero or more catalogs. A catalog contains one or more schemas, but always contains a schema named INFORMATION\_SCHEMA that contains the views and domains of the Information Schema.
### Logcial Optimization
Logical Optimization is mainly based on the rules, which is also called RBO(Rules Based Optimization). Lots of rules are provided in this step to accomplish RBO, including constant folding, predicate pushdown, project pruning and so on. As results, the Optimized Logical Plan is returned by Logical Optimization from Analyzed Logical Plan. Some figures below describe these ROB mentioned.
**Predicate pushdown** can reduce the computation of join operation by filtering unnecessary data before join.

![](https://github.com/JoeAsir/blog-image/raw/master/blog/18/18-4.png)
**Constant folding** avoids calculating the same operation between constants for each record.

![](https://github.com/JoeAsir/blog-image/raw/master/blog/18/18-5.png)
**Column Pruning** makes Spark SQL only load data which would be used in the table.

![](https://github.com/JoeAsir/blog-image/raw/master/blog/18/18-6.png)
### Physical Planning
Since we get the Optimized Logical Plan, Spark still doesn't know how to execute the it. For instance, Spark knows that there is a join operation, while whether sortMerge Join or Hash Shuffle Join should be invoked. So all the operation would be mapped into a real exection through [SparkStrategy](https://github.com/apache/spark/blob/v2.3.2/sql/core/src/main/scala/org/apache/spark/sql/execution/SparkStrategies.scala). Transformed from Opktimized Logical Plan by Physical Planning, Physical Plan would guide the Spark on how to handle the data. In Physical Planning, several Physical Plans are generated from Logical Plan, using physical operator matches the Spark execution engine. Using CBO(Cost Based Optimization), Catalyst will select the best performance one as the Physical Plan. However, the CBO is now only used for join and aggregation algorithms selection. Also, CBO is used in Physical Planning to pipelining projections or filters into single Spark *map()* transformation. What's more, all

### Code Generation
Getting the Physical Plan from the Physical Planning stage, Spark will translate the AST into Java bytecode to run on each machine. Thanks to quasiquotes, a greate feature of Scala, ASTs would be built by Scala language based on the Physical Plan. And the AST will be send to Scala Compile at runtime to generate 
ava bytecode. 
> We use Catalyst to transform a tree representing an expression in SQL to an AST for Scala code to evaluate that expression, and then compile and run the generated code.

In Spark 2.x, the WholeStageCodeGen has been used in code generation stage, you can read more from [here](https://joeasir.github.io/2018/11/14/spark-second-generation-tungsten-in-spark).

At last, if you want to see the Logical Plan and Physical Plan of your own Spark SQL, you may do as follows:
```Scala
// for Logical Plan
spark.sql("your SQL").queryExecution
// for Physical Plan
spark.sql("your SQL").explain
```

## References
* [Deep Dive into Spark SQL’s Catalyst Optimizer](https://databricks.com/blog/2015/04/13/deep-dive-into-spark-sqls-catalyst-optimizer.html)
* [Spark SQL Optimization – Understanding the Catalyst Optimizer](https://data-flair.training/blogs/spark-sql-optimization-catalyst-optimizer/)
* [Catalyst Source Code](https://github.com/apache/spark/tree/master/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst)
* [Quasiquotes Introduction](https://docs.scala-lang.org/overviews/quasiquotes/intro.html)
