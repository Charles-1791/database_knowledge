# Predicate transition
## Introduction
Given a query, the database application parses it into a syntax tree, based on which an execution plan is generated. 
A plan is of a tree structure -- a tree node receives data from its child nodes and do some data manipulation over it, such as filter out some lines(Selection), 
remove or addd some columns(Projection), or match rows from two children(Join). Then the processed data is returned and accpected by the father node.
After the root node finish its job, the final output is send to the client as the query result.

Usually, some optimizations are apply to the execution plan tree to achieve faster execution while guaranting the correctness of the query result, for instance, 
join reordering, which changes the join order when more than two tables are joined together, or column pruning, which removes unused columns from the output of each
tree node, in a way reducing the data transafering among nodes. Here i'd like to share my views on a common but crucial optimization method -- predicate transtition.

Predication transtition is an advanced version of traditional predicate push down, a tranditional optimizing strategy pushing filter conditions, i.e. predicates, downtowards into
to the leaf nodes.

## Outline
In this passage, i would firstly give a brief introduction of trandition predicate push down, and then the concept of predication transition.
After that, some selective plan nodes are introduced, on which we discuss how to perform predicate transition. Finally, some drawbacks and limitations are covered.

## Concept of Predicate Transition
Predicate pushdown must be introduced beforehand
