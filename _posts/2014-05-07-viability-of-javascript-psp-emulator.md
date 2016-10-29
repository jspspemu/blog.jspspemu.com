---
layout: post
title: Viability of a javascript psp emulator
date: '2014-05-07T02:00:00.000+02:00'
author: Carlos Ballesteros Velasco
tags:
- tech info
modified_time: '2014-05-13T02:05:48.074+02:00'
thumbnail: http://1.bp.blogspot.com/-5Gto6bIkNas/U3FgvD_DAQI/AAAAAAAAMD0/ju1a_hupbmA/s72-c/compilerPerf.png
blogger_id: tag:blogger.com,1999:blog-5631155793606690437.post-3107284900411096914
blogger_orig_url: http://blog.jspspemu.com/2014/05/viability-of-javascript-psp-emulator.html
---

![]( {{ '/img/compilerPerf.png' |link }})

So, It is viable to create a javascript emulator that will run at full speed? The answer is yes.  

Results greater than 100% mean that will work faster than on a real PSP (in theory):  
[http://jspspemu.com/#samples/compilerPerf.elf](http://jspspemu.com/#samples/compilerPerf.elf)  

<!--more-->

Due to javascript to not have "goto" keyword, it will harder to make it faster, but it is definetively possible to get fullspeed with current hardware.  

![]({{ '/img/sprites.png' |link }})

[http://jspspemu.com/#samples/sprite.pbp](http://jspspemu.com/#samples/sprite.pbp)  

About rendering speed. jspspemu is using WebGL and the speed is mostly limited by the gpu, so the rendering speed will be similar to other emulators using OpenGL|ES.
