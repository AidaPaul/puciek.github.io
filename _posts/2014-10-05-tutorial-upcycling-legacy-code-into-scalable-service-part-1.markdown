---
layout: post
title:  "Tutorial: Upcycling legacy code into scalable service part 1"
date:   2014-10-05 12:15:0
---

Dealing with legacy (accepted euphemism for manure) code is an integral part of every company that develops software. You cannot escape it; you cannot avoid it, you can only roughly manage the level of it in the codebase. And because of that, every piece of code eventually gets to the point where it has to be up-cycled so it will meet with the new business requirements. What it, usually, means is that it needs to be faster, better and stronger. And this is what we will do today. We will start by looking at some awful code, analyze the shortcomings, figure out how to overcome them and, eventually, turn this piece of legacy into a scalable service.

<!--more-->

Let’s start with a code sample. Initially, I’ve planned to write something bad, but then my luck turned, and I saw `py_mail_check.py` in my `~/misc`. Just by the name I knew it was good, and cat didn’t disappoint. Let’s have a look

{% highlight python %}
import re
import poplib
from socket import error as socket_error
 
f = open('mail_list')
 
good = []
bad = []
 
for line in f:
    if len(line) > 10:
        dd = re.match('(.+@(.+));(.+)', line)
        try:
            bs = "%s:%s:%s\n" % (dd.group(1), dd.group(3), dd.group(2))
            mb = poplib.POP3_SSL(dd.group(2), 995)
            mb.user(dd.group(1))
            mb.pass_(dd.group(3))
            good.append(bs)
        except (poplib.error_proto, socket_error) as e:
            bad.append(bs)
 
fig = open('good', 'a')
big = open('bad', 'a')
 
for item in good:
    fig.write(item)
 
for item in bad:
    big.write(item)
 
fig.close()
big.close()
{% endhighlight %}

What a mess it is, and I do not mean it in just one way. Luckily enough it came bundled with the mail_list file:

{% highlight text %}
abc@gmail.com;def
foo@bar;foobar
{% endhighlight %}
So let’s start from the top. This code seems to take an oddly-formatted file, process every line of it for email details and try to login to that email address. If the credentials are ok, we save the result to file “good”, or if they are wrong then to file “bad” instead. A simple concept but the implementation is lacking, and it creates quite a few issues for itself:
1. We are building two, quite sizeable, lists that hold results of the run. While this will work for small scale, when exposed to couple hundred thousand emails to process it will explode. For a simple reason – it will run out of memory.
2. We are processing one line at a time. Due to the nature of the task, checking whether we can login to specific email accounts, it will spend a lot of time waiting to hear back from the socket. That will cause the one thread running to spend most of its life in idle – not good.
3. The style of coding is non-existing. Variable names are a complete mess, and author wrote it through spaghetti stringer. While using OOP here would be just silly, breaking the logic into a function would improve the readability a lot.

Those are all major issues, and we will be fixing them all very soon. But before we get to that let’s fix one issue I didn’t mention, but is the most important one – writing a test. If your code doesn’t come with a suite of tests then how will you know whether it’s working? And while tests do not necessarily have to be automated, there is nothing wrong with going over a checklist, they have to be there. And in written form, not in your head.
To write a good test you first need to define what are the use cases and their expected results. Here we have two:

1. We have provided correct email credentials. In that case the details should be moved to the “good” file.
2. We have provided wrong email credentials. In that case the details should be moved to the “bad” file.

The flow of such test is fairly simple:

1. Replace the mail_list with a pre-made one for debugging. It should have two lines in it – one with good email details, and the other with wrong details. You can use more if that pleases you.
2. Run the python script.
3. Check the content of the “good” and “bad” whether right lines went into correct files.

Writing automation for this checklist, I will leave to each reader, consider this as blog-homework that won’t be graded but is a good exercise!

With test ready we can finally touch the code. I want to embed the habit of writing a test before any code into your brain as it will benefit your greatly in the future. The very first thing we should do is to fix the issue that may crash our application – the two massive lists.
As it is now, the script buffers the results of its work before writing them down to file. Quite frankly the buffering part here doesn’t make any sense (and it rarely does) so we should just get rid of it and replace it with a direct write to the right file.

{% highlight python %}
import re
import poplib
from socket import error as socket_error
 
f = open('mail_list')
 
for line in f:
    if len(line) > 10:
        dd = re.match('(.+@(.+));(.+)', line)
        try:
            bs = "%s:%s:%s\n" % (dd.group(1), dd.group(3), dd.group(2))
            mb = poplib.POP3_SSL(dd.group(2), 995)
            mb.user(dd.group(1))
            mb.pass_(dd.group(3))
            fig = open('good', 'a')
            fig.write(bs)
            fig.close()
        except (poplib.error_proto, socket_error) as e:
            big = open('bad', 'a')
            big.write(bs)
            big.close()
{% endhighlight %}

Now, with the danger of crashing already out of the way lets refactor this code into something more readable. The very first thing to do will be to move all the logic of processing a line into a function. Since it will be checking emails, we will call it check_email. While doing so, I took the liberty of moving definition of bs outside of try; catch block and refactoring the variable names to something sensible.

{% highlight python %}
import re
import poplib
from socket import error as socket_error
 
source_file = open('mail_list')
 
def check_email(username, password, host, port=995):
    details_string = "%s:%s:%s\n" % (username, password, host)
    try:
        pop3_connection = poplib.POP3_SSL(host, port)
        pop3_connection.user(username)
        pop3_connection.pass_(password)
        good_results = open('good', 'a')
        good_results.write(details_string)
        good_results.close()
    except (poplib.error_proto, socket_error) as e:
        bad_results = open('bad', 'a')
        bad_results.write(details_string)
        bad_results.close()
 
for line in source_file:
    if len(line) > 10:
        results = re.match('(.+@(.+));(.+)', line)
        check_email(results.group(1), results.group(3), results.group(2))

{% endhighlight %}

Isn’t that pretty? In couple easy steps, we have transformed pile of manure into something that is not only works, but is also easy to understand and comes with automated test to ensure that it’s working as intended. But we are not done yet as there is an outstanding ticket left – making it work with more than one line at a time.

While python comes with many excellent options for parallelizations, in case of this script I find multiprocessing.Process to be most suitable. For many reasons, but most important ones will be that we don’t care about controlling amount of workers, or need to exchange any data between them. It is also easiest one to implement – just have a look:

{% highlight python %}
import re
import poplib
import multiprocessing
from socket import error as socket_error
 
source_file = open('mail_list')
 
def check_email(username, password, host, port=995):
    details_string = "%s:%s:%s\n" % (username, password, host)
    try:
        pop3_connection = poplib.POP3_SSL(host, port)
        pop3_connection.user(username)
        pop3_connection.pass_(password)
        good_results = open('good', 'a')
        good_results.write(details_string)
        good_results.close()
    except (poplib.error_proto, socket_error) as e:
        bad_results = open('bad', 'a')
        bad_results.write(details_string)
        bad_results.close()
 
for line in source_file:
    if len(line) > 10:
        results = re.match('(.+@(.+));(.+)', line)
        process = multiprocessing.Process(target=check_email, args=(results.group(1), results.group(3), results.group(2)))
        process.start()
{% endhighlight %}

That is all there is to it – two lines of code, and now you will be utilizing every piece of CPU that is there to spare. The way it runs it will create processes as fast as possible, so keep that in mind as with large volumes amount of emails to check it may bring loads beyond what is sane. But don’t worry, having thousands of processes like that will, most likely, not cause any instability contrary to what some people say. But if you are concerned about it, feel free to replace the launcher with more controllable solution. A quick hack would be put a sleep() after starting each process.

And that is it for the first part of the article. While it may not seem like we’ve done much, we did quite a lot. Becuase the process you’ve seen here can be applied not only to small random scripts. By applying it, you can chop down and rework any piece of code you will find, including large spaghetti monsters that plague large systems. All you have to remember is to keep on breaking it down into functions, and writing tests before you do so. Small steps get you far.

In the next part of the, up-cycling series, we will convert this script into a clog of a large machine that is messaging system. We will also, with the help of behave, put our tests into much more structured manner and see how, as a result of all this, maintenance and expansion suddenly became very easy.
