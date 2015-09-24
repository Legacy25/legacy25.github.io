---
layout: post
title: Designing a SQL Evaluator - Introduction
comments: true
categories: [Databases]
tags: [SQL, Query Evaluator]
---

### Challenges to evaluating SQL

Efficiently evaluating SQL is not trivial. In fact, its not even straightforward. Naive execution strategies, given large enough data, will starve your program of memory. Optimizing I/O becomes significantly more important. The same program fragments will be executed for millions of rows, and any bottleneck in this pipeline, however harmless on its own, has the potential to take a significant toll in the larger scheme of things.

### The declarative dogma

SQL, or Structured Query Language, is a <i>declarative</i> language. This means that the analyst using SQL to run queries on structured data specifies <i>what</i> data is required, rather than <i>how</i> to get that data. This not only makes the job of the analyst much easier, but also gives the evaluator a lot of leeway in deciding how best to go about answering that query.

For example, let's say we have a database of information about cars and manufacturers that participate in a racing championship. There are five tables, with schemas as listed below -

```
Cars [ CID | Model | MID | Year | Power | Torque | Price ]

Manufacturers [ MID | Manufacturer | Headquarters ]

Revenues [ MID | Year| Revenue ]

Race_Events [ RID | Track | Year ]

Standings [ RID | CID | Finishing_Position ]
```
Bob wants to see if there is any relation between how well the company does in terms of revenue to how well it performs in racing the next year. He issues the following SQL query -

{% highlight sql %}

SELECT 
	Manufacturer, count(*) as Wins, Revenue
FROM
	Cars, Manufacturers, Revenues, Race_Events, Standings
WHERE
	Cars.MID = Manufacturers.MID
	and Manufacturers.MID = Revenues.MID
	and Race_Events.RID = Standings.RID
	and Cars.CID = Standings.CID
	and Revenues.Year = Race_Events.Year + 1
	and Standings.Finishing_Position = 1
GROUP BY
	Manufacturer
ORDER BY 
	Revenue desc;

{% endhighlight %}

This is quite a complex query, with several ways to evaluate it. Do we compute the join between Cars and Manufacturers first? Or the join between Manufacturers and Revenues? Does it make a difference?

Turns out, it does.

### Optmize optimally

The order of joins is the largest contributing force in selecting a good execution strategy. You ideally want to join the smallest table with the second smallest table, then the result with the third smallest table, and so on till you get to the largest table. This is because of the way joins are implemented as we will see in a later post.

The second major concern is to reduce IO bandwidth. Disks are eras worth of time slower than memory. Unfortunately, tables almost always <i>have</i> to reside in files on the disk. For example, during Bob's query, the evaluator can very easily see that the only fields required from the ```Cars``` relation is the ```CID``` and the ```MID```. It can safely ignore the rest of the columns when scanning through the table. When tables have a lot of columns, and only two or three of them are of interest in most queries, which is a quite common scenario, this is an easy optimization with big payoffs.

There are other factors that come into play for optimization operations, like cost-based analysis. The objective always is to filter out as much of the data as soon as possible. 

In the next [post]({% post_url 2015-09-23-designing-a-sql-evaluator-ast-optimizations %}), we will see how we can represent a SQL query as an algebra, called the Relational Algebra. This allows us to reason about SQL queries at a higher level of abstraction. Then we will see how to efficiently represent a SQL query as a tree called the Abstract Syntax Tree or the RA AST, and then use that as a basis for our otimizations.
