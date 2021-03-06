Title: JavaScript Engine Exploitation: Study Session 0
Date: 01-01-2021
Tags: study,javascript_engine,exploitation
Author: Josiah Pierce
Summary: Part 0 of a series of writeups on resources for learning JavaScript Engine Exploitation

I've been interested in JavaScript engine exploitation and browser exploitation for a while now. While I've spent some time studying this topic and performing vulnerability research on JS engines on and off, I feel I haven't really applied myself to focused, consistent study. As a result, I'll occasionally spend a weekend  poring over some resources and auditing a little code, then take a long break and promptly forget most of what I learned. 

In an attempt to remediate that behavior, I'm kicking this blog off with a series of "study sessions" on various JavaScript engine exploitation writeups and presentations. Every two weeks, I plan to spend time reading or watching a resource on JS engine exploitation or internals and then writing up a summary of the resource and what I learned. Ideally, this will encourage me to consistently engage with new material and understand it well enough to be able to explain some of the key points here. I'll continue this series until I feel I've consumed and summarized a reasonable corpus of learning resources.

Hopefully, other readers will find these summaries valuable as quick references and as encouragement to study the full versions of these resources themselves.

There's certainly no shortage of resources to choose from; if you'd like to find some to study for yourself, there's an excellent list here:
[https://zon8.re/posts/javascript-engine-fuzzing-and-exploitation-reading-list/](https://zon8.re/posts/javascript-engine-fuzzing-and-exploitation-reading-list/)

I'm primarily interested in attacking Firefox, which uses the Spidermonkey JS engine. It's where I've spent the most time, and most of the resources I cover will be at least partially aimed at Firefox. However, I also have an interest in Chrome (which uses the V8 JS engine). Some resources are also generic enough that even if they cover a different JS engine, they're still valuable and offer useful insight into general vulnerability research or exploitation methods.

For this first study session, I took a look at this excellent talk by [@maxpl0it](https://twitter.com/maxpl0it) entitled "Actions Speak Browser Than Words (Exploiting n-days for fun and profit)":
<iframe width="560" height="315" src="https://www.youtube.com/embed/L7aiFKDg0Jk" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

The talk is aimed at folks new to browser exploitation. Writing n-day exploits based on CVE descriptions or other details is often mentioned as a good way to hone one's exploit development skills, so this talk seemed like a good starting point. The first bug the talk covers is a bug in Internet Explorer (which uses the JScript JS engine), and the second bug is in Firefox. Since I'm less interested in IE, I'll still cover general takeaways from that section, but won't cover as many IE-specific technical details.

In an effort to avoid simply reproducing the entire contents of the study resource in each post for this series, I'll cover the portions I found most interesting or useful, and reproduce code where necessary. If you want more context, be sure to check out the resource for yourself.

#### Non-technical takeaways

There were two particularly useful takeaways in this talk that weren't related to specific technical details:

-People are often intimidated by large, complex targets because they feel they need to learn all of the internals of the target before they can hunt for bugs. Max mentions that it's possible to proficiently bug hunt without needing to know everything about a target's internals; knowing 40% of the internals could be enough. He also mentions that it becomes easier to expand on your knowledge once you have an understanding of some of the internals. 

-Both of the analyzed vulnerabilities were variants of older bugs. This reinforces something I've heard lots of vulnerability researchers say, which is that a good method for finding new bugs is to look at locations which have had lots of bugs in the past and search for slight variations on those bugs. Sometimes old bugs can even be reintroduced through some regression years after the original issues were fixed. To me, this stresses that it's important to study bugs in components you're interested in attacking, since it helps with pattern recognition. I imagine it's also helpful to know the general exploitation algorithms for previous bugs, since they may largely apply to new bugs that are variants on older ones.

#### Technical details

**The IE bug**

The IE vulnerability was a use-after-free (UAF) due to improper garbage collection. To track down the specific patch that corrected the vulnerability, Max performed binary diffing to identify changed locations. Upon identifying a specific object and function call that appeared to be added in the patched version, he ran a proof-of-concept code snippet on the patched version and used a debugger to skip over the function he believed had been added to patch the issue. Skipping over the function caused a crash, verifying that this was indeed the patch for the issue.

The idea of just using a debugger to skip over suspected patch code sounds obvious to me now, but I'd actually never used that method before. It's a cool trick to be aware of; it helps provide an option for dynamic analysis if static binary diffing alone isn't enough to be certain whether the patch has been identified.

Since I'm not super interested in attacking IE myself, I haven't spent further time digging into the JScript internals. However, the general exploitation steps for obtaining some useful primitives are worth noting. My (potentially wrong) understanding of the bug is that the garbage collector doesn't correctly track arguments passed to the `Array.prototype.sort()` function. Therefore, the garbage collector doesn't realize that objects referenced by those arguments are still in use. It goes ahead and `free()`s the objects, meaning that the arguments are now stale pointers. Any deferencing of those stale pointers should now trigger the UAF.

The first general exploitation steps are to create lots of these untracked arguments, then manually call the garbage collector (which is possible in JScript). This causes the arguments to become stale pointers, since the objects they were pointing to were freed. Then new allocations are peformed that should re-use the available space the free objects were occupying. By allocating something of a different type than the original objects the arguments pointed to, it's possible to construct a type confusion bug.

From that point forward, grasping the JScript internals seems to be more important for following along with the rest of the exploit. However, I thought just seeing those first steps broken down was helpful. I haven't looked at a browser bug related to garbage collection before, so this was a nice intro.

**The Firefox bug**

The Firefox vulnerability was a bug in the just in time (JIT) compiler. JIT compilers seem to be one of the most popular components for vulnerability researchers to audit these days (understandably, since they're extremely complex and therefore provide fertile ground for bugs). I'm generally more interested in bugs that are purely in the primary JS engine and not the JIT compiler (especially since not all JS engines feature JIT compilers at all -- smaller engines such as ones used on embedded devices may not have them, for example). However, I thought this was a good introduction to JIT bugs, and covers a nice method of leveraging a JS setter while bypassing a check to ensure an argument is an integer.

Like most JIT bugs I've seen, this bug was due to optimizing out a specific check while failing to account for potential side effects. You should really check out the presentation if you want to see the full details, but the part that was most interesting to me was the method of triggering the side effect.

A common method I've seen in JS engine exploitation is to use a setter (code that's called when a property is assigned) or getter (code that's called when a property is accessed) to run some code and make an unexpected change. For example, a getter might change the length of an array during the execution of a loop, causing the loop to go out-of-bounds.

For example, here's a snippet of code taken from the presentation showing a setter being created for the property "a" for an array:

```
let arr = [1,2,3]
arr.__defineSetter__("a",function(x){
    console.log("Reached!");
```


In this example, an array is created, and then a setter is created for that array. If the property "a" is assigned to the array, the inlined function called "x" will be run, which in this case just prints the message "Reached!" to the console. However, it could do something else, like changing the length of the array, which might cause out-of-bounds access during the execution of whatever function triggered this setter (spoilers: this is exactly what the exploit does).

However, if my understanding is correct, one additional wrinkle for this bug was that to trigger the vulnerable JIT node, an index of type int32 was required. However, the setter needed to be triggered on a property, not an element. Therefore, the chosen argument for the function was -1, which is still an int32, but won't be considered an element in an array (because it's a negative integer); instead, -1 will be considered a *property*, triggering the setter. It's a clever method of working around several constraints.

#### Conclusion

Even though this talk covered areas that aren't my primary interest (JScript and JIT bugs), I still felt it was well worth the time investment to watch. There were good technical and non-technical takeaways, and it was nice to start with a resource aimed at newcomers to browser exploitation.

