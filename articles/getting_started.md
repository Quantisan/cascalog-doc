---
title: "Getting Started with Cascalog"
layout: article
---

## About this guide

This guide is a quick tutorial to help you get started with using Cascalog. It should take about 30 minutes to go through these examples. This guide covers:

1. How to add Cascalog dependency to your project
2. Word count example in both Java and Clojure
3. Run a Cascalog query from Clojure nREPL
4. Introduction to basic concepts
5. Pointers for where to go next

## What version of Cascalog does this guide cover?

This guide covers Cascalog 3.0.0.

## Prerequisites

1. A Hadoop installation. You can [setup a local single-node Hadoop](http://wiki.apache.org/hadoop/GettingStartedWithHadoop) on your computer.
2. [Leiningen](https://github.com/technomancy/leiningen) or Maven.

## Adding Cascalog dependency to your project

Clojure artifacts are released to [Clojars](https://clojars.org/cascalog) repository.

### With Leiningen

Add dependency to Cascalog.

```clj
[cascalog/cascalog-core "3.0.0"]
```

Add development dependency to Hadoop in your `project.clj`.

```clj
:profiles { :dev {:dependencies [[org.apache.hadoop/hadoop-core "1.2.1"]]}}
```

Bump up heap size for running Hadoop in local mode, also in your `project.clj`. Make sure that your heap size is set to at least 768 MB.

```clj
:jvm-opts ["-Xms768m" "-Xmx768m"]
```

### With Maven

If you prefer to use Maven instead of Leiningen, add Clojars to your repositories.

```xml
<repository>
  <id>clojars.org</id>
  <url>http://clojars.org/repo</url>
</repository>
```

Then add Cascalog dependency to your project.

```xml
<dependency>
  <groupId>cascalog</groupId>
  <artifactId>cascalog-core</artifactId>
  <version>3.0.0</version>
</dependency>
```

## Use Cascalog through Java or Clojure?

The native implementation of Cascalog is in Clojure. However, Cascalog provides a pure-Java interface called JCascalog that is perfectly interoperable with the Clojure version. Choose whichever interface that is comfortable for you to get started.

## Word count example in Cascalog

Ready to begin? Bring up the Clojure prompt using Leiningen with `lein repl` in your project. Once you see `user=>`, type in the following to use the main Cascalog API.

```clj
(use 'cascalog.api)
```

The prompt should print `nil` and return control to you. For ease of demonstration, also load the built-in example namespace.

```clj
(use 'cascalog.playground)
```

Now that we have everything ready, let's get some words to be counted. Conveniently, the `cascalog.playground` namespace contains a var `sentence` containing sentences. Take a look at it by typing:

```clj
sentence
```

Which prints out a series of vectors each of size 1.

### Execution operator

Cascalog queries contain a source and a sink. The most basic of which as shown here is to read your data and directly write it back out.

```clj
(?- (stdout)
    sentence)
```

Running this will kick off a job in your local hadoop node and prints off the result.

`?-` is the query execution operator in Cascalog. It takes a sequence of \<output tap, query\> pairs, and executes all supplied queries in parallel. Here, we have defined the *output tap* on the first line as `(stdout)`, which prints to the standard output. The second line contains the query. In this example though, there is no querying as we merely specified an in-memory *generator* as the source. A **generator** is a source of data. This particular query is similar to doing `SELECT * FROM sentence` in SQL.

### Query operator

To actually perform a query, we introduce the query creation operation -- `<-`.

```clj
(?- (stdout)
    (<- [?line]
        (sentence :> ?line)))
```

The result from this query is the same as the previous one. However, here we have an explicit query with *output field* between square brackets, `[?line]`, *generator* `sentence`, *input field*, also named `?line`. `:>` and it's sibling
`:<` are examples of Cascalog predicate operators which treat variables on one side as input to what's on the other.

This query is similar to performing `SELECT line FROM sentence` with SQL.

As Cascalog is declarative, we tell it what we want and the logic solver figures out how to do it. In this example, we wanted output `[?line]`. Which happens to be the same as input `?line`. The fact that the two fields have the same name is treated as though they are the same entity. Thus, given that we want `[?line]`, and input `?line` is fed by `sentence`, this query reads in data from our *generator* `sentence` and pipes it directly out.

### Data Tuple

To understand how to write Cascalog queries, we need to understand *Tuple*. In Cascading and Cascalog, the basic data structure is a *Tuple*. Let's digress briefly here to take a look at other variables in the `cascalog.playground` namespace.

```clj
=> (take 2 person)
[["alice"]
 ["bob"]]
```

`person` is a series of 1-Tuples each with one element. To use `person` as a data generator in a Cascalog query, you would do `(person ?name)` to assign the Tuple element to `?name`.

Whereas `age` is a series of 2-Tuples.

```clj
=> (take 2 age)
[["alice" 28]
 ["bob" 33]]
```

To use `age` as a data generator in a query, you do `(age ?name ?age)` to assign the 2-Tuples to two vars. `?name` and `?age` can be thought of as a columns (spanning horizontally) whereas each entry can be thought of as rows (spanning vertically).

### Operation

Once you bind your data *Tuple* to vars, you can operate on them individually (via `defn`, `defmapop`, etc), as a group either horizontally (via operators) or vertically (via *aggregator*), as well as transposing horizontal to vertical (e.g. `defmapcatop`), and vice versa (e.g. `defbufferop`). We will discuss the different types of operations in Cascalog later.

Let's continue with the word count example and process the sentence by tokenising our lines into words and thus spanning each 1-Tuple sentence vertically into multiple 1-Tuple words.

```clj
(?- (stdout)
    (<- [?word]
        (sentence :> ?line)
        (tokenise :< ?line :> ?word)))
```

`tokenise` is a custom operator as defined below. It takes input `?line` and return its result to `?word`. Notice the use of the input predicate `:<` here.

`defmapcatop` takes each Tuple and returns multiple Tuples. It is just a regular Clojure function that returns a sequence.

```clj
(require '[cascalog.logic.def :as def])

(def/defmapcatfn tokenise [line]
  "reads in a line of string and splits it by a regular expression"
  (clojure.string/split line #"[\[\]\\\(\),.)\s]+"))
```

If you run `tokenise` as a regular function, you will get an exception.

```
user=> (tokenise sentence)

ClassCastException clojure.lang.PersistentVector cannot be cast to java.lang.CharSequence  clojure.string/split (string.clj:222)
```

This is because `tokenise` is not a regular Clojure function but a Cascalog operator. Before we see `tokenise` in action, let's take a look at its expected input and output tuples to get a sense of what it does first. Consider the first Tuple of `sentence`.

```clj
[["Four score and seven years ago our fathers brought forth on this continent a new nation"]]
```

This is a 1-Tuple with one element in each row, and one row only. Passing it into `tokenise` would return a 1-Tuple also with one element in each row, but with 16 rows, each containing a word.

```clj
[["Four"]
 ["score"]
 ["and"]
 ["seven"]
 ["years"]
 ["ago"]
 ["our"]
 ["fathers"]
 ["brought"]
 ["forth"]
 ["on"]
 ["this"]
 ["continent"]
 ["a"]
 ["new"]
 ["nation"]]
```

To use `tokenise`, we need to put it a Cascalog query and use it as an operator.

### Aggregation

Now that all the words in `sentence` has been tokenised, we can group and count them. First we `require` the Cascalog operator namespace.

```clj
(require '[cascalog.logic.ops :as c])
```

And use the built-in `count` operator as such. If you have followed through with this tutorial and executed the sample code, you should get a word count of the `sentence` data. Otherwise, copy and paste the code chunk in the next section into your REPL and return here to continue.

```clj
(?- (stdout)
    (<- [?word ?count]
        (sentence :> ?line)
        (tokenise :< ?line :> ?word)
        (c/count :> ?count)))
```

Recall that we write Cascalog queries by telling it (1) what we want, (2) what's the input, and (3) the contraints, then the logic solver would figure out what to do. This word count query above is a good example.

In this query, we want each `?word` and the number of occurrences of it, i.e. output `[?word ?count]`. The input data is a 1-Tuple `sentence` and we assign single element to `?line`. We `tokenise` each line `?line` to get our words, `?word`.

By now, we have `?word` var satisfied. Then the query just needs to solve for `?count`. In the last statement, we apply the output of `c/count` operator to `?count`. This is where Cascalog differs.

To perform a count in SQL, we would do,

```sql
SELECT    word, COUNT(*)
FROM      words
GROUP BY  word
```

But in Cascalog, the GROUP BY is implicit because `?word` output is already satisfied in the query constraint, the `c/count` would count all rows grouped by `?word`.

### Shorthand

It is often the case that you would want to define and execute your queries at the same time, so there's a query execute and definition function `?<-` that combines `?-` and `<-` together.

Furthermore, both input and output predicates, `:<` and `:>`, are optional if the expression is not ambiguous. For example, `sentence` generator only outputs to `?line`, so we don't need to use `:>` here as it is evident. Whereas `tokenise` has both input and output. So we only need to specify the output predicate `:>` as the input `?line` is obvious since it is not an output.

```clj
(?<- (stdout)
     [?word ?count]
     (sentence ?line)
     (tokenise ?line :> ?word)
     (c/count ?count))
```

## Word count example in JCascalog

### JCascalog overview

JCascalog is a pure-Java interface to Cascalog that comes bundled with Cascalog as of version 1.8.7. All the functionality available in Cascalog is available via the JCascalog interface. Moreso, Cascalog and JCascalog are perfectly interoperable: JCascalog subqueries, operations, and predicate macros can be used in regular Cascalog code and vice-versa.

### Example

Work In Progress. Refer to [JCascalog basics on Wiki](https://github.com/nathanmarz/cascalog/wiki/JCascalog) for now.

## What next?

There are a lot more features to Cascalog to make your life munging big data easier, [continue with the rest of our guides to learn how](/articles/guides.html).
