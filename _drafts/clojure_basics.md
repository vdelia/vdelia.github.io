---
layout: post
title:  "Clojure basics"
date:   2015-06-26 23:58:58
categories: [clojure, basics]
permalink: /clojure/basics/
tags:
  - clojure
  - functional-programming
---

In this post I cover the basics of the Clojure programming language. Then buy a book.

## Syntax
A Clojure program is a sequence of data structures, called **forms**, which are evaluated by the **compiler**.
Forms can be written in a source file (*programming*), and then loaded by the **reader**. Alternatively, they
are produced by evaluating other forms ([*metaprogramming*][metaprog]).
This last step allows the developer to create new syntax.

You can think of it as if the syntax of the language was `json`. The reader loads the textual representation from a source file, and
deserializes it to binary objects: constants, lists, etc.
Then, the compiler evaluates those objects to produce a result, perform side-effects, or generate
new `json` to evaluate.

The REPL, like the InstantRepl of LightTable, is the best tool to understand how this stuff works.

[metaprog]: https://en.wikipedia.org/wiki/Metaprogramming
