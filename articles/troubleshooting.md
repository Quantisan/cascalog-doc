---
title: "Troubleshooting, testing and live coding"
layout: article
---

Troubleshooting
---------------

### Catching data errors with traps

You can use [Cascading Traps](http://docs.cascading.org/cascading/2.0/userguide/htmlsingle/#N21366) with Cascalog to capture tuples whose processing fails. To store those tuples into a sink tap (for example a local file or hfs-textline), use the `:trap` keyword with an error sink:

```clojure
(def errors (lfs-textline "file:///tmp/people.bad_records" :sinkmode :replace)) 
;; or (stdout) or (hfs-textline "hdfs:///tmp/...") if running on Hadoop

(<- [?name ?age]
      (people ?name ?age)
      (:trap errors)
      (< ?age 40))
```

Testing
-------

You may use the functions and macros from the [midje.cascalog](https://github.com/nathanmarz/cascalog/blob/develop/midje-cascalog/src/midje/cascalog.clj) namespace together with clojure.test test your queries. See [Cascalog's own tests](https://github.com/nathanmarz/cascalog/tree/develop/cascalog-core/test/cascalog) for examples.

It uses for example `fact?-` to execute a query and compare its outputs with the expected ones or something like `(facts query => (produces [[3 10] [1 5] [5 11]])` where `(def query (<- ...))`. Read Sam Ritchie's blog post [Cascalog Testing 2.0 ](http://sritchie.github.io/2012/01/22/cascalog-testing-20/) for more details and examples of midje-cascalog 0.4.0.

Live coding
-----------

There are certain features that support live, interactive coding:

* Use simple Clojure collections as data sources (`(def people [["ben" 21] ["jim" 42]])`)
* You can during development easily change some parts of Cascalog code to standard Clojure functions and call them from the REPL, for example a custom operator by replacing `(defaggregateop ` with `(defn `.
* Queries can be of course executed from the REPL 

