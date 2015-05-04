---
layout: post
title:  "Bad technical interviews – the plague of modern IT"
date:   2014-10-26 12:15:0
---

Interview. The part of hiring people that is invaluable and yet, done oh so wrong by most of the companies. I can’t count how many companies I’ve spoken with, only to be turned off after first five to ten minutes of it. Last week was one of those, where I gave up on the company in short 10 minutes after walking through the doors.

<!--more-->

I went to a meeting with this particular company after exchanging some emails and phone calls – we both smelled each other and liked it so far. Even more than that, stars were aligned, and this was one of those rare moments where both sides seem to match. So I don’t have to say that I went with particular jolly and optimism. Sadly that was blown quickly, when after very short introduction the interviewer turned his macbook across the table and started to hover over my shoulder, explaining the test.
Test was very fair and simple – there is a file, where each line has four sets of numbers separated by a tabulator. Size of the file was rather pitiful – 320kb, so clearly performance wasn’t an issue here. All the guidelines provided for this task were:
– We want to tally up all the numbers in the file
That is it, as I said – simple and sound test. But as always the devil is in details.
As I sat there, staring at python interpreter nagging for my input, I asked can I use VIM instead. “No,” was the short, yet heart-breaking answer, I replied “Are you serious?” but the answer was silence. So I sat down to write down the code, and be quite verbal about what I am doing.
You see, the task is very simple – you need to run a loop through each line, strip the end of each line of the new-line mess and then split the result by “tab”. Then just sum the content of that particular line and add the result to the global variable called total. There is nothing complex here, expect the unfriendly environment.

And that kicked me in the butt right away – “import csv” yielding and error 404, not found. Oh well, not a biggie, I drop down to console and start defining functions to do appropriate parts of code. Quickly sum_items was conceived that took an iterable object, iterated over its content and returned the sum. Making a typo is common, especially when you try to use proper variable names, not “zcv” just to keep it short, I had to retype the function in multiple times. Just because of the typing errors. Not to mention that when working with python interpreter, there is no auto indentation – enjoy counting the spaces you press!
But finally we got through it, and the function was working fine. I have to say that I already gave up on the company by now; I will explain later why in detail, so I decided to test the tester. To launch the processing, I put out this nasty, and obviously wrong, loop:

{% highlight python %}
source_file = open("data", "r")
for line in source_file.readline():
  print(line)
{% endhighlight %}

The beauty of it is that it’s deceiving. It makes perfect sense to assume that read_line() will return you a line each time you iterate over it. It, of course, is a wrong assumption, but, as a result, it is a very nasty “trick” question. And it worked perfectly – and I enjoyed the look on interviewers face when he didn’t know what was going on, in his test. And my asking “are you sure that this isn’t a trick question?” wasn’t helping. To make a long story short, eventually, after many lines re-typed, I have fixed the code, and it ran properly, giving us the tally. The interviewer rated it couple times. The ratings ranged from “not terrible” to “mixed”; my internal was “get me out of here”.
I can hear a lot of you saying “you just can’t write software that is why you complain about it, and pick on codility”. But that is not the case as I have written at least two pieces of code that handle files. Why I hate, this test is because it puts a developer in an environment where the best code, is a bad code, and it will do you a lot of good to ignore best coding practices. Let’s look into that.

A bad programmer in this spot would start with writing the line loop, and then would write a very nasty one-liner that handles the splitting, counting and adding it back to the total number. While this indeed would work, and can be written quickly, it is not a piece of code I would signature or use. It’s hard to maintain, modify and even read as you write it, now think about in 6 months time. Think about it – when you are looking at a one-liner, you have to think couple times to understand what is it trying to do, while in properly structured code – you know it at a glance. But this is a programming-related blog, so let’s get some code in, starting with what a bad programmer would write.

{% highlight python %}
f = open("data")
total = 0
for l in f:
  total += sum([int(num) for num in l.strip("\n").split("\t")])
print(total)
{% endhighlight %}

It sure as hell works, but I need a decoder ring to understand what is going in here, and I don’t eat enough cereal to own one. Code like that never passes my code review and is sent back to the author, coupled with some choice words. Repeated offenders also get a lecture. But it got one very convenient thing going for it – it is very debugger-friendly. It is easy to write it in a console, and it’s almost as easy to maintain it in a console, so it fits the test perfectly. And now let’s compare it with my solution.

{% highlight python %}
def sum_items(numbers):
  sum = 0
  for number in numbers:
    sum += int(number)
  return sum
 
def parse_line(line):
  items = []
  trimmed_line = line.strip("\n")
  items = trimmed_line.split("\t")
  return items
 
source_file = open("data", "r")
total = 0
for raw_line in source_file:
  total += sum_items(parse_line(raw_line))
print(total)
{% endhighlight %}

I admit that it’s not perfect, but how much can you expect from quick interview code, that is constrained to bare-debugger as your environment. No fancy tools like code suggestion, tab-completion and the high-end features like :12i (for non-vim people, this jumps to line 12 and starts insert mode). And without them writing the proper code, that is broken into functions, uses nice variable names and is, easy to read and maintain, becomes a living hell. Don’t’ believe me? Try to do it yourself and take a note of how many times you had to rewrite large chunks of code. And don’t forget to put a time constraint of 5 minutes on the whole task, just to add some fun to it.

And the lesson from that is to the interviewers out there. Testing a programmer during an interview is essential. If they do not write even smallest piece of code during it then, you may as well not be interviewing at all. But when you do, use some common sense. Don’t force them to slave over debugger – give them nice IDE. Don’t cut off essential libraries just to make it harder, because if his first reaction here was to use the csv library – he clearly faced similar problem before. If you will stick to your silly tests, coupled with questions in the sort of “guess what keyword means” (more on that in other blog entry), will scare away all the good candidates. But they will be a breeze for all the fakes and bad programmers, the ones after which you will be cleaning years after they have left your company.
