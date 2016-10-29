---
layout: post
title: Benchmark + headless running (using node.js)
date: '2015-04-09T14:40:00.004+02:00'
author: Carlos Ballesteros Velasco
tags:
- tech info
modified_time: '2015-04-09T14:40:39.897+02:00'
thumbnail: http://4.bp.blogspot.com/-YFPCnD6je_Y/VSZum_gQ3iI/AAAAAAAAQgw/NEF8QqMXZ8I/s72-c/Captura%2Bde%2Bpantalla%2B2015-04-09%2Ba%2Blas%2B14.18.54%2B1.png
blogger_id: tag:blogger.com,1999:blog-5631155793606690437.post-6491380828144642661
blogger_orig_url: http://blog.jspspemu.com/2015/04/benchmark-headless-running-using-nodejs.html
---

I have created a benchmark that allows to test several javascript engines. You can check it here:
[http://benchmark.jspspemu.com/v1/](http://benchmark.jspspemu.com/v1/)

Also now you can run the benchmark from node:
<pre><code>node jspspemu_headless.js data/benchmark/benchmark.prx</code></pre>

This allow you to run any PSP program and see the output it produces using the stdout, also the host process exists when the guest exists. This should run with any node version in any supported host architecture including arm, mips or x86/x64.

<!--more-->

![]({{ '/img/Captura+de+pantalla+2015-04-09+a+las+14.18.54+1.png'|link }})

![]({{ '/img/Captura+de+pantalla+2015-04-09+a+las+14.19.22.png'|link }})
![]({{ '/img/Captura+de+pantalla+2015-04-09+a+las+14.20.09.png'|link }})

Internet Explorer and Spartan are known to be slower for this version. Opera should behave as Chrome, since it uses V8 as javascript engine.

The idea behind this benchmark is to test how browsers/javascript engines are optimized for the emulator (at this stage). After a few seconds it generates a score, the lower value, the better.

The first version of the benchmark tests the emulator's cpu as it is today April, 9th 2015\. Featuring:

* No ASM.JS
* Memory access is behind two methods/objects, so good inlining is necessary, in browsers. It allocates a big ArrayBuffer of ~160MB, in order to be able to reach all the normal memory faster just with a single mask op. Since Node.JS doesn't allow to allocate that memory for a single buffer, memory access in node is slower, because I have to select which buffer to use depending on the address.
* Registers uses small ArrayBuffers with Uint32Array, and Float32Array views. Without local variable allocations.
* It is just grouping simple instruction blocks in functions, except for a single branch level for simple loops (memory copying, setting, ...)
* Each branch (not being a simple loop), jump, function call in the guest is returned to a main loop that lookups the new PC and generates (first time) and calls that function. So here there is most of the current work.

So there is plenty of room for optimizations. The v1 of the benchmark is freezed so we can compare javascript engines. And will be able to compare with newer versions when optimizations are done.
