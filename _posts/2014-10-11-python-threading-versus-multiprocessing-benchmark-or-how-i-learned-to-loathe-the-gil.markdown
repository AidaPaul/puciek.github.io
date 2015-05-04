---
layout: post
title:  "Python: Threading versus Multiprocessing benchmark; Or how I learned to loathe the GIL."
date:   2014-10-11 12:15:0
---

Long gone are days where single-threaded software was a viable solution, nowadays even a proper simple hex editor is running multiple execution stacks to achieve whatever it does. And when we need that there are two main options – Multithreading and Multiprocessing. While I don’t feel like going into the whole debate of which one is better (and why the answer is multiprocessing), let’s just have a very quick look at how they perform CPU-based tasks. And, spoiler alert, threading fails at it big time (which won’t be a surprise to anyone who have used them).

<!--more-->

Let’s start with our contestants. Welcome singly.py, thready.py and the processy.py – our contestants. They will all do a very simple thing for us – countdown from 100000000, four times. And the one with the best result will be declared the winner.

singly.py
{% highlight python %}
def count(n):
 while n > 0:
  n -= 1
 
count(100000000)
count(100000000)
count(100000000)
count(100000000)
{% endhighlight %}

thready.py
{% highlight python %}
from threading import Thread
 
def count(n):
 while n > 0:
  n -= 1
 
t1 = Thread(target=count,args=(100000000,))
t1.start()
t2 = Thread(target=count,args=(100000000,))
t2.start()
t3 = Thread(target=count,args=(100000000,))
t3.start()
t4 = Thread(target=count,args=(100000000,))
t4.start()
t1.join(); t2.join(); t3.join(); t4.join()
{% endhighlight %}

processy.py
{% highlight python %}
from multiprocessing import Process
 
def count(n):
  while n > 0:
    n -= 1
 
p1 = Process(target=count,args=(100000000,))
p1.start()
p2 = Process(target=count,args=(100000000,))
p2.start()
p3 = Process(target=count,args=(100000000,))
p3.start()
p4 = Process(target=count,args=(100000000,))
p4.start()
 
p1.join(); p2.join(); p3.join(); p4.join()
{% endhighlight %}

To run the tests, I am using a Gentoo virtual machine running via Hyper-V with 12 logical threads and Python 3.3.3 (default, May 15 2014, 04:33:04). So, without further ado, let’s proceed to the results, starting with the worst possible case – singly.py, where we run the code sequentially, without any utilization of other CPUs.

{% highlight python %}
buildbox performanc_tests # time python singly.py
 
real    0m23.718s
user    0m23.710s
sys     0m0.000s
{% endhighlight %}

Not a bad result, especially given how crippled is the implementation. Now lets run my favorite way to approach this sort of tasks – multiprocessing.Process way.

{% highlight python %}
buildbox performanc_tests # time python processy.py
 
real    0m6.786s
user    0m24.880s
sys     0m0.000s
{% endhighlight %}

That is pretty much going as you would expect – we utilize four cores, so it takes, roughly 1/4th of the real time as the singly.py, but the user time remains roughly the same. And that leaves us only with one to test – the dreaded… I mean threaded one.

{% highlight python %}
buildbox performanc_tests # time python thready.py
 
real    0m23.784s
user    0m23.480s
sys     0m0.280s
{% endhighlight %}

Keep in mind that data provided here is an average of ten runs. I’ve presented them in bash-y form just for the dramatic effect. And it sure is dramatic – turns out that threading doesn’t work at all. At least not when it comes strictly to computation needs. The obvious question to ask is – why? And the answer is – GIL.

What is GIL? It’s effect of laziness of some developers who decided that there is no point in putting the extra effort to make memory management in CPython thread-safe. As a result no two threads, within the same process, can execute bytecode at the same time, and as you can imagine – calculations consist almost entirely of executing bytecode. And the effect is what you’ve seen above (and can reproduce on your computers) – no real difference between running computation in sequence or threads.
Luckily enough CPython developers were not all bad and put most (yes, most, not all) blocking operations outside of GIL. So you can still benefit by putting long-running blocking functions in their separate threads. You can, but you will be better of putting them in separate processes, which, admittedly, is not as easy to do as threading is well worth it. Not only for the performance, but it will also force you to put much more thought into the design of your code, and ensures that every bit of code observes the rules of encapsulation.

Some of the credit goes to [David Beazley][dabaz] for his presentation on this subject way back in 2009.

[dabaz]: http://www.dabeaz.com/
