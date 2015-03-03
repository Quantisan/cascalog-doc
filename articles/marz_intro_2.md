---
title: "New Cascalog features: outer joins, combiners, sorting, and more"
layout: article
---

# New Cascalog features: outer joins, combiners, sorting, and more

Note: This guide is adapted from [Nathan Marz's blog post](nathanmarz.com/blog/new-cascalog-features-outer-joins-combiners-sorting-and-more.html) introducing the Cascalog project back in April 2010.

In the [first tutorial](/articles/marz_intro_1.html) for Cascalog, I showed off many of Cascalog's powerful features: joins, aggregates, subqueries, custom operations, and more. Since Cascalog's release a couple weeks ago, I've added a number of new features to Cascalog that seriously increase the expressiveness and performance of the language without compromising its simplicity or flexibility.

Like the first tutorial, go ahead and load up the playground by issuing the following commands:

```clj
lein compile-java && lein compile
lein repl
user=> (use 'cascalog.playground) (bootstrap)
```

## Outer joins

As we saw in the first tutorial, you can join together multiple sources of data in Cascalog by using the same variable name in multiple sources of data. For example, given "age" and "gender" sources, we can get the age and gender for each person by running:

```clj
user=> (?<- (stdout) [?person ?age ?gender]
          (age ?person ?age) (gender ?person ?gender))
```

This is an inner join. We will only have results for people that appear in both sets of data. We can do a full outer join by running:

```clj
user=> (?<- (stdout) [?person !!age !!gender]
          (age ?person !!age) (gender ?person !!gender))
```

The results of this query will have null values for people with nonexistent ages or genders.

Cascalog's outer joins are triggered by variables that begin with "!!". These variables are called "ungrounding variables". A predicate that contains an ungrounding variable is called an "unground predicate", and a predicate that does not contain an ungrounding variable is called a "ground predicate". Joining together two unground predicates results in a full outer join, while joining a ground predicate to an unground predicate results in a left join.

Here's an example of a left join. To get all the follow relationships for each person in our dataset, or null if the person has no follow relationships, we run:

```clj
user=> (?<- (stdout) [?person1 !!person2]
          (person ?person1) (follows ?person1 !!person2))
```

To get all the people who do not have a follows relationship, we can run:

```clj
user=> (?<- (stdout) [?person]
          (person ?person) (follows ?person !!p2) (nil? !!p2))
```

Notice that the (nil? !!p2) predicate gets applied after !!p2 gets joined to a ground predicate. This is an important part of the semantics of outer joins in Cascalog.

Now let's say we want the follows count for each person. A normal "count" aggregation won't work because it counts the number of tuples and doesn't distinguish between null and non-null follows. In this case, we want null follows to be counted as 0 and non-null follows to be counted as 1. Cascalog has an aggregator called "!count" that does exactly this:

```clj
user=> (?<- (stdout) [?person ?count]
          (person ?person) (follows ?person !!p2) (c/!count !!p2 :> ?count))
```

People that don't have a follows relationship will have a count of 0.

An ungrounding variable may only appear within a query one time. Other than that, ungrounding variables behave just like regular variables.

## Combiners and "Parallel Aggregators"

A regular aggregator transfers all tuples for a group to a single machine and computes the aggregation in a single pass over the data. However, there are many aggregations, such as count, sum, min, and max, that can be computed in parallel. For example, to compute "sum", you can split the tuples into subsets, compute the sum of each subset, and then sum the sums together to get your final answer. There are many other aggregators that can be computed this way, such as min, max, and count.

Cascalog now allows you to define "parallel aggregators" that compute as much as possible during the map phase before finishing the computation in the reducer. These map side aggregations are called "combiners". Cascalog is even able to insert combiners when you use multiple parallel aggregators, such as both a count and a sum. For example, the following query will make use of combiners:

```clj
user=> (?<- (stdout) [?count ?sum]
          (integer ?n) (c/sum ?n :> ?sum) (c/count ?count))
```

Cascalog automatically inserts combiners when possible - you don't have to do anything to take advantage of the optimization.

If you try to use a parallel aggregator with a regular aggregator defined using defaggregateop or defbufferop, Cascalog will be unable to insert combiners and all the aggregation will happen in the reduce task. For example, the next query that makes use of a custom aggregator will do all the aggregation in the reduce phase:

```clj
user=> (defaggregateop product
         ([] 1)
         ([total val] (* total val))
         ([total] [total]))
user=> (?<- (stdout) [?prod ?count]
          (integer ?n) (product ?n :> ?prod) (c/count ?count))
```

Parallel aggregators can be defined using the defparallelagg function. Examples can be found in cascalog.ops.

You'll see a massive speed boost due to this feature for aggregations that operate on very few groups, such as global counts.

## Implicit equality constraints

The "implicit equality constraints" feature is a neat way to specify equality constraints. This feature is best explained by example. The playground defines a source called "integer" that defines a set of numbers. If we want all the numbers that equal themselves when squared, we can run:

```clj
user=> (?<- (stdout) [?n] (integer ?n) (* ?n ?n :> ?n))
```

Cascalog detects that we are trying to rebind the ?n variable and will automatically filter out tuples where the output of the * predicate is not equal to the input.

There are other cases where you can make use of this feature. To find all the pairs of numbers in the "num-pair" source where both numbers are the same, we run:

```clj
user=> (?<- (stdout) [?n] (num-pair ?n ?n))
```

If you want to know all pairs where the second number is two times the first number:

```clj
user=> (?<- (stdout) [?n1 ?n2]
         (num-pair ?n1 ?n2) (* 2 ?n1 :> ?n2))
```

There's not much more to say about this feature, it should be intuitive to use.

## Sorting

By default, aggregators receive tuples in some arbitrary order. Cascalog now has ":sort" and ":reverse" predicates that let you control the order in which tuples arrive at an aggregator. For example, let's find the youngest person each person follows:

```clj
user=> (defbufferop first-tuple [tuples] (take 1 tuples))
user=> (?<- (stdout) [?person ?youngest] (follows ?person ?p2)
          (age ?p2 ?age) (:sort ?age) (first-tuple ?p2 :> ?youngest))
```

To find the oldest person each person follows, we simply add a :reverse predicate:

```clj
user=> (?<- (stdout) [?person ?youngest] (follows ?person ?p2)
          (age ?p2 ?age) (:sort ?age) (:reverse true)
          (first-tuple ?p2 :> ?youngest))
```          

## Duplicate elimination

If your query doesn't have any aggregators, Cascalog can, by default, insert a reduce step to remove all duplicate tuples from your output. 
Previous to version 2 of cascalog, you could control that behavior with the :distinct predicate. Compare the following two queries:

```clj
user=> (?<- (stdout) [?a] (age _ ?a))
user=> (?<- (stdout) [?a] (age _ ?a) (:distinct false))
```

The second query will have duplicates in the output. One use case for this functionality is making a subquery that does some pre-processing on an input source.
Note that version 2 of Cascalog defaults to (:distinct false) so to enable this behaviour, you would need to add a (:distinct true).
