# Predicate transition
## Introduction
When receiving a query, the database parses it and create an abstract syntax tree(AST), based on which an execution plan is then generated. 

An execution plan has a tree structure, where data is generated at the leaf node and flows to the root. Data received by a non-leaf node usually undergoes some modification before being transferred to another node, including filtering out unwanted rows(Selection), removing or adding extra columns(Projection), matching rows between two children(Join), etc. In this way, data moves from below to above and is returned to the client at root node. The whole process is similar to magma rises to the surface during volcanic eruption.

![image](https://github.com/Charles-1791/database_knowledge/assets/89259555/3148e37e-92f6-4831-bd81-443f4d7c790f)

Usually, the execution plan generated directly from AST is just a verbatim translation of the query -- it gives the right answer but unfortunately fails to taking into consideration the execution cost.

In fact, optimization can be applied to the raw plan to achieve higher efficiency while maintaning the integrity of the result, for instance, 
join reordering, which changes the join order when more than two tables are joined together, or column pruning, which removes unused columns from each tree node, reducing the amount of data transaferred among nodes. But today, i'd like to share my opinion on a commonly used optimization strategy - predicate transtition.

## Outline
In this passage, I would firstly give a brief introduction of tranditional predicate push down, and then switch to predication transition.
After that, some execution plan nodes are introduced, on which we discuss how to perform predicate transition. Finally, some drawbacks and limitations are discussed.

## Predicate Pushdown
Predicate pushdown aims to push predicates, i.e. filters or conditions, towards leaf nodes, so that data is filtered as early as possible. 
The rationale is simple: less amount of data means less time spent on transferring data among nodes; it also reduce memory use. 

Another remarkable advantange is that it precludes unnecessary computation:
Suppose we have table t1 and t2 like these:

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

the generated raw plan is:

![image](https://github.com/Charles-1791/database_knowledge/assets/89259555/d27055c7-876d-4f89-a4d2-75d144695625)

Two Table scans retrieve rows of t1 and t2 from storage and send them to Join; the Join does a Cartesian Product over these rows(a better practice is hash join) and only keeps result satisfying condition b = d and b > 1.0. Afterwards data goes into the Selection node(filter), where tuples not satifsfying 'f like abc%' are removed. What is left is sent to the projection node, where only columns a, b, d, e are preserved.

If we push the predicates 'b > 1.0' and 'f like 'abc%'' to the left and right table scan nodes, the number of rows received by Join is significantly reduced, which decreases join's computing pressure. 

## Predicate Transition
Predication transtition is an advanced version of traditional predicate push down. It derives new predicates from existing ones and pushes them downwards. In addition, the new predicates may displace the original ones if favored by databases(considering indexes). 

Revisit the previous example:

> select a, b, d, e from t1 join t2 on b = d and b > 1.0 where f like 'abc%'

Once we have b = d and b > 1.0, we instantly know d > 1.0, which can be pushed to table scan of t2.
Our deduction is based on equivalent relationship(b = d), and this would be the only type of deduction covered in this passage. 
The more advance kinds of deductions are:

- a > b and b > 0 so a > 0, (greater-than/less-than relationship is transitive)
- max(a) < 10 so a < 10, (a <= max(a) < 10)
- sum(a) < 10 and a > 0 so a < 10, (mathematics),
...

However, not all derived predicates are meaningful. Let's say, we have the following query:

> select a, b, d, e from t1 join t2 on b = d and b > 1.0 where b = a

Seeing b = a, b > 1.0, we immediately know a > 1.0. Nevertheless, column 'a' and 'b' are both from the same table, the new predicate 'a > 1.0' is no better than 'b > 1.0' unless an index has been created on column 'a'. Adding unnecessary predicates like 'a > 1.0' even slows down the query for it introduces redundant conditions. 

Another example is that for query:

> select * from t1 join t2 on b = d and a + b < 100

Although we know 'a + d < 100' is also valid, this filter, once generated, cannot be push to left or right child of Join. So such derivation is also meaningless.

To sum up, predicates transition deduces new predicates from existing ones and pushes them downwards. During the process, however, we should be cautious not to generate trivial predicates; sometimes we replace the existing conditions with derivations for better performance.

## Implementation
Different from the regular practice in papers, where a closure of predicates is generated, my approach is to 'deduce only when you have to' - we do not generate predicates as much as possible(predicate closure) and eliminate redundancy afterwards. Instead, only meaningful and pushable predicates are deduced.

### An overview of plan nodes
A plan node may have two, one or no child nodes; the leaf node read data from storage so it has no child, while others receives data from at least one child. During plan optimization, no real data is transferred, but each node owns meta infos, that is, how many columns it would receive and send in execution phase, together with the names and types of these columns. In our framework, each column is assigned a global unique id to distinguish from others. As a result, a node takes in some uids from its children, appling some changes over these columns, then it returns some uids.  

For example, the query:
> select a, b, d, e from t1 join t2 on b = d and b > 1.0 where f like 'abc%'

yields the plan tree:
![image](https://github.com/Charles-1791/database_knowledge/assets/89259555/bb5ce754-eb0a-47da-94f8-6a6e80c57180)

According to the input and output uids, one-child-noded can be classified into three categories:
- output uids are same as input uids. (Limit, Selection, Sort.)
- output contains possibly extra uids(inputUids is a subset of outputUids). (Aggregation and WindowFunction)
- output only preserves part of uids from input and introduce new or no uids to its output. (Projection)

For node with two children, it is categorized into two classes: 
- output uids are the combinataion of uids of left and right child. (Join)
- output uids are completely different from any uids from its children. (Set operation such as. Union, Minus, Intersect...)

### Two phases
Predicate transition is implemented in two steps -- predicate pullup and push down.

In the pullup phase, every node(except the table scan) receives from its child a 'PredicateSummary', which records the arithmatic relationship among uids, i.e. columns. The 'PredicateSummary' is processed differently according to the nature of the node, and is then returned to its father node, where further modifications are made. Similary to a bubble rising to the water surface, one by one, the 'PredicateSummary' ascends from the bottom(leaves) to the top(root).

![image](https://github.com/Charles-1791/database_knowledge/assets/89259555/82d6e0ff-49a7-4d89-b259-f787f1542552)

The push down phase begins upon the finish of the pull up phase. The summary returned by root now traverses down from root to leaves. While permeating through a node, the summary changes accordingly - some new predicates may be generated while some are attached to the node and stop going any further. When a summary reaches the leaf nodes, which are invariably the Table Scan, it 'flattens' into a group of predicates(filters) for TableScan.

![image](https://github.com/Charles-1791/database_knowledge/assets/89259555/c6f90a55-02ab-445c-9847-fb59ee8c0c73)

For a certain plan node, if we name as 'summary_up' the predicate summary returned by it during pullup phase, and 'summary_down' the summary it receives from its father during pushdown phase, it's always true that summary_up <= summary_down. In other words, the summary_down always contains more 'knowledge' or 'information' than summary_up. Another thing is certain: a summary received by a node must include and only include columns returned by its children.

### Predicate Summary
#### Structure
A predicate is actually an arithmetic relationship among columns, for instance, predicate 'a > 0' add restriction to column a, predicate 'b < c' forces 'c' to be greater than 'b'. Another way to see a predicate is to consider it a 'data feature' or 'pattern', which a tuple must follow to appear in the result set. 

Our predicate summary is the synthesis of all predicates. It consists of two structures -- a set of all equivalent sets and a list of predicates.

```
struct PredicateSummary{
  EqRelations relations
  Expression[] conditions
}
```

#### relations
A 'relations' is an array containing multiple non-overlapping 'equivalent sets', each of which comprises columns equal to each other.
An equivalent set looks like (remember we use an uid to represent a column):

> {#1, #2, #3},

which indicates

> #1 = #2 = #3

Following this pattern, a 'relations' looks like:

> \[ {#1, #2, #3}, {#4}, {#5, #6}, {#7}, {#8, #9} \]

and it suggests:

> #1 = #2 = #3, #5 = #6, #8 = #9

A 'relations' also keeps a lookup table mapping UID to the set to which it belongs.
A typical example looks like:

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

The map specifies that #1, #2, #3 are in the first equivalent set, while #4 and #6 are in the third equivalent set. By examining their set indexes, we can quickly tell whether two uids are equal.

#### conditions
'conditions' store's all predicates other than those owning the form 'column1 = column2', which ought to be recorded in 'relation'.

A predicate in 'conditions' should be interpreted as 'a relation among equivalent sets', rather than 'a relation among columns'.
Considering the following case:
![image](https://github.com/Charles-1791/database_knowledge/assets/89259555/07754464-9f43-4be2-82d0-818edc837023)

The purple block illstrates a 'relation', based on which the predicate

> #1 + #2 - f(#4) * g(#5) = #8 + #9
 
should be construed as a 'template'

> $0 + $0 - f($1) * g($2) = $3 + $4

where the $ are placeholders -- $0 stands for the first equivalent set, $1 stands for the 2nd equivalent set, etc...
In this way, a predicate reveals the arithmetic relation among equivalent sets. By replacing every placeholders with a column from its corresponding set, a valid predicate is generated. In the example given above, in total 3 * 3 * 2 * 2 * 1 * 1 = 36 different predicates can be generated.

### Pullup and Pushdown logic
Since the pullup and pushdown logic strongly correlate, the following content is structured in a manner where each node is followed by its corresponding pullup and pushdown details.

#### Table Scan
##### Pull up
A table scan initially does not have any predicates in a raw plan, so the pull up process is quite simple: create an empty summary and add all output uids into summary.relations and return the summary.

##### Push down
As table scan has no child, all the data feature recorded in summary must be converted back into predicates. 

Similar to stringing some pearls into a necklace, the process of flattening a 'relations' is using equation marks to connect each column, which can be expressed as:
```
for each equivalent set S in relations {
  if |S| == 1 {
    continue
  }
  for i:=1; i < S.size(); ++i {
    generate predicate 'S[i-1] = S[i]'
  }
}
```
The logic behind is quite intuitive, since columns(uids) in an equivalent set equate to each other, we could pick the first element and create an equation connecting it with others. A quick example:

> flatten(\[ {#1}, {#2, #3, #4}, {#5, #6} \]) = \[#2 = #3, #3 = #4, #5 = #6\]

Since 'conditions' itself is an array of predicates which can be used directly, a better practice, nevertheless, would be converting them into new conditions more 'favored' by database through subsituting columns. As metioned above, a predicate reveals relations among equivalent sets; after replacing column with an  indexed(and equivalent) one, a new predicate usually outperforms the original one thanks to the more efficient index scan. The pseudocode is:

```
// we need relations to tell us the equivalent info
func TryToConvert(const expression.Expr& predicate, const EqRelation& relations) (expression.Expr, bool) {
  copied = predciate.copy()
  all_cols = predicate.extract_columns()
  for each col in all_cols {
    if there is index on col {
      continue
    }
    eqSet = relations.GetEquivalentSet(col)
    newCol = pick one indexed column from eqSet
    if newCols is not null {
      copied.replace(col, newCols)
    } 
  }
  return copied
}
```

### Selection
A selection node receives from its child tuples, some rows of which are filered out based on the predicates of the Selection. Since Selection only remove some rows without adding new or removing existing columns, so the output columns remain the same as input ones.
#### Pull up
<img width="640" alt="image" src="https://github.com/Charles-1791/database_knowledge/assets/89259555/27f11e5d-1185-44b6-a522-cc1fc90beae1">

As a 'filter node', a selection node could have one or multiple predicates, each of which could be converted into CNF(conjunctive normal form: https://en.wikipedia.org/wiki/Conjunctive_normal_form). For each clause in a CNF, check whether it owns the form 'col1 = col2', if so, we find a pair of equivalent columns <col1, col2>, which is added to the 'relations'.  field of the PredicateSummary returned from child node; else, we consider the clause a normal predicate and simply move it into 'conditions'.

#### Push down
In push down phase, selection recieved from its father a summary, which contains the predicates the current node used to have. Since selection node must have a child, to which we could directly send the summary.

### Projection
A projection node removes unwanted columns and does computation over one or several columns to generate new columns.
Naming columns returned from projection's child as input columns, and those returned by projection itself as output columns, we can divide these columns into three categories:
- Survivors: columns appear in both input and output
- Victims: columns appear in input but not output
- Newcomers: columns appear in output but not input
<img width="249" alt="image" src="https://github.com/Charles-1791/database_knowledge/assets/89259555/af4a6671-3721-4216-89f5-367eae7abf70">

For example, in the illstration above, we have:
| category | columns |
|----------|---------|
| survivors | #2, #4 |
| victims | #1, #3 |
| newcomers | #5 |

#### Pull up
During pull up phase, the summary returned from child must contain info(equivalent relation and conditions) about input columns. Since an input column may be a victim, any infos involving such column should be modified or erased from the summary. 

<img width="674" alt="image" src="https://github.com/Charles-1791/database_knowledge/assets/89259555/382af549-1923-4251-af21-f42dd6826660">

Let start from relation. For each equivalent set in relation, victims should be pruned from it in that the father node has no idea what such columns are(victims are not return to father node, and in execution phase no data of these columns are reieved). Nevertheless, simply removing of victims leads to an irreversible loss of information -- there's no way to add them back in push down phase. For example, in the last example, assuming in child summary we have an equivalent set {#1, #2}, if we just remove #1, when we do push down, the information #1 = #2 is forever lost. Thus, we need to save the equivalent set in a buffer before filtering out the victims.

Conditions contaning victims should not be returned to father node either, but different from victims in relation, it's possible to convert a predicate to make it 'acceptable' by father node, i.e. in a form only contains survivors. Suppose we have '#1 = #2', from predicate '#1 + #4 < 100', we deduce that '#2 + #4 < 100', which is now valid for parent plan. So the general idea is to substitute victims with its equivalent survivors(if possible), if all victims are replaced, the generated predicate can be reported to father, otherwise, move such predicate into buffer.

#### Push down

<img width="199" alt="image" src="https://github.com/Charles-1791/database_knowledge/assets/89259555/eca2b7b4-854e-45ce-8ff6-2e02388fb994">

##### relation
Again, let's begin with 'relation'. An equivalent set now contains newcomers and survivors, and the newcomers should not appear in the summary sent to child(remember, a summary received by a node must contain exactly its output columns). Removing from the set newcomers again cause information loss, for example, if we have a set {#2, #5}, where #5 is newcomer, erasing #5 from the set causes the loss of data feature '#2 = #5'. 

Fortunately, a newcomer is always an expression comprising exclusively columns from child, i.e. victims + survivors -- the #5 in previous example is defined as #3 + #4. Thus, we could generate a predicate '#2 = #3 + #4' and store it in the Summary.conditions, which would later be sent to child plan. The pseudo-code of such procedure is:
```
colum_from_child = victims + survivors
ret = []
for each equivalent set S in relation {
  if all cols in S in column_from_child
    continue
  else
    S+ = S intersect column_from_child
    S- = S intersect victims
    validCol = S+.first
    for each invalid_col in S-:
      gen = 'invalid_col.corres_expr = validCol'
      ret += gen
}
return ret
```
In short, when handling a relation, some columns must be removed from equivalent sets, meanwhile new predicates are generated and would be pushed down to child. 
At last, we merge the relation after pruning with relation in buffer to draw a comprehensive picture of column equal relationship.
##### condition
A predicate in condition may also contain newcomers, but we could firstly try to replace such columns with their equivalent survivors. For example, if we have:

<img width="224" alt="image" src="https://github.com/Charles-1791/database_knowledge/assets/89259555/2670c5a4-8eb0-4a3e-a497-a8f8ae2db69c">

| category | columns |
|----------|---------|
| survivors | #1, #2 |
| victims | #4 |
| newcomers | #3, #5 |

and 
relation
> \[{#1}, {#2, #3}, {#5}\]

For predicate #1 + #3 < 100, we could generate #1 + #2 < 100, which is child-acceptable.
But for predicate #1 + #5 = 128, since there is no survivor equivalent to #5, this predicate is transformed into '#1 + (#2 * #1 +100) = 128'.

##### summary

<img width="766" alt="image" src="https://github.com/Charles-1791/database_knowledge/assets/89259555/06f72d75-b6a1-4453-b991-a0078552c947">

The whole process can be split into two parts -- relation and condition. For relation, we remove newcomers and generate expressions which is added to condition; for condition, we examine each predicate and displace newcomes with equivalent survivors or their expressions. Finally, we merge the relation and condition sperately with the ones in buffer to draw a full picture.

### Limit
#### Pull Up
Limit keeps a certain number of rows returned from child without changing their values, therefore no further modification on summary is required. A natural practice is to directly return to father the child summary, but due to the reason to be discussed, we keep in buffer an extra copy of summary.

#### Push down
What is fascinating about limit is its semipermeability - a summary from limit's child can be straightly sent to limit's parent whereas a summary from parent cannot go down further. The retionale behind it is simple: doing filter before limit is different from doing limit and then filter. A natural practice is to flatten the summary into an array of predicates, which are then sent to a newly generated Selection node hanging above the Limit. Amongst these predicates, though, some can pass through Limit, namely, the ones included in summary offerred by Limit's child when we do predicate pull up, call this summary 'summary_up'. Let say we have the following two summaries：

<img width="282" alt="image" src="https://github.com/Charles-1791/database_knowledge/assets/89259555/2457e4b7-7f97-4dfa-a063-53cd26291599">

The green block above stands for the summary Limit receives from above in push down phase and the purple block denotes the summary returned by child in pull up phase. 
Flattening the equivalent set {#1, #2, #3} in the green summary, we generate #1 = #2 and #2 = #3. You may have found that #1 = #2 is presented in the purple summary. On top of that, the predicate #1 + #2 < 100 mirrors the one in purple summary. These conditions, since they originate from the nodes below, can be pushed down. 

So far, we know only predictes included in summary_up can be push down. 

<img width="484" alt="image" src="https://github.com/user-attachments/assets/10c844d4-ff14-4cc4-9fda-6924bbfac8e3">

Recall the fact that 'the summary_down always contains more knowledge or information than summary_up', which menas: 

<img width="250" alt="image" src="https://github.com/user-attachments/assets/d6bbc430-4680-4d63-bfe0-a0edd172f764">

Thus, we conclude that the summary which Limit should send to its child is exactly 'summary_up'. And this explains why we need to store in buffer a copy of summary_up.

<img width="342" alt="image" src="https://github.com/user-attachments/assets/0dfa51be-9428-460c-b4e6-4dcef0cbaaed">

To sum up, the whole process is: initially, we flatten into a list of predicates the summary received from father, namely, summary_down; then we remove the pushable from the list, what is left goes into a generated Selection node; eventually, the summary_up in buffer is sent to child. In practice, a better practice is to truncate the summary before flattening, which is discussed in the following two sections.

##### relation
For limit, an equivalent set in fatherSummary.relation can invariably be segregated into several subsets such that each of them is a full equivalent set in childSummary.relation.

<img width="805" alt="image" src="https://github.com/Charles-1791/database_knowledge/assets/89259555/2f53cb4e-59c0-42ff-81b6-58ff8b688df9">

For example, in the above image, the set {#1, #3, #5, #6, #7} is perfectly split into {#1, #3}, {#5}, {#6, #7}. The following case could never possibly happens:

<img width="315" alt="image" src="https://github.com/Charles-1791/database_knowledge/assets/89259555/dd64099a-26bf-48a3-baab-50e8bf4795bc">

The rationale behind is simple: except for Projection, an equivalent set in summary_up.relation is always contained in some equivalent set of predicate_down.relation. 

![image](https://github.com/user-attachments/assets/ddd668cc-59a4-4e5e-8b6e-9217cf4e9982)

In our instance, #4, #5 are in the same set, so they equal to each other; such relation, once built, cannot be broken - $4 and $5 would always stay in the same set and there's no operation tearing them apart. In short, columns in the same set forms an unbreakable group.

Our algorithm is: for each set in summary_down.relation, split it into several, say N, smaller subsets based on summary_up.relation; then pick a single column from each subsets(a regular practice is to choose those with indexes) and generate N-1 equations. Assume the following columns are chosen from subsets:

#c<sub>1</sub>, #c<sub>2</sub>, #c<sub>3</sub>, ..., #c<sub>i</sub>, 

from which we assemble the following predicates:

#c<sub>1</sub> = #c<sub>2</sub>, #c<sub>2</sub> = #c<sub>3</sub>, ..., #c<sub>i-1</sub> = #c<sub>i</sub> 

The proccess is similar to tearing a pizza into several pieces, leaving cheese strands connecting every two pieces. The generated predicates are just like these strands, gluing together 'shards' of an equivalent set.

##### condition
A predicate pd1 is considered redundant when there exists in childSummary.condition a predicate pd2, such that pd1 is equivalent to pd2. Validating equivalence of prediates necessitates consulting a relation(specifing which column equates to which) and since we have two - one in fatherSummary, the other in childSummary - we are in a dilemma: which one to use? Let's gain some intuition from a specific instance.

<img width="206" alt="image" src="https://github.com/Charles-1791/database_knowledge/assets/89259555/db96f306-d63f-477f-96d8-3543d1b14abc">

The first predicate #1 + #3 < 100 in green block looks similar to the one in purple #1 + #2 < 100. According to green relation, #2 and #3 are equivalent, so #1 + #3 < 100 is equivalent to #1 + #2 < 100 and should be removed, whereas based on purple relation, such equivalence does not hold. Which one is corret? The answer is the former one. Let's remove #1 + #3 < 100 and see what happened. 

<img width="305" alt="image" src="https://github.com/Charles-1791/database_knowledge/assets/89259555/42b882ea-af01-4bd0-b5ed-987fb7226f4d">

Following the steps for relation, predicate #2 = #3 would be generated and sent to the generated selection node above. The data received by limit follows rule #1 + #2 < 100, which still holds for selection. So at the node selection, we have both #2 = #3 and #1 + #2 < 100, apparently #1 + #3 < 100 holds. Should a predicate be deduced at this point, why bother to keep it? 

To sum up, we check all predicates stored in summary_down.conditions, if we find an equivalent one in summary_up.conditions based on summary_down.relation, we remove it.

### Aggregation
An aggregation node divides the rows into several groups based on the group-by clause; in each group, the aggregate functions 'combines' or 'merge' the expression(specified in select clause) results for all rows into a single tuple. For instance, for query:

> select a, b, sum(c+d), avg(a-b), max(e) from t group by a + b

Aggregation calculates the value a, b, c+d, a-b, e, a+b for each row, then it splits tuples into groups based on the value of a+b. After that, in one group, the sum, average and maximum is evaluated before a result tuple is outputed. 

Aggregation generates new columns, such as sum(c+d), and could also erase some columns, like column c, d, e. We can reuse the logic of Projection for pull up and pushdown.

#### Join
Join has the most complicated logic of pull up and push down for two child nodes are involved. Let's start with the simplest Inner Join.

##### Inner Join




