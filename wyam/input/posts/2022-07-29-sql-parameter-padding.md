Title: SQL Parameter Padding
Published: 2022-07-29
Tags: [SQL]
---    

An interesting approach for SQL query optimization reducing the number of generated and cached query execution plans.  

## Background  

Let's look at these two SQL queries:  

```sql
select x from y where z in (1, 2, 3)
select x from y where z in (2, 3, 4)
```

The database query engine will create query plans for each one and cache them. This is because the queries are different. No matter that they may use the same query plan, they are different as much as the database engine is concerned. To reuse a cached query execution plan the query must literally match a previously executed SQL statement with a cached plan, character per character.

## Parametrization  

The way to deal with this is to parametrize the queries

```sql
@p1 = 1, @p2 = 2, @p3 = 3
select x from y where z in (@p1, @p2, @p3)

@p1 = 2, @p2 = 3, @p3 = 4
select x from y where z in (@p1, @p2, @p3)
```

Now both queries will reuse the same query execution plan.  
  
But the use of `IN` keyword implies we don't know the exact number of values we will want to match. If we look at these queries - we will again have an execution plan generated for each one:  

```sql
select x from y where z in (@p1, @p2, @p3)
select x from y where z in (@p1, @p2, @p3, @p4)
select x from y where z in (@p1, @p2, @p3, @p4, @p5)
```   

## Parameter Padding  

A technique called "parameter padding" can reduce the query execution plans generated and cached. We need to define a set of numbers of parameters and construct queries so the number of their parameters aligns with the numbers in the set. When a query has the exact number of parameters matching a value in the set then we are ok. Otherwise, we choose the minimum number in the set higher than the number of parameters we need to pass. For the surplus parameters we "pad" them by repeating one of the parameter values.  

Let's apply this to the last example. Let us choose our set of parameter numbers to be 4, 8, 16, etc.

```sql
@p1 = 1, @p2 = 2, @p3 = 3, @p4 = 3
select x from y where z in (@p1, @p2, @p3, @p4) <-- "pad" @p4 giving it the same value as @p3

@p1 = 5, @p2 = 6 @p3 = 7, @p4 = 8
select x from y where z in (@p1, @p2, @p3, @p4)

@p1 = 5, @p2 = 6 @p3 = 7, @p4 = 8, @p5 = 9, @p6 = 9 @p7 = 9, @p8 = 9
select x from y where z in (@p1, @p2, @p3, @p4, @p5, @p6, @p7, @p8) <-- again "pad" last 3 parameters
```  

Now the first two queries will reuse the same execution plan. We made them the same by adding parameter to the first one and repeating the 3rd one's value. We did the same "padding" for the third query, repeating the last value tree additional times to bring the number of parameters to 8.  

In fact, all such queries having up to 4 parameters will use the same plan, all having up to 8 will use the same plan, and so on. We effectively managed to reduce the number of generated and cached query execution plans from the maximum possible 8 to a maximum of 2 in the given example.  
  
Selecting the set of numbers for parameters will depend on concrete query patterns. They may form uniform or non-uniform intervals. There is no "one to fit them all" and we'll need to figure out what works best on a per-case basis.  

Note that for this to work:
  - the database we use generates and caches query execution plans,
  - for a given query we need to use the same set of parameter numbers.


## Conclusion  

Parameter padding is a way to optimize queries using the `IN` SQL keyword when the parameter number is varying. We do this by reducing the number of generated and cached execution plans. As for everything related to performance improvements - don't take it as a fact, always measure!  
