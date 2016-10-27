---
layout: post
title: 'Next version: HUGE graphics speedup: Valhalla Knights at 60fps + ffmpeg based
  media engine'
date: '2015-05-11T00:49:00.001+02:00'
author: Carlos Ballesteros Velasco
tags:
- tech info
modified_time: '2015-05-18T06:02:48.449+02:00'
thumbnail: http://2.bp.blogspot.com/-lRCRmqu3wIc/VU_T1P_nYzI/AAAAAAAAQ_s/AQJL-Mio3Y0/s72-c/vk_fullspeed.png
blogger_id: tag:blogger.com,1999:blog-5631155793606690437.post-4282185199809098907
blogger_orig_url: http://blog.jspspemu.com/2015/05/next-version-huge-graphics-speedup.html
---

I have been working on some optimizations. Basically I have mostly rewritten the gpu engine and have improved a lot the performance. In a i5@2.2ghz with a Intel HD Graphics 5000, Valhalla Knights works most of the time at full speed. Also I have been working on a ffmpeg based media engine for decoding audio and video (WIP).  
[![](/img/vk_fullspeed.png)](/img/vk_fullspeed.png)

More information after the jump.

<!--more-->

### Huge GPU speedup

In a simplistic way GE in PSP works writting commands into memory and signaling the address of the last command written (updating the stall address). Then media engine, read the commands and process them in parallel. So while one is writting commands, the other is reading them. So it is parallelizable and it could be handled in a multithreaded fashion. But it requires read and write the same memory at once, and that means that it is not compatible with web workers. You could read commands in one worker and when stalling sending a copy of the memory to other worker. Since this is not a bottleneck (at least for the moment), I won't do this.  

Each command is a 32bit word. 8 upper bits are the command type and 24 lower bits are data for that command.  

Processing the commands should be fast.  
Before the optimizations I was reading every command, and using the 8 upper bits to access a table of functions to execute that command. Most of the commands were updating a state structure.  

#### 1st - Simplifying command decoding
First optimization was to change the table for a switch to enable some optimizations.  
The switch was huge and the benefits weren't big.  
Because most of the commands were just updating the state, I simplified it a lot.  
Removed most of the cases of the switch, and in the default case I was copying the data of each command to a Uint32Array state table. So at least 256 words (one for each command).  
The switch just contained some flow commands, and commands that were writting to matrices.  
And I changed fields of the state structure to getters that accessed values from Uint32Array.  
That simplified things a lot and were more readable and less prone to errors.  

#### 2nd - Uploading vertices as they were defined instead of decoding them
In previous version I were reading vertices with a reader, that converted them into a known structure that worked with floats. That way I was able to generate and modify them easily. But that was slow. So I started to read vertices as they are and send them to the host gpu. I had to do several things: sprites, without having geometry shaders, required generating two additional vertices per couple of vertices. So I had to generate dynamic functions that were cloning and modifying those vertices.  
Also colours were in several formats, including 5650, 4444 and 5551 formats. So I modified the shader to decode colors on the fly with code grabbed from jpcsp. Webgl 1.0 doesn't support integer bit operations, and won't support until webgl 2.0, so decoding was a bit tricky.  

#### 3nd - Degenerate trianglestrip buffering
One of the commands is the PRIM command; that draws primitives from vertex/index buffers. It would be similar to drawElements/drawArrays opengl functions.  
PSP supports optimized drawing of elements without indices. Some games generate several PRIM commands adding several vertices. When drawing trianglestrips in several PRIM. You are generating several trianglestrips. But you can combine those trianglestrips adding a degeneratede triangle between strips. That way you can draw all the strips using the same state, at once.  

#### 4th - Webgl requires draw all at once + generating batches
In webgl you can't decide when to flip the buffers. Instead, you draw all the stuff, and when javascript returns to the mainloops is when the flip buffer occurs. So when stalling several times, the code was executed in several frames and that caused flickering. So I needed to be able to render all at once, so I started to prepare a Batch class that includes the state and the vertices/indices to draw. Also, when the emulator will run in a worker, it will be able to send that batches to the UI thread so performing the drawing in parallel to running code. Improving the performance even more.  

After the first optimization, the state was easy to clone, so I was able to queue Batch objects, and draw them when waiting for gpu to complete drawing.  

#### 5th - Upload just one vertex and index buffer once per frame
Instead of upload several index and vertice buffers per frame. I started to draw all the heterogeneous vertices and indices in the same memory segment, and then uploaded before drawing batches.  
Then each batch changes the start address of each attribute in the single buffer and draws a segment of indices. So no buffer changing at all, and that improved performance a lot.  

With those optimizations. I improved a lot performance in several games. With those+previous cpu optimizations Valhalla Knights started to work at full speed.  

### Future optimizations:

1. Run the emulator in a web worker, and send batches to the UI thread. This will reduce the gpu overhead.**  
2. Search for state changes and just call the required gl calls **  
3. Some games do lots of non-degenerable draws with distinct small textures (Tales of Eternia for example). The idea to improve it is to be able to draw at once. This will be able uploading textures into an atlas, and then reconstructing texture coords to target that atlas. That will improve performance a lot in those cases.**  

### Media Engine

PSP has a dedicated processor for gpu processing, decoding video and audio and so on.  
In previous versions I was decoding atrac3+ with MaiAtrac3+ decoding, compiled using emscripten. I was not decoding mp3 or video.  

In latest versions I have compiled ffmpeg with emscripten and created a small bridge for decoding audio, and video in the future. I have compiled with the same configuration as ppsspp, that included just the required formats and codecs.  

After compiling with optimizations the javascript generated code is 6.7 MB, and I am loading it asynchronously, so the emulator starts before it has finished loading, and waits for it when required.  
The last version of the MediaEngine that just included atrac3+ decoding, sized 1.0 MB.
