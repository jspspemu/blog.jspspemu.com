---
layout: post
title: Tools for debugging the emulator
date: '2014-05-18T22:29:00.001+02:00'
author: Carlos Ballesteros Velasco
tags:
- tech info
modified_time: '2014-05-18T22:29:42.496+02:00'
thumbnail: http://2.bp.blogspot.com/-1oPPVkNCIto/U3kT6xlfuAI/AAAAAAAAMFI/FP5nyGuTCKQ/s72-c/javascript_console.jpg
blogger_id: tag:blogger.com,1999:blog-5631155793606690437.post-5948543100880518301
blogger_orig_url: http://blog.jspspemu.com/2014/05/tools-for-debugging-emulator.html
---

While developing the emulator I needed to debug stuff. Besides the pspautotests project that allowed me to verify some behaviours, I need to stop the execution of the emulator in several places, and to trace stuff. Also I needed to debug webgl and to detect which parts of the emulator to optimize.  

### The simplest way of debugging:
{% highlight js %}
console.log(), console.warn(), console.error()
{% endhighlight %}

![]({{ '/img/javascript_console.jpg'|link }})

<!--more-->

### Collapsable groups and tables, helped a lot for testing:

{% highlight js %}
console.groupCollapsed(), console.table()
{% endhighlight %}

![]({{ '/img/collapsable_groups_and_tables.png'|link }})

### Chrome's profiler helped a lot in determining slow functions:

![]({{ '/img/profiling.jpg'|link }})

### Stopping the execution in some conditions:

The **<span style="color: red;"><code>debugger;</code> keyword</span>** (that I didn't know before starting the emulator) helped was a very valuable tool that allowed me to determine some problems. I was able to use that keyword in functions I made, but also in generated code; that includes CPU generated functions, and vertex decoding. Also browsers have a console, that allows you to execute code and view internals, while debugging or while running. So I was able to call stuff like "emulator.threadManager.threads.elements" or evaluate expressions while in a breakpoint.

![]({{ '/img/debugger.png'|link }})

And for debugging webgl, there was a very useful project: [WebGL Inspector](http://benvanik.github.io/WebGL-Inspector/).

---

![]({{ '/img/webgl_inspector.png'|link }})
