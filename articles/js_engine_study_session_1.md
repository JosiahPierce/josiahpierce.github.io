Title: JavaScript Engine Exploitation: Study Session 1
Date: 01-09-2021
Tags: study,javascript_engine,exploitation
Author: Josiah Pierce
Summary: Part 1 of a series of writeups on resources for learning JavaScript Engine Exploitation

To continue this series of summaries of various resources on JS engine exploitation, I decided to watch one of many outstanding talks by [Natalie Silvanovich](https://twitter.com/natashenka) of Project Zero. She has multiple talks on attacking ECMAScript engines, but I started with this one entitled "Attacking ECMAScript Engines with Redefinition":
<iframe width="560" height="315" src="https://www.youtube.com/embed/KT2vYWcwj-w" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

As with all posts in this series, please keep in mind that this is only a summary of the points I found most useful. I highly recommend following along with the video to see all the examples and get full context.

#### Technical details

In the context of the presentation, "redefinition" means changing something with an understood meaning in ECMAScript standards to mean something else. For example, in the first code snippet of the presentation, the function `alert()` is redefined to be equal to a different function, as seen in the following line:
`alert = f`

In the case illustrated in the presentation, calling `alert()` now calls the function `f()`, rather than the built-in `alert()` function most of us are probably familiar with that creates a pop-up message.

However, in some cases, there's also the possiblity that the redefintion doesn't override the original definition -- meaning the original definition gets used anyway. The final side case is that execution just fails entirely, meaning that neither the original nor the updated definition will be used.

One way to perform redefinition, as seen in that first example, is to use the equality operator "=".  In JavaScript specifically, Natalie mentions that one native method can be redefined as another native method, but this can't be done directly. Instead, a wrapper needs to be placed around the second native method (as seen where `alert()` is redefined to `f()`, which in the example was just a wrapper around `document.write()` -- note that the example isn't simply setting `alert()` to `document.write()` directly).

The first example that the talk provides of abusing redefinition with the equality operator is to set up a type confusion. After instantiating an object, the constructor for the object is then redefined to be a different constructor. If my understanding is correct, later on in the code, some content from that object gets copied into a temporary array and a new object is created to point to it. However, the new object is created using the redefined constructor. Since it's been redefined, the code thinks it's using the right constructor, but really it's using a different one that will create a different object, leading to a type confusion.

Basically, the simplified idea here seems to be to find some constructor, redefine it to be a different constructor of some useful type, and then call a function that will create an object using the original constructor without realizing that the constructor has been redefined. This means it'll instantiate an object of a type other than the one it was expecting.

Natalie then covers another way to perform redefinition -- proxy objects. Proxy objects could be leveraged to change the methods within an object that end up being called, or change the results that are provided based on how many times the proxy object has been enumerated before. This concept is only briefly covered, but it's inspired me to do more research into Proxy objects and find some good example bugs. Here's one by Saelo from Pwn2Own 2018 that uses a Proxy object to create side effects that the JIT compiler doesn't consider:

[https://github.com/saelo/pwn2own2018#stage-0](https://github.com/saelo/pwn2own2018#stage-0)

Yet another redefinition idea is to use conversion operators such as valueOf and toString. The example I immediately thought of for using valueOf was this paper by Saelo, which covers a bug in JavaScriptCore (JSC): 
[http://phrack.org/papers/attacking_javascript_engines.html](http://phrack.org/papers/attacking_javascript_engines.html)

Basically, when an object is converted to an integer, if the object has a valueOf property, its return value can be used as the result of the integer conversion. The valueOf property can contain an inline function that peforms additional operations besides just returning an integer; for example, it could change the length property of an array, causing an unexpected side effect (definitely check out Saelo's Phrack paper I linked above; it's got a great illustration of this for getting OOB array access).

There's a discussion of “watches”, which sound sort of like getters/setters. I'll skip over this portion of the talk, because at least in Firefox, it appears that watches have been deprecated and removed, according to this documentation: [https://developer.mozilla.org/en-US/docs/Archive/Web/JavaScript/Object.watch](https://developer.mozilla.org/en-US/docs/Archive/Web/JavaScript/Object.watch)

There's also some discussion of some methods that could theoretically be used for redefinition attacks, but didn't have specific bug examples. Since this talk is from 2015, I'll have to keep these in mind and research whether any of them did indeed end up being possible to leverage for redefinition attacks. 

Finally, there's a brief overview of how to hunt for these types of bugs. I'm primarily interested in finding bugs through source code review, so I'll ignore the portions on fuzzing and reverse engineering (as the targets I'm interested in attacking are open source).

The two methods that are relevant to my interests are code review and API docs. For code review, Natalie says to hunt for functions that peform calls that allow script execution, and then look for vulnerabilities there. If API docs are available, she suggests looking for functions that take objects or arrays as parameters, because the objects or elements in the array need to be converted to another type, which might in turn mean that calls to valueOf or toString will be performed.

#### Conclusion

This talk provided a nice overview of some methods of performing redefinition attacks. In particular, it's encouraged me to look into using Proxy objects for exploitation, which is something I was vaguely aware of prior to watching this talk, but had never investigated.

