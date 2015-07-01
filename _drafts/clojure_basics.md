---
layout: post
title:  "Clojure basics"
date:   2015-06-26 23:58:58
categories: [clojure, basics]
permalink: /clojure/basics/
tags:
  - clojure
  - functional programming
---

In this post I cover the basics of the Clojure programming language. Then buy a book.

## Syntax

Clojure has a very simple syntax. I won't spend a paragraph to explain you that to write a conditional you need the two characters `if`,
followed by a condition between `(` and  `)`, and so on.
A Clojure program is built by creating a sequence of data structures, called **forms**, which are evaluated by the compiler.
Forms can be written in a file (*programming*), or produced by evaluating other forms ([*metaprogramming*][metaprog]).

The **Reader** is the component which reads sequence of characters, and produces the forms which are then processed by the **compiler**.

You can think of it as if the syntax of the language would be `json`. The reader reads its textual representation from a source file, and
deserializes it to objects. Then, the compiler evaluates those objects to produce a result, perform side-effects, or potentially generate
new objects to evaluate.

## The Reader
The REPL, or even better the InstantRepl of LightTable, is the best way to understand how the Reader works.

[metaprog]: https://en.wikipedia.org/wiki/Metaprogramming