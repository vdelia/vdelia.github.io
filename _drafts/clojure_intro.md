---
layout: post
title:  "Getting Started with Clojure"
date:   2015-06-26 23:58:58
categories: [clojure, intro]
permalink: /clojure/intro/
tags:
  - clojure
  - install
  - LightTable
  - leiningen
---

Why Clojure? Pick your favorite answer from [Quora][quora], and don't forget that the most important one is that **Clojure is fun**.

About Lisp: yes, Clojure belongs to the glorious family of Lisp languages. Full stop. No more on this, because Common Lisp is completely different from Scheme, and the two have very little in common with Clojure.

So, why is it fun?

1.  Great [tooling][lein] for project automation

2.  Clojure code is a sequence of lists, called forms, that you can manipulate exactly as other data lists. The right way to create the API of your framework is to provide the most convenient syntax to use it. That's way it is so great to create Domain Specific Languages (DSLs). More elaboration on this? Check Paul Graham.

3.  It targets the JVM, and the layer separating Clojure from Java is ultra-tight. Using Java code is immediate.

4.  It is a great platform for experimentation. What is the next paradigm you want to play with? For sure you will find an implementation in Clojure. Think about asynchronous programming based on [channels][core-async] (something popularized by Go), [Software Transactional Memory][stm],
or [transducers][transducers-blog].

5.  If you like data analysis, you find Clojure DSLs everywhere: [Cascalog][cascalog] for Hadoop, [Flambo][flambo] for Spark, [Storm][storm] has a nice Clojure DSL, [Riemann][riemann] etc.

6.  Web development in Clojure is cool. On the backend, you have an modular ecosystem of components talking [ring][ring]. But Clojure is a good choice for the frontend also. [Clojurescript][cljscript] is an effective Clojure compiler targeting javascript, and providing Sbindings to Javascript code. I mean... it's Node.js but the other way around.

These are my notes on Clojure, and these five points are also the TOC for this series of posts.

## Install

**You need a JDK installed. I assume you have it.**

The Clojure compiler is distributed as a jar file. You can download this file from [clojure.org][cljwebsite], and run it using `java`,
but I strongly suggest to use [**Leiningen**][lein].
It makes incredibily easy to manage clojure installations, as well as the handling of libraries your projects depend on.

### Leiningen

Leiningen is probably the most popular Clojure project. It is the Clojure package manager, and it takes care of

* building new projects, whose layout is based on templates

* downloading the specified version of Clojure (which by the way comes as a jar archive)

* downloading all the dependencies of your project. The community repository of Clojure libs is [Clojars][clojars]. It handled Java dependencies as well, targeting any Maven repository.

* building, compiling, running, deploying your project

It has a rich set of plugins, that allows to extend its capabilities.
There is much delivered in that little script! Not really.
It's a little wrapper around the leiningen jar, which will be downloaded the first time you run it.

To **install**, you just have to download the [lein][lein-script] (or [lein.bat][lein-bat] if you are on Windows) script.

On Linux or Mac you need to run something like

{% highlight bash linenos=table %}
$ mkdir ~/bin
$ cd ~/bin
$ wget https://raw.githubusercontent.com/technomancy/leiningen/stable/bin/lein
$ export PATH=~/bin:$PATH
{% endhighlight %}

on Windows, download  `lein.bat`. Then open the command prompt, go in the directory containing the script
and run

{% highlight powershell linenos=table %}
> lein.bat self-install
{% endhighlight %}


Now it's time for the first project.

### Hello world

So, assuming it is in your path, run

{% highlight bash linenos=table %}
$ lein new app helloworld
{% endhighlight %}

It will create for you a directory called `helloworld` containing the stub of a project.

Check leiningen documentation about the project templates. Important parts are:
1.  `project.clj`, the leiningen configuration. The `clj` extension is used for Clojure source code files. Indeed, that's Clojure code.

2.  `src` for source code. There is one directory for the namespace `helloworld`, which contains the file `core.clj`.

3.  `tests` for unit tests.

Type

{% highlight bash linenos=table %}
$ lein repl
{% endhighlight %}

to ask for a [REPL][repl].

The first time you run it, it will take some time, because it has to download everything, including the Clojure jar.
You should get something like

{% highlight bash linenos=table %}
$ lein repl
nREPL server started on port 52074 on host 127.0.0.1 - nrepl://127.0.0.1:52074
REPL-y 0.3.5, nREPL 0.2.6
Clojure 1.6.0
Java HotSpot(TM) 64-Bit Server VM 1.8.0_40-b27
    Docs: (doc function-name-here)
          (find-doc "part-of-name-here")
  Source: (source function-name-here)
 Javadoc: (javadoc java-object-or-class-here)
    Exit: Control+D or (exit) or (quit)

helloworld.core=>
{% endhighlight %}

Congratulations! You correctly installed everything.

The most interesting part of the output is the first line. A server is running on our local machine and it is executing a REPL.

In the REPL you can submit Clojure code for evaluation, get the results, and get the prompt back waiting for more input. This is the mechanism used by all IDEs  to give us an interactive environment.

Want to see the `Hello world` message coming up?
If you are still in the REPL, run

{% highlight bash linenos=table %}
helloworld.core=> (-main)
Hello, World!
nil
{% endhighlight %}

The prompt informs us that we are in the `helloworld.core` namespace, and we ask to the REPL to evaluate the function `-main`, without arguments. It prints `Hello, World!`, and it does not return any value, so it evaluates to `nil`.
Who provided the `-main`? lein in its template called `app` (remember the `new app` stuff?).

If you quit the REPL with `CTRL-D`, you can discover other leiningen functionalities.

You can run directly the project with

{% highlight bash linenos=table %}
$ lein run
Hello, World!
{% endhighlight %}

or compile and build a jar ready to be deployed with
{% highlight bash linenos=table %}
$ lein uberjar
$ java -jar target/uberjar/helloworld-0.1.0-SNAPSHOT-standalone.jar
Hello, World!
{% endhighlight %}

The standalone `jar` will contain all it's needed to run the project, included the clojure jar and all dependencies.

So, the question is now... which IDE? I am a tmux + vim enthusiast, and maybe I will cover it in another post.
But here I'm going to play with [LightTable][lighttable]. I really enjoy it, and it runs very well on Mac, Linux and Windows.

### LightTable
Point your browser to its [website][lighttable] and download the big archive containing everything.
Uncompress it where you can find it, and that's it. Installation done.

In the directory, run the executable, and you are ready to code.

In the toolbar, click on `View > Connections`. A new pane will open on the right. Click on `Add connection`, and choose `Clojure`. You will have to pick a `project.clj` file, and LightTable will start executing a REPL in background.

![LightTable connections]({{ site.url }}/assets/clj/light_connections.png)

Then open the Console, with `View > Console`. This is useful to see stdout/stderr.

How can you interact with the connection with the REPL?
Click on `View > Commands`. A new pane on the right opens, containing the list of available commands.

![LightTable commands]({{ site.url }}/assets/clj/light_commands.png)

Type `repl`, and pick `Instantrepl: open a Clojure Instantrepl`.

![LightTable commands]({{ site.url }}/assets/clj/light_instantrepl.png)

This is an editor window where you can write any Clojure form, and get it evaluated.
This is made explicit by the UI. On the top-right corner of the editor you should have a blue label telling you that you are going `Live`.

You can evaluate the `-main` created by leiningen, but we need to specify the namespace, so either

{% highlight clojure linenos=table %}
(helloworld.core/-main)
{% endhighlight %}

either

{% highlight clojure linenos=table %}
(ns helloworld.core)
(-main)
{% endhighlight %}

In both cases you will see a `nil` at the end of the form, and a `Hello, world!` line in the console.
This is a very convenient way of experimenting with Clojure, in particular because it's interactive to the same degree of a REPL, but you can edit it/save it as in a text editor.

Thus, the next trick is quiet expected. LightTable can turn any editor window in a REPL.
First of all, you need to open your project.
With `File > Open Folder` you can navigate to your `helloworld` project and open it. Open the file `src/helloworld/core.clj`. That's where we will start to code.

Open again the `View > Commands` panel, type `Instantrepl`, and pick `Instantrepl: make current editor an Instantrepl`.

This time, LightTable will evaluate every form in the editor file, and will add the output of its evaluation at the end of it.

![LightTable instant repl editor]({{ site.url }}/assets/clj/light_instantrepl_editor.png)

You can toggle this mode on/off simply by clicking on the `live` label on the top right of the window.


[lighttable]: http://lighttable.com

[repl]: https://en.wikipedia.org/wiki/Read–eval–print_loop

[lein-script]: https://raw.githubusercontent.com/technomancy/leiningen/stable/bin/lein

[lein-bat]: https://raw.githubusercontent.com/technomancy/leiningen/stable/bin/lein.bat

[clojars]: https://clojars.org
[lein]: http://leiningen.org
[quora]:   http://www.quora.com/Why-would-someone-learn-Clojure
[cascalog]: http://cascalog.org
[flambo]: https://github.com/yieldbot/flambo
[storm]: https://storm.apache.org/documentation/Clojure-DSL.html
[riemann]: http://riemann.io
[cljscript]: https://github.com/clojure/clojurescript
[stm]: https://en.wikipedia.org/wiki/Software_transactional_memory
[ring]: https://github.com/ring-clojure/ring
[core-async]: http://clojure.com/blog/2013/06/28/clojure-core-async-channels.html
[transducers-blog]: http://blog.cognitect.com/blog/2014/8/6/transducers-are-coming
[cljwebsite]:  http://clojure.org
