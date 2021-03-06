---
title: "Getting Started with Titanium, a Clojure graph library"
layout: article
---

## About this guide

> Every journey begins with a single `(`. 

This guide is meant to provide a quick taste of Titanium and all the
power it provides. It should take about 10 minutes to read and study
the provided code examples. The contents include:

 * What Titanium is
 * What Titanium is not
 * Clojure and Titan version requirements
 * How to include Titanium in your project
 * A very brief introduction to graph databases
 * How to create vertices and edges
 * How to find vertices again
 * How to execute simple queries
 * How to remove objects
 * Graph theory for smug lisp weenies

This work is licensed under a <a
rel="license"href="http://creativecommons.org/licenses/by/3.0/">Creative
Commons Attribution 3.0 Unported License</a> (including images &
stylesheets). The source is available
[on Github](https://github.com/clojurewerkz/titanium.docs).

## What version of Titanium does this guide cover?

This guide covers Titanium 1.0.0-beta1.



## Titanium Overview

Titanium is a Clojure graph library built on top of
[Aurelius Titan](http://thinkaurelius.github.com/titan/). Titanium
strives to be easy to use, support all Titan features including the
various storage backends (Cassandra, HBase, BerkeleyDB Java Edition),
and takes the "batteries included" approach towards development.

To learn more about the Titan database, please see the following:

 * [The Rise of Big Graph Data](http://www.slideshare.net/slidarko/titan-the-rise-of-big-graph-data)
 * [Big Graph Data With Cassandra](http://www.youtube.com/watch?v=ZkAYA4Kd8JE)


## What Titanium is not

Titanium is not a database. Titan relies on other data store engines
to take care of on-disk durability. Titanium's value is in providing
an expressive Clojure API for graph operations. Titanium is not an
ORM/ODM. It does not provide graph visualization features. Depending
on the definition, Titanium may or may not be Web Scale. Currently,
the developers of Titanium are focused on correctness and productivity
and not benchmarks.

## Supported Clojure versions

Titanium is built from the ground up for Clojure 1.4 and later. The
most recent stable release is always recommended.

## Adding Titanium dependency to your project

### With Leiningen

``` clojure
[clojurewerkz/titanium "1.0.0-beta1"]
```

### With Maven

Add Clojars repository definition to your `pom.xml`:

```xml
<repository>
  <id>clojars.org</id>
  <url>http://clojars.org/repo</url>
</repository>
```


And then add the dependency:

``` xml
<dependency>
   <groupId>clojurewerkz</groupId>
    <artifactId>titanium</artifactId>
   <version>1.0.0-beta1</version>
</dependency>
```

It is recommended to stay up-to-date with new versions. New releases
and important changes are announced via
[@ClojureWerkz](http://twitter.com/ClojureWerkz) and the
[Clojurewerkz blog](http://blog.clojurewerkz.org/).


## Brief introduction to graph databases

A graph is a data structure that represents objects and the
connections between them. The objects being connected are called
"vertices" or "nodes" and the connections are called "edges" or
"relationships". Vertices may have a multitude of different
properties. A vertex could be used to represent a person and so it
might have properties for age, name, and gender. An edge links two
vertices and it too can have many different properties. If two
vertices represented two people, an edge connecting them could
represent their friendship and have properties that detailed the last
time they spoke and when the friendship started. In addition, there
can be multiple edges between any two given nodes and edges can have a
direction associated with them.

## Opening a graph

Titanium can
[be configured](https://github.com/thinkaurelius/titan/wiki/Graph-Configuration)
to use
[many different durable storage backends](https://github.com/thinkaurelius/titan/wiki/Storage-Backend-Overview):

 * [BerkeleyDB](https://github.com/thinkaurelius/titan/wiki/Using-BerkeleyDB)
 * [Apache Cassandra](https://github.com/thinkaurelius/titan/wiki/Using-Cassandra)
 * [Apache HBase](https://github.com/thinkaurelius/titan/wiki/Using-HBase)

Additionally, Titanium can also store graphs in memory.

In this guide, we will be introducing some features of Titanium using
an embedded instance of BerkeleyDB. For information about how to use
other database backends, please see the [configuration guide](https://github.com/thinkaurelius/titan/wiki/Graph-Configuration). To
start working with Titan, we will pass a map of configuration
properties to `clojurewerkz.titanium.graph/open`. The function will
use the Titan API to open up a new graph with the specified
configuration and store the resulting object in a dynamic var.


``` clojure
(ns titanium.pre-intro
  (:require [clojurewerkz.titanium.graph :as tg]))

(defn- main
  [& args]
  ;; opens a BerkeleyDB-backed graph database in a temporary directory
  (tg/open (System/getProperty "java.io.tmpdir"))
  "Graph business goes here")
(main)
```

If that worked, then you've installed everything correctly and can get
started with Titanium. For the rest of the tutorial we will be working
in the following namespace. You'll soon learn what all the required
namespaces do and how they interact.

``` clojure
(ns titanium.intro
  (:require [clojurewerkz.titanium.graph    :as tg]
            [clojurewerkz.titanium.edges    :as te]
            [clojurewerkz.titanium.vertices :as tv]
            [clojurewerkz.titanium.types    :as tt]
            [clojurewerkz.titanium.query    :as tq]))
            
(tg/open (System/getProperty "java.io.tmpdir"))            
```

## Creating Vertices

Vertices are created using `clojurewerkz.titanium.vertices/create!`.
This function optionally takes in a map of properties to assign to the
node. Please note that `clojurewerkz.titanium.graph/transact!` must
always be the context in which any code touching the database is run.
For more information, please see the [transaction management guide](/articles/transactions.html).

To create a single empty node: 

``` clojure
(tg/transact! (tv/create!))
```

To create two nodes with associated data:

``` clojure
(tg/transact!
 (tv/create! {:name "Michael" :location "Europe"})
 (tv/create! {:name "Zack"    :location "America"}))
```

## Creating Edges

Now that we know how to create vertices, we can begin creating edges
with the `clojurewerkz.titanium.edges/connect!` function. Edges can
optionally have properties, just like vertices.

``` clojure
(tg/transact!
 (let [p1 (tv/create! {:title "ClojureWerkz" :url "http://clojurewerkz.org"})
       p2 (tv/create! {:title "Titanium"     :url "http://titanium.clojurewerkz.org"})]
   (te/connect! p1 :meaningless p2)
   (te/connect! p1 :meaningful  p2 {:verified-on "February 11th, 2013"})))
```

## Indexing and Retrieving Vertices 

Titanium provides a
variety of functions for indexing and finding various objects.
`clojurewerkz.titanium.vertices/find-by-kv` takes in a keyword and a
value and finds all of the vertices with the corresponding key/value
pair. `clojurewerkz.titanium.types/defkey` takes in a
keyword, a class, and, optionally, a map specifying how to configure
the type and then creates a
[corresponding type in the Titan database](https://github.com/thinkaurelius/titan/wiki/Type-Definition-Overview). 

```clojure
(tg/transact! 
 (tt/defkey :age Long {:indexed-vertex? true :unique-direction :out}))

(tg/transact! 
 (tv/create! {:name "Zack"   :age 22}))
(tg/transact! 
 (tv/create! {:name "Trent"  :age 22}))
(tg/transact! 
 (tv/create! {:name "Vivian" :age 19}))


(tg/transact!
 (tv/find-by-kv :age 22))
```

## Simple Queries 

The final building block we need is to be able to ask questions about
the graph we are constructing. Simple queries are handled by functions
in the `clojurewerkz.titanium.query` namespace, while more complex
queries can be performed using [Ogre](http://ogre.clojurewerkz.org/).

One of the most basic questions when working with graphs is
determining the number of edges a vertex has. This quantity is
commonly called the degree of the vertex. We can use various functions
from `clojurewerkz.titanium.query` to create a simple `degree-of`
function that provides the degree of a given vertex. 

``` clojure
(defn degree-of 
  "Finds the degree of the given vertex."
  [v]
  (tq/count v))

(tg/transact!
 (let [v1 (tv/create!)
       v2 (tv/create!)
       v3 (tv/create!)
       v4 (tv/create!)]
   (te/connect! v1 :link v2)
   (te/connect! v1 :link v3)
   (println (map degree-of [v1 v2 v3 v4]))))
```

## Removing Objects

Removing object from a graph is straightforward. Call
`clojurewerkz.titanium.edges/remove!` to remove an edge. To remove
vertices, first make sure that all edges incident to the vertex are
removed and then call `clojurewerkz.titanium.vertices/remove!`. Let's
use these methods to write a method that clears a database completely
of all it's objects. We'll use
`clojurewerkz.titanium.vertices/get-all-vertices` and
`clojurewerkz.titanium.edges/get-all-edges` to make this task
straightforward.

``` clojure
(defn clear-graph! 
 "Clears the graph of all objects."
 []
 (doseq [e (te/get-all-edges)]
   (te/remove! e))
 (doseq [v (tv/get-all-vertices)]
   (tv/remove! v)))

(tg/transact!
 (clear-graph!)    
 (let [v1 (tv/create!)
       v2 (tv/create!)
       v3 (tv/create!)
       v4 (tv/create!)]
   (te/connect! v1 :link v2)
   (te/connect! v1 :link v3)
   (println (count (tv/get-all-vertices)) (count (te/get-all-edges)))
   (clear-graph!)    
   (println (count (tv/get-all-vertices)) (count (te/get-all-edges)))))
```

## A Bit of Graph Theory

Now we know how to create vertices and connect them together and ask
questions about the degree of those vertices. Which, when you think
about, covers, like, half of graph theory. Jokes aside, we can already
start doing some interesting experiments. For this example, we will
switch over to an in memory graph, since we don't much care about
saving any of these nodes.

Let's calculate the average degree for the
[complete graph](http://en.wikipedia.org/wiki/Complete_graph):

``` clojure
(defn average-degree 
  "Finds the average for the degree of the vertices."
  []
  (float (/ (count (te/get-all-edges))
            (count (tv/get-all-vertices)))))

(defn complete-graph 
  "Clears the graph and generates a complete graph."
  [n]
  (clear-graph!)
  (let [vs (map #(tv/create! {:i %}) (range n))]      
    (doseq [v vs w vs :when (not= v w)]
      (te/connect! v :link w))))

(tg/open {"storage.backend" "inmemory"})
(doseq [i (range 1 10)]
  (tg/transact!
   (complete-graph i)
   (println i (average-degree))))
```

Which gives results which are completely as expected. Let's get
tricky and build a random graph. It'll have n nodes and the chances
that any two nodes will be connected will be one half. What does the
average degree become then? It'll vary, so we'll run make sure to
run the experiment a few times. 

``` clojure
(defn random-graph 
  "Clears the graph and generates a random graph."
  [n]
  (clear-graph!)
  (let [vs (map #(tv/create! {:i %}) (range n))]      
    (doseq [v vs w vs :when (and (not= v w) (> 0.5 (rand)))]
      (te/connect! v :link w))))
      
(defn run-experiment [i]
  (tg/transact!
   (random-graph i)
   (average-degree)))

(defn average [col]
 (/ (reduce + col) (count col)))

(doseq [i (range 1 10)]
  (let [results (map run-experiment (take 10 (cycle [i])))]
    (println i (average results))))
```

## Wrapping up

Congratulations, you now can use Titanium to do basic graph theory!
Now, it is time to start learning enough to start building something
real. There are many features that we haven't covered here; they will
be explored in the rest of the guides. We hope you find Titanium
reliable, consistent and easy to use. In case you need help, please
ask on the
[mailing list](https://groups.google.com/forum/#!forum/clojure-titanium),
subscribe to [our blog](http://blog.clojurewerkz.org) and/or
[follow us on Twitter](http://twitter.com/ClojureWerkz).

## What's next

We recommend that you read the following guides in this order:

 * [Transaction Management](/articles/transactions.html)
 * [Configuring Titan](/articles/configuration.html) 
 * [Working with Vertices](/articles/vertices.html)
 * [Working with Edges](/articles/edges.html) 
 * [Defining Types](/articles/types.html)  
 * [Ogre Integration](/articles/ogre.html)


## Tell Us What You Think!

Please take a moment to tell us what you think about this guide on
Twitter or the [Titanium mailing
list](https://groups.google.com/forum/#!forum/clojure-titanium)

Let us know what was unclear or what has not been covered. Maybe you
do not like the guide style or grammar or discover spelling
mistakes. Reader feedback is key to making the documentation better.
