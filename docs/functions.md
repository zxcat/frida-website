---
layout: docs
title: Functions
prev_section: presentations
next_section: messages
permalink: /docs/functions/
---

We show how to use Frida to inspect functions as they are called, modify their
arguments, and do custom calls to functions inside a target process.

## Setting up the experiment

Create a file `hello.c`:

{% highlight c %}
#include <stdio.h>
#include <unistd.h>

void
f (int n)
{
  printf ("Number: %d\n", n);
}

int
main ()
{
  int i = 0;

  printf ("f() is at %p\n", f);

  while (1)
  {
    f (i++);
    sleep (1);
  }
}
{% endhighlight %}

Compile with:

{% highlight bash %}
$ gcc -Wall hello.c -o hello
{% endhighlight %}

Start the program and make note of the address of `f()` (`0x400544` in the
following example):

{% highlight bash %}
f() is at 0x400544
Number: 0
Number: 1
Number: 2
…
{% endhighlight %}

## Hooking Functions

The following script shows how to hook calls to functions inside a target
process and report back a function argument to you. Create a file `hook.py`
containing:

{% highlight py %}
import frida
import sys

session = frida.attach("hello")
script = session.create_script("""
Interceptor.attach(ptr("%s"), {
    onEnter: function(args) {
        send(args[0].toInt32());
    }
});
""" % int(sys.argv[1], 16))
def on_message(message, data):
    print(message)
script.on('message', on_message)
script.load()
sys.stdin.read()
{% endhighlight %}

Run this script with the address you picked out from above (`0x400544` on our
example):

{% highlight bash %}
$ python hook.py 0x400544
{% endhighlight %}

This should give you a new message every second on the form:

{% highlight py %}
{u'type': u'send', u'payload': 531}
{u'type': u'send', u'payload': 532}
…
{% endhighlight %}

## Modifying Function Arguments

Next up: we want to modify the argument passed to a function inside a target
process. Create the file `modify.py` with the following contents:

{% highlight py %}
import frida
import sys

session = frida.attach("hello")
script = session.create_script("""
Interceptor.attach(ptr("%s"), {
    onEnter: function(args) {
        args[0] = ptr("1337");
    }
});
""" % int(sys.argv[1], 16))
script.load()
sys.stdin.read()
{% endhighlight %}

Run this against the `hello` process (which should be still running):

{% highlight bash %}
$ python modify.py 0x400544
{% endhighlight %}

At this point, the terminal running the `hello process` should stop counting
and always report `1337`, until you hit `Ctrl-D` to detach from it.

{% highlight bash %}
Number: 1281
Number: 1282
Number: 1337
Number: 1337
Number: 1337
Number: 1337
Number: 1296
Number: 1297
Number: 1298
…
{% endhighlight %}

## Calling Functions

We can use Frida to call functions inside a target process. Create the file
`call.py` with the contents:

{% highlight py %}
import frida
import sys

session = frida.attach("hello")
script = session.create_script("""
var f = new NativeFunction(ptr("%s"), 'void', ['int']);
f(1911);
f(1911);
f(1911);
""" % int(sys.argv[1], 16))
script.load()
{% endhighlight %}

Run the script:

{% highlight bash %}
$ python call.py 0x400544
{% endhighlight %}

and keep a watchful eye on the terminal (still) running `hello`:

{% highlight bash %}
Number: 1879
Number: 1911
Number: 1911
Number: 1911
Number: 1880
…
{% endhighlight %}
