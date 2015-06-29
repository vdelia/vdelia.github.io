---
layout: post
title:  "Clojure basics"
date:   2015-06-26 23:58:58
categories: [clojure, basics]
permalink: /clojure/basics/
tags:
  - clojure
  - basics
---

In this post I cover the basics of the Clojure programming language. Then buy a book.

## Syntax

In Clojure you don't have a syntax. I won't spend a paragraph to explain you that to write a conditional you need the two characters `if`,
followed by a condition between `(` and  `)`, and so on.
A Clojure program is built by creating a sequence of data structures, called **forms**, which are evaluated by the compiler. You can think of it as if the syntax of the language would be `json`.

Forms can be written in a file (*programming*), or produced by evaluating other forms ([*metaprogramming*][metaprog]).

The **Reader** is the component which reads sequence of characters, and produces the forms which are then processed by the compiler.


## The Reader

The REPL, or even better the InstantRepl of LightTable, is the best way to understand how the Reader
[metaprog]: https://en.wikipedia.org/wiki/Metaprogramming
