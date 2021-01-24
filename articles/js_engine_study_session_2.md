Title: JavaScript Engine Exploitation: Study Session 2
Date: 01-24-2021
Tags: study,javascript_engine,exploitation
Author: Josiah Pierce
Summary: Part 2 of a series of writeups on resources for learning JavaScript Engine Exploitation

For this post, I chose to summarize some of the details from the following 2017 talk entitled "The ECMA And The Chakra":

<iframe width="560" height="315" src="https://www.youtube.com/embed/P4NI4qYWqes" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

This is another talk by [Natalie Silvanovich](https://twitter.com/natashenka). This talk focuses on how new implementations of ECMAScript standards can be vulnerability-prone, and uses the now-moribund Chakra (the JS engine formerly used by Microsoft's Edge browser) as an example. While the engine may be a bit less relevant for security research these days, identifying bugs in newly-implemented ECMAScript standards is certainly still relevant.

As with all posts in this series, please keep in mind that this is only a summary of the points I found most useful. I highly recommend following along with the video to see all the examples and get full context.

#### Technical details

The first chunk of the talk covers several different key features of ECMAScript engine design. This section served to highlight how complex some features can really be; for example, arrays can hold elements of seemingly any type, and can be laid out in memory differently depending on whether they're “sparse” or “dense”. In the talk, the following example was provided for creating a sparse array:

`d[100000] = 7;`

Reserving memory for 100000 elements would be wasteful if the array currently only holds a single element. Therefore, the memory for all those elements isn't actually allocated for a sparse array. The gaps where elements could be but aren't currently allocated are called “holes”. In contrast, a “dense” array is just an array without holes (an array with contiguous elements).

Some of the more esoteric features include the ability to configure arrays -- for example, making an array read-only, or making a specific element of an array read-only. The code example provided in the talk for configuring an array is as follows:

`Object.defineOwnProperty(a, “0”, {value: 1, writable: false});`

The above line configures the array such that element 0 of the array is read-only. There's also the Object.freeze() function, which can make an entire array read-only.

Even more interesting is the ability to create a getter or setter (code that runs when something is accessed or given a value) on specific array indexes. Here's the example provided in the talk of creating a getter and setter for the first element of array “a”:

`Object.defineOwnProperty(a, “0”, {get : func, set : func});`

There's also some discussion of the array prototype chain. An array object has a prototype, but it also inherits methods from Array.prototype, which in turn inherits from Object.prototype. Bizarrely, array prototypes themselves can have elements. Once again, here's a code snippet from the talk illustrating this behavior:

`a.__proto__ = [];`

`Object.defineProperty( a.__proto__, "0", {get : func, set : func});`

All this serves to reinforce that there are a lot of lesser-known features of major script engine elements like arrays. I'd be curious to learn if someone's compiled a public list somewhere of these features and instances of using them to trigger vulnerabilities; there are so many that it seems overwhelming to discover and remember them all (which just underlines how hard securing JS engines must be).

Another helpful portion of the talk describes some ECMAScript engine terminology -- specifcally, the terms “fast path” and “slow path”. Essentially, “fast path” behavior is a more optimized behavior that gets leveraged when an object is in a common state, whereas “slow path” behavior is less optimized, but intended to be safer for handling objects that are in less common states.

The talk also covers some bugs in Chakra based on some of the concepts described. If you want the specific details of each bug, I recommend watching the talk -- for this synopsis, I'll just summarize a few of the bugs so I can refer back to this for issues to look out for / techniques to know when auditing engines.

-Infoleak in the Array.join function by using Object.defineProperty to set a getter on an array index. This getter changed the type of the array by placing an object into one of its indices. Chakra implements type switching for arrays based on the data they hold, so adding an object to an array that was previously composed of integers would change its type. A takeaway here might be to look into how a JS engine handles switching the types of an array. If functions only check the type of the array they've been passed initially, it might be possible to change the type using an array index getter/setter.

-Similarly, an overflow was possible in the array spread operator by using an array index getter/setter (another term for these appears to be “array index interceptor”). In this case, Chakra actually implemented a check in the spread operator code to determine if the array length had changed. However, it then performed a loop that fetched the length of the array in each iteration. Within that loop, a function was called that could fetch a specific array element; if that element featured a getter or setter, then a callback could occur that would change the length of the array. Since the loop itself fetched the length on each iteration, the initial check to see if the length had changed would miss this. I'm curious if this issue would've been avoided had the length only been fetched once. 

-There's yet another bug involving array index getters/setters in the Array.reverse function. Clearly, array index getters/setters are difficult to account for. It's also stated in the talk that array reversing is generally really hard to get right, so that sounds like something that'd be good to audit closely. 

Another interesting detail covered in the talk is that sometimes slow path functions are written in JavaScript, since that way it's easier to safely handle so many different potential cases. A potential risk associated with this approach is the potential for JS functions called in the slow path implementation to have been redefined, potentially changing what the slow path function actually does.

#### Conclusion

This talk covered quite a lot of ground very quickly, but I thought the most useful part for my puroses was the set of example bugs and their causes. In the future, I think I need to frequently consider whether an array index getter/setter could be used to subvert some native function, as that's clearly been a cause of problems in the past.
