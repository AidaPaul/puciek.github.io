---
layout: post
title:  "Introducing watchbuilder, the basic tool for automation, and a guide to using watchdog in python"
date:   2014-02-25 04:35:0
---

As I did mention in latest post about using Visual Studio for Linux development, I do a lot of low-level work, very often involving ASM and C. And while there is a lot of sophisticated software for new technologies, even C++, [ASM and C development seems unloved][asm-ide] to say the least (asmCharm, please!). Because of that, low-level developers write their own tools that harness the power of ssh, rsync, toolchain and all the coolness that is needed just to make the process a little less of a hassle. And that is how [watchbuilder was born][watchbuild-repo] and conceived, as a necessity.

<!--more-->

So what is it? It’s a very simple, 25 lines long python script that utilizes watchdog to build stuff for you at the very moment you save it. Without any further ado, [have a peek into the code][watchbuild-repo], and if you are a newbie keep on reading as I will explain what is going on line-by-line. It’s working with both lines of python, and only dependency is watchdog. To use it, just download the script, adjust the build conditions to fit your needs and run it in the top directory of your source code tree. That is it, when it’s running it will automatically do what you tell it to do after you save a file. If you did not understand exactly how is that happening or how to make those adjustments, keep on reading as I will go line-by-line explaining exactly how this small script works.

So, let’s start from the bottom, as most code should be read, in the end main() is always on the bottom (you can thank C++ for the fact that even in python it’s on the bottom, not on the top where it is logical to place it).

{% highlight python linenos=table %}
event_handler = AutoBuilder()
{% endhighlight %}
First we create a new instance of AutoBuilder class, so it seems like a good time to have a peek.

{% highlight python linenos=table %}
class WatchBuilder(FileSystemEventHandler):
{% endhighlight %}
First we define a new class called WatchBuilder that extends watchdog.FileSystemEventHandler class. This class holds methods that are called whenever there is a corresponding change in the filesystem. If we want to add some logic to them, so they will do as we want we have to overwrite a corresponding function, which we do.

{% highlight python linenos=table %}
def on_any_event(self, event):
{% endhighlight %}
While there are a few of them to pick from, in my case I go the lazy way and settle for on_any_event, which will be fired every time a change on filesystem is done. This is simple because of laziness, as I do need three of them (on_created, on_modified, on_moved) and the only downside is occasional compiler/linker error, which is which is unmistakeable with an actual error. As an argument, this function receives an instance of watchdog.event class, from which we can extract vital information about what happened. And what can be more important than getting name of the file that interests us? Nothing, so let’s get cracking!

{% highlight python linenos=table %}
filename = str(event.src_path.decode('utf-8'))
{% endhighlight %}
Here we extract absolute file path from occurring event. This is all that happens here, as the rest of it is just to maintain comparability between python 2 and 3. For the curious of why exactly we have to do all that, you can lookup how string handling changed in python 3.
Now that we have a name of the file, we can dig down and write out own logic to it. For me, I know that I need to use a different compiler for different files so the very first step will be to extract the extension.

{% highlight python linenos=table %}
extension = os.path.splitext(filename)[1]
{% endhighlight %}
This is one of the neat tricks on python.The os.path.splitext(string) will return you tuple, where second element is the text after last dot, and first one is everything before that last dot. So when we call that function, we know that, at index 1, we will have the file extension, which we assign to a variable. And with that, we can just do our little version of switch/case to see what we have to do with the file

{% highlight python linenos=table %}
if extension == '.nasm':
    os.system('nasm -f elf %s 2>&1' % filename)
elif extension == '.o':
    print('ld -m elf_i386 -s -o %s %s 2>&1' % (filename[:-2], filename))
    os.system('ld -m elf_i386 -s -o %s %s 2>&1 &> /dev/null' % (filename[:-2], filename))
elif extension == '.c':
    os.system('cc -mpreferred-stack-boundary=2 -fno-stack-protector -ggdb %s -o %s'
              % (filename, filename[:-2]))
    os.system('execstack -s %s' % filename[:-2])
{% endhighlight %}
I am not going to get into explaining all the compilation options, and why I use this and that. You will most likely not need them, and if you are curious google is just CTRL+T away ;). With that in mind, let’s go back to our __main__, which we left after declaring our event_handler.

{% highlight python linenos=table %}
observer = Observer()
observer.schedule(event_handler, BASEDIR, recursive=True)
{% endhighlight %}
After that, we create an instance of watchdog.Observer class and specify a schedule for it. What we provide it with an instance of a class is to be used to handle events, absolute path to the directory that we want to observe, and additional argument to watch the directory recursively. All that is left now is to start the observer and see it work.


{% highlight python linenos=table %}
observer.start() # Starts a loop over events, to quit just mash ctrl-c
{% endhighlight %}
And that is it. Since the watchdog uses threading extensively, adding to it a proper handle for sigint is not very trivial (and I do not want to make it a part of this article), let’s just say that in order to stop it you will have to press ctrl+c couple times. And here comes full listing for those who hate Github (probably mercurial users).

{% highlight python linenos=inline%}
__author__ = 'Tymoteusz Paul'
import os
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler
BASEDIR = os.path.abspath(os.path.dirname(__file__))
 
class WatchBuilder(FileSystemEventHandler):
    def on_any_event(self, event):
        filename = str(event.src_path.decode('utf-8')) # Ugly workaround but it will work with both 2.7 and 3.3
        extension = os.path.splitext(filename)[1]
        if extension == '.nasm':
            os.system('nasm -f elf %s 2>&1' % filename)
        elif extension == '.o':
            os.system('ld -m elf_i386 -s -o %s %s 2>&1 &> /dev/null' % (filename[:-2], filename))
        elif extension == '.c':
            os.system('cc -mpreferred-stack-boundary=2 -fno-stack-protector -ggdb %s -o %s'
                      % (filename, filename[:-2]))
            os.system('execstack -s %s' % filename[:-2])
 
if __name__ == '__main__':
    event_handler = WatchBuilder()
    observer = Observer()
    observer.schedule(event_handler, BASEDIR, recursive=True)
    observer.start() # Starts a loop over events, to quit just mash ctrl-c
{% endhighlight %}
And that is it. If you want to test it on existing compilation definitions, with observer running, just create something.nasm file and fill it with garbage. Once you will save it, you will see errors in observer output, either complaining that the file doesn’t make sense, or that nasm doesn’t exist.
As you’ve probably noticed, this may be used for much more than just building, but also, for example, launching automated tests against the new code, without the need of dropping down to console and launching that command manually. And I see programmers do it all the time, every time when they finish some part of the code that they cannot automate inside their IDE they drop down to console and run it manually. And that is expensive, not as much in time as it is in focus, as with this approach you can just put it on the second screen and never move focus away from code.

I hope that you guys enjoyed the article, please let me know what do you think in the comments. This was just a spur of the moment article, something to fill the place as I am working on three larger pieces. One about latest hit – stratosphere, other about basics of reverse engineering and a newbie guide to artificial neural networks so make sure to stay tuned!

[asm-ide]: http://diamond.kolibrios.org/hll/hll_nasm2.gif
[watchbuild-repo]: https://github.com/Puciek/watchbuilder