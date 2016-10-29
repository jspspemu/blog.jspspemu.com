---
layout: post
title: Streaming PSP games, embedding and CORS
date: '2014-05-09T01:33:00.000+02:00'
author: Carlos Ballesteros Velasco
tags:
- tech info
modified_time: '2014-05-13T01:54:55.272+02:00'
thumbnail: http://3.bp.blogspot.com/-LtkiF63AP0I/U3FcP7whwtI/AAAAAAAAMDg/pRO_zHcUURs/s72-c/flashback.jpg
blogger_id: tag:blogger.com,1999:blog-5631155793606690437.post-7508014631276011175
blogger_orig_url: http://blog.jspspemu.com/2014/05/streaming-psp-games-embedding-and-cors.html
---

![]({{ '/img/flashback.jpg' | link }})

One of the strengths of this emulator is that it is able to stream large games, including zip files, isos, csos, eboots, etc.  

So it is possible for example to stream the psp homebrew reminiscence, a port of flashback, Doom, abuse or kaiten patisser.  

You can test them here (some could be broken in master release):  
[http://jspspemu.com/#http://games.jspspemu.com/Doom.zip](http://jspspemu.com/#http://games.jspspemu.com/Doom.zip) (3MB)  
[http://jspspemu.com/#http://games.jspspemu.com/reminiscence.zip](http://jspspemu.com/#http://games.jspspemu.com/reminiscence.zip) (95MB)  
[http://jspspemu.com/#http://games.jspspemu.com/abuse.zip](http://jspspemu.com/#http://games.jspspemu.com/abuse.zip) (19MB)  
[http://jspspemu.com/#http://games.jspspemu.com/UraKaitenPatissierPSP.zip](http://jspspemu.com/#http://games.jspspemu.com/UraKaitenPatissierPSP.zip) (13MB)

It won't download the whole file, but will read relatively small chunks (usually 128KB) (and perform buffering) when required. So the game will start immediately, won't require large local storages and the bandwidth used will be limited.  

This will allow homebrew developers to allow people to play their games online. Or even embedding it in an iframe.  

In order to be able to do this, you need to use [CORS (Cross-origin resource sharing)](http://en.wikipedia.org/wiki/Cross-origin_resource_sharing)...  

<!--more-->  

To allow this to work, you should expose several headers: "<code>If-Modified-Since,Range,Content-Range,Content-Length</code>" and methods "<code>GET, HEAD, OPTIONS</code>".

An example of nginx configuration would be:  

<pre><code>
add_header 'Access-Control-Allow-Origin' "*";  
add_header 'Access-Control-Allow-Credentials' 'true';  
add_header 'Access-Control-Allow-Methods' 'GET, HEAD, OPTIONS';  
add_header 'Access-Control-Max-Age' 1728000;  
add_header 'Access-Control-Allow-Headers' 'Authorization,Content-Type,Accept,Origin,User-Agent,DNT,Cache-Control,X-Mx-ReqToken,Keep-Alive,X-Requested-With,If-Modified-Since,Range,Content-Range,Content-Length';  
add_header 'Access-Control-Expose-Headers' 'Authorization,Content-Type,Accept,Origin,User-Agent,DNT,Cache-Control,X-Mx-ReqToken,Keep-Alive,X-Requested-With,If-Modified-Since,Range,Content-Range,Content-Length';  
You can read more about CORS here: [http://enable-cors.org/](http://enable-cors.org/)
</code></pre>

![]({{ '/img/abuse.jpg'|link }})
