Download Link: https://assignmentchef.com/product/solved-comp2300-lab-11
<br>
Before you attend this week’s lab, make sure:

<ol>

 <li>you understand how stacks work</li>

 <li>you can write &amp; enable an interrupt handler function</li>

</ol>

In this week’s lab you will:

<ol>

 <li>explore (and exploit) the way the NVIC saves &amp; restores register values when an interrupt handler is executed</li>

 <li>construct the stack for a new <em>process</em>, then (manually) switch the stack pointer and watch the discoboard execute that process</li>

 <li>use multiple stacks to create your own multi-tasking operating system!</li>

</ol>

<h2 id="introduction">Introduction</h2>

Today you’ll write your own operating system—you can call it <em>yournameOS</em> (feel free to insert your own name in there). At the beginning of this course the possibility of writing your own OS may have seemed pretty far away, but you’ve now got all the tools to write a (basic) multitasking OS. This lab brings together all the things you’ve learned in this course, especially if you have a crack at some of the extension challenges in <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/11-diy-operating-system/#exercise-4">Exercise 4</a>.

<p class="think-box">Discuss with your imaginary lab neighbour: how is it that your computer can do <em>heaps</em> of things at once (check emails, have multiple programs and browser tabs open, check for OS updates, idle on Steam, etc.)? Is there just a giant <code>main</code> loop which does all those things one-at-a-time? Or is there some other way to achieve this?

The basic idea of today’s lab is this: instead of just using the default stack (i.e. leaving the stack pointer <code>sp</code> pointing where it did at startup) you’ll set up and use <em>multiple</em> different stacks. As you’ll see, a stack is all you need to preserve the <strong>context</strong> for a <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/lectures/week-10/#process-definition">process</a>—an independent sequence of execution—and switching between processes is as simple as changing the stack pointer <code>sp</code> to point to a different process’s stack. The interrupt hardware (i.e. the NVIC which you’ve been <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/courses/comp2300/labs/09-input-through-interrupts/">using in labs</a> <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/10-connecting-discoboards/">for the last couple of weeks</a>) even does a bunch of this work for you.

Plug in your discoboard, fork &amp; clone the <a class="acton-tabs-link-processed" href="https://gitlab.cecs.anu.edu.au/comp2300/2021/comp2300-2021-lab-11">lab 11 template</a> and let’s get started.

<h2 id="exercise-1">Exercise 1: anatomy of an interrupt handler stack frame</h2>

In the first exercise it’s time to have a close look at how the current <em>execution context</em> is preserved on the stack when an interrupt is triggered.

Using a simple <code>delay</code> loop and the the usual helper functions in <code>led.S</code>, modify your program so that after <code>main</code> it enters an infinite <code>redblink</code> loop which blinks the red LED on and off at a frequency of <em>about</em> 1Hz. The exact numbers aren’t important in this exercise, so pick some timing values which seem about right to you.

When the <code>redblink</code> loop is running, pause the execution using the debugger and have a look at the various register values—<code>lr</code>, <code>pc</code>, <code>sp</code> <code>r0</code>–<code>r3</code>—you should be starting to get a feel for the numbers you’ll see in each one. These values make up the execution <em>context</em>—the “world” that the CPU sees when your program (i.e. your <code>redblink</code> loop) is running.

Then, enable and configure the SysTick timer to trigger an interrupt every millisecond. There’s a big comment (starting at <code>main.S</code> line 12 in the <a class="acton-tabs-link-processed" href="https://gitlab.cecs.anu.edu.au/comp2300/2021/comp2300-2021-lab-11">template repo</a>) giving you some hints—you just need to write the bits to the correct memory addresses. When figuring out the value for the reload value register (<code>SYST_RVR</code>) remember that your board runs at 4MHz on startup.

Once that’s working, you should be able to set and trigger a breakpoint in the “do-nothing” <code>SysTick_Handler</code> at the bottom of <code>main.s</code><sup id="fnref:lab-9-refresher" role="doc-noteref"><a class="footnote acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/11-diy-operating-system/#fn:lab-9-refresher">1</a></sup>. When this breakpoint is triggered, use the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/02-first-machine-code/#reverse-engineering">memory view</a> to poke around on the stack—remember that <code>sp</code> points to the “top” of the stack, and the rest of the stack is at higher memory addresses than <code>sp</code> (which will appear <em>below</em> the <code>sp</code> memory cell on the screen in the Memory Browser because the addresses are ordered from lower addresses at the top to higher addresses at the bottom). Can you see any values which look similar to the values you saw when you were looking around the execution context earlier?

Here’s what’s happening: when the SysTick interrupt is triggered, as well as switching the currently-executing instruction to the <code>SysTick_Handler</code> function, the NVIC also saves the context state onto the stack<sup id="fnref:gory-details" role="doc-noteref"><a class="footnote acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/11-diy-operating-system/#fn:gory-details">2</a></sup>, so that the stack before &amp; after the interrupt looks <em>something</em> like this (obviously the actual values in memory will be different, but it’s the position of each value on the stack that’s the important part):

<img decoding="async" alt="Stack before/after interrupt" data-recalc-dims="1" data-src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/labs/lab-11/exeption-stack.png?w=980&amp;ssl=1" class="lazyload" src="data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw==">

 <noscript>

  <img decoding="async" src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/labs/lab-11/exeption-stack.png?w=980&amp;ssl=1" alt="Stack before/after interrupt" data-recalc-dims="1">

 </noscript>

Don’t be fooled by the register names (e.g. <code>lr</code> or <code>xpsr</code>) alongside the values in the stack. While the interrupt handler (in this case <code>SysTick_Handler</code>, but it’s the same for all interrupts) is running, that context isn’t in the registers, it’s “frozen” on the stack. When the handler returns (with <code>bx lr</code>, as usual) this context is popped off the stack and back into the registers and the CPU picks up where it left off before.

<p class="think-box">Discuss with your imaginary neighbour—how does the program know to do all this context save/restore stuff when it returns from the interrupt handler? Why doesn’t it just jump back to where it came from like a normal function?

You might have noticed a slightly weird value in the link register <code>lr</code>: <code>0xFFFFFFF9</code>. You might have thought “that doesn’t look like any return value I’ve seen before—they usually look like <code>0x8000cce</code> or <code>0x80002a0</code>”. Well, the trick is that the value <code>0xFFFFFFF9</code><sup id="fnref:exc-return" role="doc-noteref"><a class="footnote acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/11-diy-operating-system/#fn:exc-return">3</a></sup> isn’t an regular location/label in the code part of your program, it’s a special <strong>exception return</strong> value. When the CPU sees this value in the target register in a <code>bx</code> instruction then it does the whole “pop the values off the stack (including the new <code>pc</code>) and execute from there” thing.

<p class="push-box">Commit &amp; push your “empty SysTick handler” program to GitLab. That’s all you need to do for Exercise 1, it’s just laying the groundwork for what’s to come.

<h2 id="exercise-2">Exercise 2: a handcrafted context switch</h2>

<p class="think-box">Using a carefully-prepared stack, is it possible to call your <code>redblink</code> loop function without calling it directly using a <code>bl</code> instruction?

The answer is <em>yes</em>, and that’s what you’re going to do in Exercise 2. Disable (or just don’t enable) your SysTick interrupt—you won’t be needing it in this exercise.

Again, the key takeaway from Exercise 1 is that the context (the “world” of the current process’s execution) can be “frozen” on the stack, and then at any time you can “unfreeze” the process and send it on its way by popping those values off the stack and back into the registers.

In the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/11-diy-operating-system/#exercise-1">last exercise</a>, the frozen context was placed on the stack automatically by the NVIC before the interrupt handler function was called, but in this exercise you’re going to hand-craft your own context stack by writing the appropriate values into memory near the stack pointer.

To do this, you’ll need a chunk of discoboard memory which isn’t being used for anything else. There are several ways you could do this, but this time let’s just pick a high-ish address (say, <code>0x20008000</code>) in the RAM section of the discoboard’s <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/lectures/week-3/#memory-address-space">address space</a>.

<p class="extension-box">You can get away with this since your program is the only thing running on the discoboard, so if the other parts of your program leave that memory alone then you’ll be ok. On a multi-tasking OS, though, you have to share the memory space with other programs (some of which you didn’t write and you don’t know how they work) and so this assumption may not hold. There are a few ways to deal with this problem—can you think of how you might do it?

Once you’ve picked an address for your new stack pointer, you need to create the stack frame. This can be anywhere in memory—there’s nothing special about “stack memory”, it’s just a bunch of addresses that you read from &amp; write to with <code>ldr</code> and <code>str</code> (and friends). The memory address described above (<code>0x20008000</code>) could be any old place where there’s a bit of RAM which you’re not using for some other purpose.

To create stack frame, write a <code>create_process</code> function which:

<ol>

 <li>loads the new stack pointer address (above) into <code>sp</code></li>

 <li>decrements the stack pointer by 32 bytes (8 registers, 4 bytes per register) to make “room” for the things you need to put on the stack</li>

 <li>writes the correct values on the stack (see the picture above) to represent a running <code>redblink</code> loop

  <ul>

   <li>the status register (you can use the default value of <code>0x01000000</code>) goes at an offset of <code>28</code> from your new stack pointer</li>

   <li>the program counter <code>pc</code> should point to the next instruction (which might be a label) to execute when the process is restored</li>

   <li>the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/lectures/week-5/#link-register">link register</a> <code>lr</code> should point to the instruction for the process to return to when it’s “done” (this doesn’t matter so much for the moment, because your <code>redblink</code> loop is infinite—it never <code>bx lr</code>s anywhere)</li>

   <li>put whatever values you need into the slots for <code>r12</code> and then <code>r3</code>–<code>r0</code>—these are just the register values (arguments, basically) for your <code>redblink</code> process (think: do you need anything particular in here, or does it not matter for how your <code>redblink</code> loop runs?)</li>

  </ul></li>

</ol>

Once you’ve created the stack for your new process, write a <code>switch_context</code> function to actually <em>make</em> the switch. This function takes one argument (the new stack pointer) and does the opposite of step 3 above, loading the “context” variables from the stack and putting them back into registers:

<ol>

 <li>restore (i.e. put back) the flags into the <code>xpsr</code> register (since this is a special register you can’t just <code>ldr</code> into it, you have to load into a normal register like <code>r0</code> first and then use the “move to special register” instruction<sup id="fnref:msr-instruction" role="doc-noteref"><a class="footnote acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/11-diy-operating-system/#fn:msr-instruction">4</a></sup> <code>msr apsr_nzcvq, r0</code>)</li>

 <li>restore the rest of the registers except for <code>pc</code></li>

 <li>make sure the stack pointer <code>sp</code> points to the “new” top of the stack (i.e. after the <code>redblink</code> context has been popped off)</li>

 <li>finally, set the <code>redblink</code> process running by restoring the <code>pc</code>. Make sure that you have declared <code>redblink</code> as a function, e.g.<pre><code class="language-ARM hljs"><span class="hljs-symbol">.type</span> redblink, %<span class="hljs-meta">function</span><span class="hljs-symbol">redblink</span>:  ...</code></pre></li>

</ol>

<p class="think-box">Why can’t you restore <code>pc</code> with the rest of the registers in step 2?

<p class="push-box">Write a program which creates a <code>redblink</code> stack frame “by hand” in <code>create_process</code> and then switches to this new <code>redblink</code> context using <code>switch_context</code>. When it runs, your program should blink the red LED. Commit &amp; push your program to GitLab.

<p id="what-about-r4-r11" class="extension-box">You may have noticed that the interrupt handling procedure only preserves <code>r0</code>–<code>r3</code>, but not <code>r4</code>–<code>r11</code>. This won’t bite you if your processes don’t use <code>r4</code>–<code>r11</code>, but how could you modify your <code>switch_context</code> function to also preserve the state of those registers?

<h2 id="exercise-3">Exercise 3: writing a scheduler</h2>

<p class="think-box">What’s the <em>minimum</em> amount of data (of any type) that you need to store to keep track of a process?

To turn what you’ve written <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/11-diy-operating-system/#exercise-2">so far</a> into a fully-fledged multitasking OS, all you need is a scheduler function which runs regularly (in the <code>SysTick_Handler</code>) and makes the context switch as appropriate.

In this exercise you’ll put these pieces together to create version 1 of <em>yournameOS</em>. <em>yournameOS</em> is pretty basic as far as OSes go, it only supports two concurrent processes (for v1, at least). One of them blinks a red light, and the other one blinks a green one (but with a different blink period—time between blinks).

The bookkeeping required for keeping track of these two pointers is just three words: two stack pointers, and a value for keeping track of which process is currently executing. You can the whole process table in the data section like this (note from the difference between the stack pointer values that the OS has a maximum stack size of about 4kB):

<pre><code class="language-ARM hljs"><span class="hljs-symbol">.data</span><span class="hljs-symbol">process_table</span>:<span class="hljs-symbol">.word</span> <span class="hljs-number">0</span> <span class="hljs-comment">@ index of currently-operating process</span><span class="hljs-symbol">.word</span> <span class="hljs-number">0x20008000</span> <span class="hljs-comment">@ stack pointer 1</span><span class="hljs-symbol">.word</span> <span class="hljs-number">0x20007000</span> <span class="hljs-comment">@ stack pointer 2</span></code></pre>

The only other tricky part is to combine the “automatic” context save/restore functionality of the interrupt handler (as you saw in Exercise 1) with the “manual” context save/restore behaviour of your <code>switch_context</code> function from Exercise 2. You probably don’t even need a separate <code>switch_context</code> function this time, you can just do it in the <code>SysTick_Handler</code>.

You can structure your program however you like, but here are a few bits of functionality you’ll need:

<ol>

 <li>a <code>create_process</code> function which initialises the stack (like you did in the previous exercise) for each process you want to run</li>

 <li>a <code>SysTick_Handler</code> (make sure you re-enable the SysTick interrupt) which will</li>

</ol>

<ul>

 <li>read the first entry in the process table to find out which process is currently executing</li>

 <li>pick the <em>other</em> process and swap <em>that</em> stack pointer into the <code>sp</code> register (but don’t change the <code>pc</code> yet!)</li>

 <li>update the <code>process_table</code> so that it shows the new process as executing</li>

 <li>trigger an interrupt return to get things moving again (make sure the handler function still exits with a <code>bx</code> to the special value <code>0xFFFFFFF9</code>)</li>

</ul>

If you get stuck, remember to step through the program carefully to find out exactly what’s going wrong.

<p class="push-box">Write <em>yournameOS</em> version 1, including both a <code>redblink</code> and <code>greenblink</code> processes which execute concurrently, and push it up to GitLab.

<h2 id="exercise-4">Exercise 4: pimp your OS</h2>

<img decoding="async" alt="xzibit" data-recalc-dims="1" data-src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/labs/lab-11/xzibit.jpg?w=980&amp;ssl=1" class="lazyload" src="data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw==">

 <noscript>

  <img decoding="async" src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/labs/lab-11/xzibit.jpg?w=980&amp;ssl=1" alt="xzibit" data-recalc-dims="1">

 </noscript>

<p class="push-box">Whatever you made for your extension task, push it up to GitLab with a short note for future-you to remind yourself what you actually did. Don’t forget to also write a suitably self-congratulatory commit message. Well done, you!

<span class="kksr-muted">Rate this product</span>

Once you’ve got your multi-process <em>yournameOS</em> up and running, there are several things you can try to add some polish for version 2. This exercise provides a few ideas—some of these are fairly simple additions to what you’ve got already, while others are quite advanced. Ask your tutor for help, read <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/v_media/manuals/stm32-L476G-discovery-reference-manual.pdf">the</a> <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/v_media/manuals/ARMv7-M-architecture-reference-manual.pdf">manuals</a>, and try to stretch yourself!

<ol>

 <li>modify the scheduler to also save &amp; restore the <em>other</em> registers (<code>r4</code>–<code>r11</code>) on a context switch (as mentioned <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/11-diy-operating-system/#what-about-r4-r11">earlier</a>) so that the processes are fully independent (currently, <em>yournameOS</em> v1 doesn’t preserve those registers, so if your processes are using them then the context switch will stuff things up)</li>

 <li>add support for an arbitrary number of processes (not just two)</li>

 <li>add the ability for processes to <em>sleep</em>—to manually signal to the OS that they’re ready to be switched out</li>

 <li>add the ability for processes to <em>finish</em>—to call their return address (in <code>lr</code>) and exit</li>

 <li>add process <em>priorities</em>, and a more complex scheduler which takes these priorities into account</li>

 <li>add the ability to press the joystick and manually trigger a context switch, but be careful—what happens if another interrupts occurs <em>while</em> the scheduler function is executing?</li>

 <li><strong>advanced</strong>: use the synchronization instructions <code>ldrex</code> and <code>strex</code> to add a critical section so that each process can <em>share</em> a resource (e.g. a memory location) without stepping on each other’s toes (for reference, look at the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/lectures/02-contents/#Chapter_6">Asynchronism</a> lecture slides &amp; recordings)</li>

 <li><strong>advanced</strong>: use thread privileges &amp; the Memory Protection Unit (<em>Section B3.5</em> in the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/v_media/manuals/ARMv7-M-architecture-reference-manual.pdf">ARM reference manual</a>) to ensure that each process can only read &amp; write to its own (independent) sub-region of the discoboard’s memory?</li>

 <li><strong>Pavel</strong><sup id="fnref:pavel" role="doc-noteref"><a class="footnote acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/11-diy-operating-system/#fn:pavel">5</a></sup>: write a 3D graphics library and build an HDMI connector &amp; driver using the GPIO pins, then port <a class="acton-tabs-link-processed" href="https://github.com/id-Software/Quake">Quake</a> to the discoboard</li>