---
layout: post
title: Experimenting with web workers
date: '2015-05-18T06:47:00.001+02:00'
author: Carlos Ballesteros Velasco
tags:
- tech info
modified_time: '2016-10-27T12:45:57.693+02:00'
blogger_id: tag:blogger.com,1999:blog-5631155793606690437.post-7496195323969177016
blogger_orig_url: http://blog.jspspemu.com/2015/05/experimenting-with-web-workers.html
---

Last week I was experimenting with web workers, trying to separate rendering related tasks from the cpu emulation itself. It was a bittersweet experience.

### A bit of context:

Webworkers allows you to do multithreading without all the complexities of traditional threads: mutex, semaphores, locks. In fact, do proper multithreading is hard and prone to errors, and the worst errors: errors that happens sometimes and are not consistently reproduced. Also sometimes you can just reproduce them when the system is overloaded and thus the worst moment for the bug to trigger.  

So open web's workers, actionscript workers or dart's isolates try to avoid most of those problems. How? There is no shared memory. Two workers can't access the same memory. Each one has its own heap and they are isolated. So you are passing messages between each other or even transferring data withot copying; in the case of web workers, you can transfer ArrayBuffer objects without copying them.  

Even when each worker can work on its own, and emit messages whenever they want. It is always preferred to have a main worker and send tasks to others, because of simplicity. Ideally you are using workers for intensive tasks like decompression, decoding, and so on. MediaEngine is ideally for this. You can decode audio and video there in a request/response/task fashion. Also you can do that for crypto and zlib compression.  

<!--more-->

### Complexity:

Trying to separate emulation from rendering in workers meant move lots of things to the worker. Since webgl can't be done in a worker directly, I was not able to move just the rendering to a worker, but I had to move everything else to a worker, and keep rendering in the UI worker. Also, even when there is a standard for Audio Workers, there is no implementation yet. So the emulator is generating audio data in the worker, but again, it has to be sent to the main UI. So I ended with lots of messing between the emulation worker and the UI. also debugging grow in complexity, even when it is supported in chrome's developer tools.  

### Inconsistencies:

Chrome doesn't support workers creating other workers, while firefox allows that.  
Chrome allows to pass a File object (created when the user selects a file), while firefox doesn't allow that.  

### Overhead:

The first problem I faced was a terrible slowdown. Sending messages was costly. At least in Chrome. So costly that I was about to give up using web workers, and I was not sending audio data yet.  

Then I explored a bit more the problem. I was sending several batches to the UI thread, so I can convert the PSP gpu state, into opengl state and then send vertices and textures to the gpu. Each batch contained: 512, 32-bit unsigned integers with the gpu state, a bunch of vertices and indices and raw texture and clut data.  

So which was the problem? The first problem is that I thought that sending a Uint8Array in a message, and marking it as transferrable, would just transfer that data. But I was wrong. What it is really transferred is the underlying ArrayBuffer. And since I have a ~130MB huge buffer for all the psp memory, and I am just slicing that memory, it was sending thet whole 130MB each frame. Wow! So well, transferring was not an option. So I cloned data in that slice, and things started to get better.  
Still laggy. And much slower than the no-workers approach.  
There was still a lot of overhead sending the batches message. So I decided to make it a ton faster.  
How?  
Well. Instead of sending a ArrayBuffer per batch, and since I was already required to clone data and was not able to transfer that (I cannot transfer the whole psp memory, even if it could be fast, I require it for the emulator and can't be used at the same time in both workers).  
Instead of sending lots of ArrayBuffers I decided to just send one with all the vertex, index, texture and clut and state data in it, and just sending integers with indices pointing to that ArrayBuffer. I had to clone data already, so instead of cloning it I was writting it to that ArrayBuffer, and then transferring it to avoid double copying. I reconstructed Array buffer Views in the UI thread with the indices. That way I was reducing the overhead of serializing those views because I had lots of them.  

That way it started to be nearly as fast as the non-worker implementation in chrome. In firefox it got much faster I think. I didn't measured that, but just an impression.  

### Audio and promisifying messages:

I had some problems too when trying to get audio right. With audio, when psp was sending audio data, it had a promise that resolved when the audio was getting empty (when it had less than 6 buffers remaining I think). So it was able to resume the waiting psp thread to prepare more data and send it. With workers I didn't had promises. So I prepared a simple and dirty first approach to be able to send messages that would get a reply eventually. It is probably that the final approach will end like a RPC.  

I got choppy audio and I still have to look at this.  

### Coupling:

To get it working I had to iterate and improve the design of some parts of the app. It was good since I was required to reduce the coupling I had with the DOM in some cases. The GPU itself was coupled to the DOM, now it is completely decoupled and all the GPU is done headlessly, and just generates batches without even knowing who is going to consume that.  

### Final thoughts:

Even when I already used webworkers in some scenarios, I never had such performance/latency requirements. It works really well for tasks like decompressing. But harder to get it right in other cases.  

I will continue experimenting with it, but I will disable it by default in the next programming lockdown. And will continue with the media engine to get video decoding at last.
