---
layout: post
title:  "Clojure"
date:   2015-06-26 23:58:58
categories: lisp-for-the-masses
tags: clojure leiningen
---

# Why Clojure?

Pick your favorite answer from [Quora][quora], and don't forget that most important one is that **Clojure is fun**.

About Lisp: yes, Clojure belongs to the Lisp family of languages. Full-stop. No more on this, because Common Lisp is completely different from Scheme, and the two have very little in common with Clojure.  

So, why it is fun?

1.  It runs of the JVM, and the layer separating Clojure
    from Java is ultra-tight. Using Java code is immediate.

2.  Clojure code is a sequence of lists, called forms, that you can manipulate exactly in the way you manipulate lists of integers. The right way to create the API of your framework is to create the syntax you need.

3.  If you want to experiment with new programming paradigms, for sure you will find a reference implementation in Clojure. Really. Think about [Software Transactional Memory][stm].

4.  If you like data analysis, you find Clojure DSLs everywhere: [Cascalog][cascalog] for Hadoop, [Flambo][flambo] for Spark, [Storm][storm] has a nice Clojure DSL, [Riemann][riemann] etc.

5.  Web development in Clojure is cool. On the backend, you have an modular ecosystem of components talking [ring][ring]. But it is not limited to the backend part. [Clojurescript][cljscript] is an effective Clojure compiler targeting javascript, and providing tight bindings to Javascript code.

[quora]:   http://www.quora.com/Why-would-someone-learn-Clojure
[cascalog]: http://cascalog.org
[flambo]: https://github.com/yieldbot/flambo
[storm]: https://storm.apache.org/documentation/Clojure-DSL.html
[riemann]: http://riemann.io
[cljscript]: https://github.com/clojure/clojurescript
[stm]: https://en.wikipedia.org/wiki/Software_transactional_memory
[ring]: https://github.com/ring-clojure/ring
