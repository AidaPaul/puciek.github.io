---
layout: post
title:  "Tutorial: Writing stateful applications with Python"
date:   2014-11-02 12:15:0
---

The typical evening/morning (depends on readers interpretation of 4am) consists of me working through mountain of outstanding tickets, with breaks taken by browsing the good ole stack overflow. Most of it consists of down-voting and copy/pasting in one of my “This isn’t a free-coding website, show us your work” templates. But tonight I’ve run into a truly great question – where the user wants to loop his application until user quits, and have some conditions depending on his choices.

<!--more-->

I can see you scratching your heads, and I heard distinct “What is so great about that?” questions echoing. It is great because I clearly see that he is trying to resolve the case, put some effort into it and is in need of a guidance. And this is for whom stack overflow was created, people who develop software, not lazy people who cannot be bothered to google their problem. So with that, go back to his question and up-vote it, we need more of that in the ongoing homework crisis. Now, to the problem at hand.

The issue here is a very common one – he needs to allow the user to execute many actions, without the need of restarting application, and preserve information about what the user has done so far. In the professional terms -, he needs to abandon his stateless approach, and go stateful. What are those states about? In stateless application, it doesn’t matter what the user has done before and in stateful one the context matters. And since it matters, we need a way to preserve it in some form or shape. And when it comes to tasteful, there is no better friend than object-oriented paradigms. So let’s do some Voodoo, that it does so well.

Let’s start with complete listing of my suggested solution, and then go over explaining why and what.

{% highlight python %}
current_state = None
 
class Action():
    label = ""
 
    def __init__(self):
        raise Exception("Not implemented!")
 
class LoadFile(Action):
    label = "Load me this lovely file!"
 
    def __init__(self):
        current_state = FileLoaded() # Here you do your loading, and if successfull - switch the state
 
class ModifyFile(Action):
    label = "Modify me!"
 
class State():
    menu_options = {}
 
    def __init__(self):
        raise Exception("No actions available for this state")
 
    def print_menu(self): # This function returns printable menu based on menu_options
        for key, item in self.menu_options.iteritems():
            print("%s: %s" % (key, item.label))
 
    def execute_action(self, action_id):
        self.menu_options[action_id]()
 
class Launch(State):
    def __init__(self):
        self.menu_options[1] = LoadFile;
 
class FileLoaded(State):
    def __init__(self):
        self.menu_options[1] = ModifyFile
 
if __name__ == "__main__":
    current_state = Launch()
    try:
        while True:
            current_state.print_menu()
            user_choice = raw_input("What will be your pleasure?")
            current_state.execute_action(int(user_choice))
    except Exception as e:
        print("Boo boo happened: %s" % e)
{% endhighlight %}

It’s pretty simple and slick, when ran it will allow you to navigate through the two options before it crashes with “Not implemented.” exception. All the actions extend our abstract class Action.

{% highlight python %}
class Action():
    label = ""
 
    def __init__(self):
        raise Exception("Not implemented!")
{% endhighlight %}

Since python doesn’t natively support the idea of abstract classes, we are enforcing it by throwing an Exception in __init__. It will also serve as a good reminder that you didn’t define what your action is trying to do. Which is quite common, and expected, error when you follow the TDD methodology, so this is a win-win situation. Now let’s define our two actions.

{% highlight python %}
class LoadFile(Action):
    label = "Load me this lovely file!"
 
    def __init__(self):
        current_state = FileLoaded() # Here you do your loading, and if successfull - switch the state
 
class ModifyFile(Action):
    label = "Modify me!"
{% endhighlight %}

Since it is a mock code, there is not much here in terms of implementation. What is important to notice here is that while we’ve changed the labels for both, but didn’t define ModifyFile. As a result when we pick ModifyFile in the application as it is, it will throw an Exception. Next, let’s have a look at our context – or as some call its state, of the application.

{% highlight python %}
class State():
    menu_options = {}
 
    def __init__(self):
        raise Exception("No actions available for this state")
 
    def print_menu(self): # This function returns printable menu based on menu_options
        for key, item in self.menu_options.iteritems():
            print("%s: %s" % (key, item.label))
 
    def execute_action(self, action_id):
        self.menu_options[action_id]()
{% endhighlight %}

Once again, we are employing that nifty trick with Exception in __init__. I repeat it because I urge all of you to employ it, not only in abstract classes, but also in place of a virtual method. Or any other function that you didn’t implement yet – but they are supposed to be in the final product. Make this your new knee-jerk reaction.
But back on the subject, here we are sporting a class variable (or member if you prefer penile jokes) which is a dictionary of available actions and their keywords. Then there is, rather self-explanatory, print_menu, followed by nifty execute_action.
It is worth explaining why are we wrapping one line of code with a function, which is a valid concern. And the answer is very simple, and one I keep on beating every software developer with – encapsulation. I cannot stress enough how important it is to encapsulate every bit and piece of your code. Because when you do that, then from a quick glance at its declaration or use, you know exactly what is going on there. You don’t have to read up comments; you don’t have to think about the context – it is just instantly clear what is going on. And in this day and age, wherein one year we develop more software that we did in last three, saving every bit of time is of the essence.
Moving on, since we have our generic state, let’s implement our example states.

{% highlight python %}
class Launch(State):
    def __init__(self):
        self.menu_options[1] = LoadFile;
 
class FileLoaded(State):
    def __init__(self):
        self.menu_options[1] = ModifyFile
{% endhighlight %}
        
Not much to see here either, all we are doing is assigning some functions to the menu_options. And this leads us to our __main__.
       
{% highlight python %}
if __name__ == "__main__":
    current_state = Launch()
    try:
        while True:
            current_state.print_menu()
            user_choice = raw_input("What will be your pleasure?")
            current_state.execute_action(int(user_choice))
    except Exception as e:
        print("Boo boo happened: %s" % e)
{% endhighlight %}
        
Ain’t that pretty? We define out first step, and the run an infinitive loop that keep on nagging the user for his input. And we will keep on doing that till it either crashes or one of the called functions will run call the quits and kill the process.

And this is how we’ve achieved the unachievable. Climbed the unclimbable. Well, not, it’s fairly basic. But by applying proper object-oriented practices, combined with encapsulation, we have a nice base to build any type of a console application we want/need. And you know what? It isn’t only for console applications, as you could take this code written in that format and, quite easily, change the IO to anything you need. And I mean anything, from some graphical interface to sockets.

Thanks for reading through, and if you’ve learned something, remember to subscribe. And if you’ve any questions, feel free to leave them in the comments section.