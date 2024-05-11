# Predicate transition
## Introduction
Given a query, the database application parses it into an abstract syntax tree(AST), based on which an execution plan is then generated. 

An execution plan is of tree structure; similar to the magma flow during volcanic eruption, data is generated by the leaf node and flows towards the root.
When a non-leaf tree node receives data from its child, it applies some manipulation over it, such as filter out some lines(Selection), 
removing or adding extra columns(Projection), or matching rows between two children(Join). After that, the modified data is returned to its father node.
The whole process is similar to the lava flow during volcanic eruption, data moves from below to above and when the root node returns, the output is send to the client as the query result.
(By the way, stream execution is often used in the process to avoid load all data into main memory.)

![image](https://github.com/Charles-1791/database_knowledge/assets/89259555/3148e37e-92f6-4831-bd81-443f4d7c790f)

Usually, the execution plan derived from AST(raw plan) is just a verbatim translation of the query -- it yeilds the correct output but fails to take into account the execution cost.
In fact, some optimization can be apply to the raw plan to achieve a shorter execution time while maintaning the correctness of the result, for instance, 
join reordering, which changes the join order when more than two tables are joined together, or column pruning, which removes unused columns from the output of each
tree node, in a way reducing the amount of data transaferred among nodes. Here i'd like to share my views on a common but crucial optimization method -- predicate transtition.

## Outline
In this passage, i would firstly give a brief introduction of tranditional predicate push down, and then switch to the concept of predication transition.
After that, some execution plan nodes are introduced, on which we discuss how to perform predicate transition. Finally, some drawbacks and limitations are covered.

## Predicate Pushdown
Predicate pushdown aims to push predicates, i.e. filters or conditions, towards leaf nodes, so that data is filtered as early as possible. 
The rationale is simple: less amount of data means less time spent on transfering data among nodes, and it reduce the pressure of memory. 

Another remarkable advantange is that it preclude unnecessary computation:
Suppose we have table t1 and t2 with the following definition:

***t1***
| a | b | c |
| --- | --- | --- |
| int | double | varchar |

***t2***
| d | e | f |
| --- | --- | --- |
| int | double | varchar |

For query:
> select a, b, d, e from t1 join t2 on b = d and b > 1.0 where f like 'abc%'

the generated RAW plan should be like:

![image](https://github.com/Charles-1791/database_knowledge/assets/89259555/d27055c7-876d-4f89-a4d2-75d144695625)

Two table scan nodes retrieve data of t1 and t2 from disk and return to Join, the Join does a Cartesian Product over rows from its child(in fact, we could do a hash join) and keeps
result meeting the condition b = d and b > 1.0. The join output is then returned to a Selection node(filter), and rows not satifsfying 'f like abc%' is removed. What is left goes to the projection node,
where only columns a, c, d, e are preserved.

If we push the predicates 'b > 1.0' and 'f like 'abc%'' to the left and right table scan nodes, the number of rows received by Join is significantly reduced, which decreases join's computing pressure. 

## Predicate Transition
Predication transtition is an advanced version of traditional predicate push down. Apart from existing predicates, predicate transition derives new predicates and pushes them downwards. In addition, the newly deduced predicates may take the place of the original ones if they are favored by databases(considering indexes). 

Revisit the previous example:

> select a, b, d, e from t1 join t2 on b = d and b > 1.0 where f like 'abc%'

Once we know b = d and b > 1.0, we instantly deduce d > 1.0, which can be pushed to table scan of t2.
Our deduction is based on equivalent relationship(b = d), and this would be the only type of deduction covered in this passage.
The more advance kinds of deductions are:

- a > b and b > 0 so a > 0, (greater-than/less-than relationship is transitive)
- max(a) < 10 so a < 10, (a <= max(a) < 10)
- sum(a) < 10 and a > 0 so a < 10, (mathematics),
...


However, not all derived predicates are meaningful. Assuming the query looks like:

> select a, b, d, e from t1 join t2 on b = d and b > 1.0 where b = a

After seeing b = a, b > 1.0, we know a must also be greater than 1.0. Since column 'a' and 'b' are from the same table, the derived 'a > 1.0' only makes a difference if there is an index on column 'a'.
Such predicates is worthless and slows down the query, in that it introduces needless computation.

In a nut shell, predicates transition deduces new predicates from existing ones, 'expanding' the predicate set, and pushes them downwards. During the process, we should be cautious not to generate trivial predicates; sometimes we replace the existing predicates with newly derivations for better performance.

## Implementation
Different from the approach often adopted in papers, our goal is to 'deduce only when you have to' -- we do not generate predicates as much as possible(predicate closure), pushes all of them and eliminate redundancy; instead, we only deduce nontrivial and pushable predicates.

In the following paragraphs, we would discuss what we should do at each plan node, but before that, let me introduce what a plan node actually is.

### Overviews of plan nodes
A plan node may have two, one or no children node; the leaf node generates data itself, while others receives data from its children. In optimization phase, there's no real data transferred, but each node is clear of the meta infos(column numbers, the name and type of each column) of the data it would receive and return in execution phase. Each column owns a global unique id to distinguish it from others; a tree node takes in some uids from its children, apply some changes over the data or filter out some rows, then it returns some uids.  

After labeling the column, the query:
> select a, b, d, e from t1 join t2 on b = d and b > 1.0 where f like 'abc%'

yields plan tree:
![image](https://github.com/Charles-1791/database_knowledge/assets/89259555/bb5ce754-eb0a-47da-94f8-6a6e80c57180)

According to the input and output uids, one-child-noded can be classified into three catagories:
- output uids are same as input uids. (Limit, Selection, Sort.)
- output contains possibly more uids than input uids(inputUids is a subset of outputUids). (Aggregation and WindowFunction)
- output only preserves some or no uids from input and add more or no uids to its output. (Projection)

node with two children 
- output uids are the combinataion of uids of left and right child. (Join)
- output uids are completely different from any uids from its children. (SetOperation such as. Union, Minus, Intersect...)

### Two phases
Predicate transition can be achieved through two phases of work -- predicate pullup and down.

During the pullup phase, every node(except the table scan) receives from its child a 'PredicateSummary', which records the arithmatic relationship among the columns. The 'PredicateSummary' is processed in distinct manner according to the nature of the node, then returned to its father node, where further modifications are made. Similary to a bubble rising to the water surface, one by one, the 'PredicateSummary' ascends from the bottom(leaves) to the top(root).
![image](https://github.com/Charles-1791/database_knowledge/assets/89259555/82d6e0ff-49a7-4d89-b259-f787f1542552)

The push down phase ensues from end of the pull up phase, and the summary returned from the root is now carried down from root to leaves. When a summary goes through a node, its content changes and some predicates may be generated, which stay there and won't go down. Once a summary reaches the leaf node, i.e. the Table Scan node, it 'flattens' into predicates and are attached to the TableScan.
![image](https://github.com/Charles-1791/database_knowledge/assets/89259555/c6f90a55-02ab-445c-9847-fb59ee8c0c73)


### Predicate Summary
#### Structure
A predicate constrains the arithmetic relationship among columns, for instance, predicate 'a > 0' add restriction to column a, predicate 'b < c' forces 'c' must be greater than 'b' in result. Another way to see predicates is to consider than as 'promises' or 'data feature', that is, a row won't appear in the result set unless it 'conforms to' all the predicates. 

A predicate summary is a synthesis of multiple predicates, and it consists of two fundamental structures -- a set of all equivalent sets and a list of predicates.

```
struct PredicateSummary{
EqRelations relations
Expression[] conditions
}
```

#### relations
A 'relations' is an array containing multiple non-overlapping 'equivalent sets', each comprising several columns that equate to each other.
If we assign each column a unique id like '#1', an equivalent set can be expressed as:

> {#1, #2, #3},

and it indicates 

> #1 = #2 = #3

So a 'relations' can be written as:

> \[ {#1, #2, #3}, {#4}, {#5, #6}, {#7}, {#8, #9} \]

and it demonstrates:

> #1 = #2 = #3, #5 = #6, #8 = #9

A 'relations' also keeps a map mapping each UID to its corresponding set index within the array.
A map looks like:

| UID | set index |
| --- | ---|
| #1 | 0 |
| #2 | 0 |
| #3 | 0 |
| #4 | 1 |
| #5 | 2 |
| #6 | 2 |
| #7 | 3 |
| #8 | 4 |
| #9 | 4 |

#### conditions
'conditions' store's all predicates other than those owning the form 'column1 = column2', which is converted into an equivalent set {column1, column2} and recorded in 'relation'.
A predicate in 'conditions' should be interpreted as 'a relation among equivalent sets', rather than 'a relation among columns'.
Considering the following case:
![image](https://github.com/Charles-1791/database_knowledge/assets/89259555/07754464-9f43-4be2-82d0-818edc837023)

The purple block illstrates a 'relation', based on which the predicate

> #1 + #2 - f(#4) * g(#5) = #8 + #9
 
should be construed as a 'template'

> $0 + $0 - f($1) * g($2) = $3 + $4

where the $ signs are placeholders -- $0 stands for the first equivalent set, $1 stands for the 2nd equivalent set...
In this way, a predicate in fact reveals the arithmetic relations among equivalent sets, and if we replace the placeholders with a column within its corresponding set, a new(and valid) predicate is generated. In the example given above, the template could be used to generate 3 * 3 * 2 * 2 * 1 * 1 = 36 different predicates in total.

### Pullup and Pushdown logic
Since the pullup and pushdown logic strongly correlate, the following content is structured in a pattern where each node is followed by its corresponding pullup and pushdown details.

#### Table Scan


