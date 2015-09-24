---
layout: post
title: Abstract Syntax Trees and Optimizations
comments: true
shortinfo: An under the hood look at how queries are represented within the query processing system, and what kind of optimizations are done to make query execution fast.
categories: [Databases]
tags: [SQL, Query Evaluator]
---

### Al-zebra

This is the second of a series of posts about designing a SQL evaluator. The previous post is [here]({% post_url 2015-09-22-designing-a-sql-evaluator-introduction %}).

The traditional model that is used to structure data is the relational model. Various fields, or <i>attributes</i>, are grouped together into tuples. Sets or bags of these tuples form a relation. Fundamental operations that operate on relations are selection, projection, cross product, union and set difference. These operators are complete, meaning that an appropriate combination of these operators can express any relation in the basic relational algebra.

Every SQL query can be rewritten as a relation. For example -

{% highlight SQL %}

SELECT
	R.A, R.B
FROM
	R
WHERE
	R.A > 5;

{% endhighlight %}

can be written as -

<pre>PROJECT<sub>(A, B)</sub> [ SELECT<sub>(A > 5)</sub> [ R ] ]</pre>

The most pleasing aspect of this model is that an operator takes as input, a relation (or two in the case of joins and unions) and produces <i>another</i> relation as output. The algebra is closed on itself.

A series of operators can thus be chained together, each consuming the output of the operator that precedes it, right down to the source. This makes it easy to think about the evaluation. You imagine the data as streams of information. You channel these streams through operators that filter out unneeded data, join it with other streams of information or process some kind of aggregate function like ```sum()``` or ```count()```. The appropriate operators applied in the appropriate order materializes the query's results. 

### Tree, tree again

We can model the chain of operators as a tree. The leaves of the tree are the raw tables. The output of the root operator is the query's result. Since the operator interface is uniform for all relational operators, it is easy to define each operator as a class implementing the operator interface. Thus, our objective is to translate SQL into a tree of Relational Operators, also termed as an ```Abstract Syntax Tree (AST)```.

For example, the previous query can be represented as -

```
			PROJECT
			   |
			SELECT
			   |
			   R
```

### Speak in code

The first step of the process of evaluating SQL off course, is to parse it. SQL is a pretty complex language with lots of niggly little corner cases and complex subqueries and what not. It is a good project to undertake on its own, but is outside the scope of this discussion. We in our graduate databases class used [JSQLParser](https://github.com/JSQLParser/JSqlParser), a Java library that parses SQL for us and returns an object representation of the query that is much easier to work with.

We have the query, we pass it through JSQLParser, we extract all the relevant clauses in the following order -

1. FROM clause, which translates into a hierarchy of cross products,
2. WHERE clause, which if exists, translates into a selection over ```[1]```, or ```[1]``` remains unchanged
3. GROUP BY clause, which if exists, translates into an aggregate operator over ```[2]```, or ```[2]``` remains unchanged
4. HAVING clause, which if exists, translates into another selection over ```[3]```, or ```[3]``` remains unchanged
5. ORDER BY clause, which if exists, translates into an order by operator over the previous results
6. UNION clause, concatenates two plain select queries  

Let's take a look at a more complex query, which is one of the TPC-H benchmark queries, TPCH-6...

{% highlight SQL %}

SELECT
	lineitem.orderkey,
	sum(lineitem.extendedprice*(1-lineitem.discount)) as revenue, 
	orders.orderdate,
	orders.shippriority
FROM
	customer,
	orders,
	lineitem 
WHERE
	customer.mktsegment = 'BUILDING' and customer.custkey = orders.custkey
	and lineitem.orderkey = orders.orderkey 
	and orders.orderdate < DATE('1995-03-15')
	and lineitem.shipdate > DATE('1995-03-15')
GROUP BY 
	lineitem.orderkey, orders.orderdate, orders.shippriority 
ORDER BY 
	revenue desc, orders.orderdate;

{% endhighlight %}

After parsing this query with JSQLParser, we generate our ```AST``` which takes this form -

```
                    PROJECT
                       |
                       |
                    ORDER BY
                       |
                       |
                    GROUP BY   
                       |
                       |
                    SELECT   
                       |
                       |
                  CROSS PRODUCT
                     /	  \
                    /      \
            CROSS PRODUCT  LINEITEM
                /   \
               /     \
          CUSTOMER   ORDERS
```

We define the Operator interface to have three methods - ```initialize()```, ```nextTuple()``` and ```reset()```. This is called the iterator model, or the volcano model. When we have generated the complete AST we can just initilize the root, then call ```nextTuple()``` successively to retrieve all the tuples. The root wiil call the nextTuple method of its child, the child will in turn call its child and so on till it reaches the root. The root is just the table, so it can trivially satisfy the request and returns a tuple to the parent. The parent processes this tuple according to what operator it is. 

For example, the selction operator will check if the tuple satisfies the predicate, and if it does, it will return the tuple to its parent or else, call nextTuple on the child again to get a new tuple. Projection will just strip out the unspecified attributes and return the tuple to the parent. Cross Product operator will take one tuple from each child, and concatenate it. For the next call to nextTuple, it will keep one of these tuples, fetch a new tuple from the other child, and concatenate the two. It will keeep doing so till it has exhausted one of the children, at which point it will fetch a new tuple from the first child, reset the second child and repeat the process.

### Algebra rewrites

If we keep calling ```nextTuple()``` on the root node of the above tree, we will get a correct answer to the query. However, unless the data is trivially small, this will take a long time. For a one gigabyte database, with 4 or 8 GB of RAM available, this strategy will very quickly just run out of memory, not completing at all. This plan is just a starting point. We scan over this tree, look for patterns that can be optimized and replace one set of operators with another equivalent set of operators, but which result in faster execution. 

The TPCH-6 query for example, can be optimized in several ways.

#### Break the SELECT condition

Instead of having just one selection with a conjunction of predicates, we can instead break it up into a series of selection operators each with a simpler predicate. For example, 

<pre>
SELECT <sub>(customer.mktsegment = 'BUILDING' 
and customer.custkey = orders.custkey 
and lineitem.orderkey = orders.orderkey 
and orders.orderdate < DATE('1995-03-15') 
and lineitem.shipdate > DATE('1995-03-15')) [ R ]</sub> 
</pre>

becomes 

<pre>
SELECT<sub>(orders.orderdate < DATE('1995-03-15'))</sub>
   [ SELECT<sub>(lineitem.shipdate > DATE('1995-03-15'))</sub>
      [ SELECT<sub>(lineitem.orderkey = orders.orderkey)</sub> 
         [ SELECT<sub>(customer.custkey = orders.custkey)</sub>
            [ SELECT<sub>(customer.mktsegment = 'BUILDING')</sub> [ R ] ]
         ]
      ]       
   ]
</pre>

Relational algebra offers several algebra rewrite rules and this is a valid rewrite. You can intuitively understand why. The other rewrite rule that we can take advantage of is that we can <i>push down</i> the selections to lower parts of the tree, thus filtering out data earlier. We can push down the predicate ```customer.mktsegment = 'BUILDING'``` to over the CUSTOMER table. This will let us read only valid parts of the customer relation. Depending on how the data is laid out on disk, or whether there's an index on the ```mktsegment``` field, this can reduce the IO bandwidth too. At the very least, it eases the burden on the cross product operator. 

#### Cross products are evil

Cross products are super expensive. They are of order n<sup>2</sup>. Whenever possible, we need to transform selections over cross products into joins. Joins are still pretty expensive, but there are several algorithms for doing joins reasonably efficiently. The join ordering matters too, as mentioned in the first post. Unless there is specific logic built into the code, the join ordering will simply be the order in which the query specifies the tables in the ```FROM``` clause. Ways to estimate which relations are the smallest, and consequently should be joined first form the topic of cost-based optimization. 

#### Push down projections

Similar to pushing down selections, pushing down projections to the leaves reduces the memory footprint of the evaluator.

Our AST at the end of these optimizations looks like this -


```
                   ORDER BY
                       |
                       |
                    GROUP BY
                       |
                       |
                      JOIN
                     /	  \
                    /      \
                JOIN     PROJECT[ SELECT [ LINEITEM ] ]
                /   \
               /     \
           PROJECT   PROJECT
              |        |
              |        |
            SELECT   SELECT
              |        |
              |        |
           CUSTOMER   ORDERS
```

These are just some of the optimization techniques that are available, but these are relatively simple to implement and have huge payoffs. Other optimizations to consider are the efficiency of the join algorithms and the I/O costs, which will be the subject of a subsequent post.