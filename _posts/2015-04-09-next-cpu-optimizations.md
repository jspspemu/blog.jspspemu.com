---
layout: post
title: Next cpu optimizations
date: '2015-04-09T19:34:00.002+02:00'
author: Carlos Ballesteros Velasco
tags:
- tech info
modified_time: '2015-04-09T19:34:56.982+02:00'
blogger_id: tag:blogger.com,1999:blog-5631155793606690437.post-6017439572160936549
blogger_orig_url: http://blog.jspspemu.com/2015/04/next-cpu-optimizations.html
---

I have resumed the work of the emulator. Now I plan to make some optimizations to be able to reach a good speed in mobile.

Now most generated functions are like this:

<pre><code>
function state(state) {
var expectedRA = state.getRA();
/*08804338*/ /*    lui */  state.gpr[8] = 0x08820000;
/*0880433c*/ /*   addu */  state.gpr[6] = (0 + 0);
/*08804340*/ /*   addu */  state.gpr[3] = (0 + 0);
/*08804344*/ /*  addiu */  state.gpr[8] = (state.gpr[8] + -20752);
/*08804348*/ /*  addiu */  state.gpr[7] = (0 + 5);
/*0880434c*/ /*   addu */  state.gpr[4] = (state.gpr[8] + 0);
/*08804350*/ /*    lui */  state.gpr[2] = 0x00500000;

while (true) {
/*08804354*/ /*     lh */  state.gpr[5] = state.lh(((state.gpr[4] + 0) | 0));
/*08804358*/ /*  addiu */  state.gpr[2] = (state.gpr[2] + -1);
/*0880435c*/ /*  addiu */  state.gpr[4] = (state.gpr[4] + 2);
/*08804360*/ /*    bne */  state.BRANCHFLAG = (state.gpr[2] != 0);
state.BRANCHPC = 0x08804354;
/*08804364*/ /*   addu */  state.gpr[3] = (state.gpr[3] + state.gpr[5]);
if (state.BRANCHFLAG) { state.PC = state.BRANCHPC; } else { state.PC = 0x08804368; }
if (!state.BRANCHFLAG) return;
}
return;
}
</code></pre>

Just a single loop, using the gpr Int32Array directly, and giving up soon. No calling or jumping other functions either. So it is slow. Faster than a pure interpreter, but much slower than a proper dynarec.

How to optimize this?

<!--more-->

## Local variable allocation:

<pre><code>
while (state.gpr[5] < 10) {
    state.gpr[5] = state.lh(((state.gpr[4] + 0) | 0));
    state.gpr[5] = state.gpr[5] + 1;
}
</code></pre>

This is the same as:

<pre><code>
var gpr5 = state.gpr[5];
var gpr4 = state.gpr[4];
while (gpr5 < 10) {
    gpr5 = state.lh(((gpr4 + 0) | 0));
    gpr5 = (gpr5 + 1) | 0;
}
state.gpr[4] = gpr4;
state.gpr[5] = gpr5;
</code></pre>

Javascript engines could do this directly, but since gpr is a view of a ArrayBuffer, that could be accessed as another view, it is a hard optimization and probably most engines won't do it. But local variable could lead to faster register allocation.
You can check it here that using a plain local variable is the fastest in every browser: [http://jsperf.com/loop-with-array-as-variables/4](http://jsperf.com/loop-with-array-as-variables/4)

The idea is to determine registers used in a loop and copy the value into local variables, and use those local variables instead in the ast. After the loop or before a return or a call, we should restore those registers in local variables into the gpr array.

## Bigger functions

Bigger function mean to have branches, and function calls from the guest inside the recompiled function.

Javascript doesn't have "goto". So the solution is harder than generating code into a virtual machine (java, C#) or a target host (directly generating x86, x64 or arm) that have goto instructions.

### Bigger functions (phase 1)

A first simple way to get it working is to create a state machine:

<pre><code>
var stateIndex = 0;
stateLoop: while (true) {
    switch (stateIndex) {
        case 0:
            state.gpr[5] = 0;
        case 1:
            state.gpr[5] = state.lh(((state.gpr[4] + 0) | 0));
            state.gpr[5] = state.gpr[5] + 1;
            if (state.gpr[5] < 10) {
                stateIndex = 1;
                continue stateLoop;
            }
        case 2:
            break stateLoop;
    }
}
return;
</code></pre>

We have a single loop and a switch inside. Each "label" where a branch happens is a case, a jump/branch either ahead or behind is a value change into the switch subject and a continue referring to the while.

That way we can simulate whiles, ifs, and arbitrary jumps that won't fit any of this.

### Bigger functions (phase 2)

After this, we can improve the state machine into something better. We can detect jumps that define a segment between the jump and the label it jumps to that have a simple property: al inside labels and jumps inside that segment, all are inside that segment. So we can convert that into a while or an if and process that segment in a divide and conquer fashion.

That way we can create loops and ifs where it is possible and use machine states when it is not.

## Calls inside functions

Would be great to have function calling other functions instead of returning to a main loop to perform that function call.

### Check the address after the return

In order to make a call we have to take into account that the function we have called could have been changed. Also we should know that MIPS "jal(r)" (Jump And Link (Register)) is not like a x86 "call" instruction. It just updates a register with the return address. But assembly methods can use it not as a normal call. And the function we call to, could have returned to other place. So after that call we have to check that the PC is indeed the PC after that function call. And if that is not the case, we should return to the execution main loop to determine the next function to execute. Either by returning recursively using a return convention or with an exception.

### Function changed/Instruction cache

In some cases programs create code or update some code and they are invalidating the instruction cache. In that case it is possible to have a function referencing another function that have been changed. If you have inlined it, you have to invalidate update all the functions that referenced it. Also what if you have a function cached in a local variable? Since javascript do very good inlining and also have optimistic optimizations. We can just return functions that call another functions that we can change later on and let javascript engines to do all the work.

## Jumps

Jumps are an interesting feature. Some jumps are local to the function we have created, so we can just treat them as an inconditional branch. Some other jumps are outside our function. But, calling a function would grow the calling stack, and that's bad, but our code is in another function.

So in the end a jump is like a call, but removes the current stack frame, or doesn't create a new one. Indeed it is like a tail call. ES6 ensures tail call optimizations. So in the future it would be as easy as calling the function we want, and just after then returning the function without doing anymore. So the stack won't need anymore, and it will replace the stackframe.

But, hey, we are in the present and we are targeting ES5 engines. How we will be able to handle that? With a return convention. So have to remove the current stackframe. How? Returning the function. But ensuring that the caller will call the function we are jumping into. So our returning convention should be able to return a function to jump in. And after every single call we should check if we are in a jump, and calling it immediately after the call that requested the jump.

---

With bigger functions, local variable allocations, calling and jumps, the performance will improve hugely and with javascript engines improving without a stop we will be able to get games running in a browser in a mobile phone sooner than later.
