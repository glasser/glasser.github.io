---
layout: post
title:  "A surprising JavaScript memory leak"
date:   2013-06-27 13:38:06
---

This week, my teammates at [Meteor][meteor] tracked down a surprising
[JavaScript memory leak][issue]. I searched the web for variations on
`javascript closure memory leak` and came up with nothing relevant, so maybe
this is a relatively unknown issue. (Most of what you find for that query talks
about the bad garbage collection algorithm in old versions of IE, but this
problem affects even my current Chrome install!)

JavaScript is secretly a functional programming language, and its functions are
*closures*: function objects get access to variables defined in their enclosing
scope, even when that scope is finished. Local variables which are captured by a
closure are garbage collected once the function they are defined in has finished
**and** all functions defined inside their scope are themselves GCed.

Now, consider this code:

{% highlight js %}
var run = function () {
  var str = new Array(1000000).join('*');
  var doSomethingWithStr = function () {
    if (str === 'something')
      console.log("str was something");
  };
  doSomethingWithStr();
};
setInterval(run, 1000);
{% endhighlight %}

Every second, we'll execute the `run` function. It will allocate a giant string,
create a closure that uses it, invoke the closure, and return. After it returns,
the closure can be garbage collected, and so can `str`, since nothing remains
that refers to it.

But what if we had a closure that outlasted `run`?

{% highlight js %}
var run = function () {
  var str = new Array(1000000).join('*');
  setInterval(function () {
    console.log('interval');
  }, 100);
};
setInterval(run, 1000);
{% endhighlight %}

Every second, `run` allocates a giant string, and starts an interval that logs
something 10 times a second. The inner function does last forever, and `str` is
in its lexical scope, so this could be a memory leak! Fortunately, JavaScript
implementations (or at least current Chrome) are smart enough to notice that
`str` isn't used in the inner function, so it's not put into the lexical
environment of the inner function, and it's OK to GC the big string once `run`
finishes.

Well, great! JavaScript protects us from memory leaks, right? Well, let's try
one more version, combining the first two examples.

{% highlight js %}
var run = function () {
  var str = new Array(1000000).join('*');
  var doSomethingWithStr = function () {
    if (str === 'something')
      console.log("str was something");
  };
  doSomethingWithStr();
  setInterval(function () {
    console.log('interval');
  }, 100);
};
setInterval(run, 1000);
{% endhighlight %}

Open up the Timeline tab in your Chrome Developer Tools, switch to the Memory
view, and hit record:

![Memory leak in Chrome Timeline](/assets/2013-06-27-leak.png)

Looks like we're using an extra megabyte every second! And even clicking the
garbage can icon to force a manual GC doesn't help. So it looks like we are
leaking `str`.

But isn't this just the same situation as before? `str` is only referenced in
the main body of `run`, and in `doSomethingWithStr`. `doSomethingWithStr` itself
gets cleaned up once `run` ends... the only thing from `run` that escapes is the
second closure which gets passed to `setInterval`. And that closure doesn't
refer to `str` at all!

So even though there's no way for any code to ever refer to `str` again, it
never gets garbage collected! Why? Well, the typical way that closures are
implemented is that every function object has a link to a dictionary-style
object representing its lexical scope. If both functions defined inside `run`
actually used `str`, it would be important that they both get the same object,
even if `str` gets assigned to over and over, so both functions share the same
lexical environment. Now, Chrome's V8 JavaScript engine is apparently smart
enough to keep variables out of the lexical environment if they aren't used by
*any* closures: that's why the first example doesn't leak.

But as soon as a variable is used by *any* closure, it ends up in the lexical
environment shared by *all* closures in that scope. And that can lead to memory
leaks.

You could imagine a more clever implementation of lexical environments that avoids
this problem. Each closure could have a dictionary containing only the variables
which it actually reads and writes; the values in that dictionary would
themselves be mutable cells that could be shared among the lexical environments
of multiple closures. Based on my casual reading of the ECMAScript 5th Edition
standard, this would be legitimate: its description of Lexical Environment
describes them as being "purely specification mechanisms \[which] need not
correspond to any specific artefact of an ECMAScript implementation". That said,
this standard doesn't actually contain the word "garbage" and only says the word
"memory" once.

Fixing memory leaks of this form, once you notice them, is straightforward, as
demonstrated by [the fix to the Meteor bug][fix]. If you have a large object
that is used by some closures, but not by any closures that you need to keep
using, just make sure that the local variable no longer points to it once you're
done with it. Unfortunately, these bugs can be pretty subtle; it would be much
better if JavaScript engines didn't require you to have to think about them.

[meteor]: https://www.meteor.com/
[issue]: https://github.com/meteor/meteor/issues/1157
[fix]: https://github.com/meteor/meteor/commit/49e9813
