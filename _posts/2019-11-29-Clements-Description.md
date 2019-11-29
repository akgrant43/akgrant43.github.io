---
layout: post
title: 29 Nov 2019 Clément's Explanation
categories: [Pharo, VM]
---

This is a copy of an email available from the [vm-dev mail archive](http://lists.squeakfoundation.org/pipermail/vm-dev/2019-November/031858.html).

(copied here so I don't lose it)


```
date:	28 Nov 2019, 22:36
subject:	Re: [Vm-dev] [OpenSmalltalk/opensmalltalk-vm] Reproduceable Segmentation fault while saving images (#444)
```

Hi Alistair,

I've just investigated the bug tonight and fixed it in VMMaker.oscog-cb.2595. I compiled a new VM from 2595 and I was able to run the 400 iterations of your script without any crashes. Thanks for the easy reproduction! Last year when I used the GC benchmarks provided by Feenk, with ~10Gb workloads, for the DLS paper [1], I initially had an image crashing 9 times out of 10 when going to 10Gb. I fixed a few bugs on the production GC back then (mainly on segment management) which led the benchmarks to run successfully 99% of the times. But it was still crashing on 1%, since I was benchmarking on experimental GCs with various changes I thought the bug did not happen in the production GC, but it turns out I was wrong. And you found a reliable way to reproduce :-). So I could investigate. It's so fun to do lemming debugging in the simulator.

The GC bug was basically that when Planning Compactor (Production Full GC compactor) decided to do a multiple pass compaction, if it managed to compact everything in one go then it would get confused and attempt to compact objects upward instead of downward (address wise) on the second attempt, and that's broken and corrupts memory.

I started from this script:

```
| aJson anArray |
aJson := ZnEasy get: 'https://data.nasa.gov/resource/y77d-th95.json' asZnUrl.
Array streamContents: [ :aStream |
	400 timesRepeat: [ 
		aStream nextPutAll: (STON fromString: aJson contents).
		Smalltalk saveSession ] ].
```

It makes me however very sad that you were not able to use the simulator to debug this issue, I used it and that's how I tracked down the bug in only a few hours. Tracking things down in lldb would have taken me weeks, and I would not have been able to do it since I work during the week :-).

Therefore I'm going to explain you my process to reproduce the bug in the simulator and to understand where the issue comes from. The mail is quite long, but it would be nice if you could track the bug quickly on your own next time using the simulator. Of course you can skip if you're not interested. @Eliot you may read since I explain how I set-up a Pharo 7 image for simulator debugging, that might come handy for you at some point.

1] The first thing I did was to reproduce your bug, based on the script, both on Cog and and Stack vm compiled from OpenSmalltalk-VM repository. I initially started with Pharo 8, but for some reason that image is quite broken (formatter issue? Integrator gone wild?). So I switched to Pharo 7 stable. It crashes on both VMs, so I knew the bug was unrelated to the JIT. Most bugs on the core VM (besides people mis-using FFI, which is by far the most common VM bug reported) is either JIT or GC. So we're tracking a GC bug.

I then built an image which runs your script at start-up (Smalltalk snapshot: true andQuit: true followed by your script, I select all and run do-it).

2] Then I started the image in the simulator. First thing I noticed is that Pharo 7 is using FFI calls in FreeType, from start-up, and even if you're not using text or if you disable FreeType from the setting browser, Pharo performs in the backgrounds FFI calls for freetype. FreeType FFI calls are incorrectly implemented (the C stack references heap object which are not pinned), therefore these calls corrupts the heap. Running a corrupted heap on the VM has undefined behavior, therefore any usage of Pharo 7 right now, wether you actually text or not, wether freetype is enabled or not in the settings, is undefined behavior. I saw in the thread Nicolas/Eliot complaining that this is not a VM bug, indeed, pinning objects is image-side responsibility and it's not a VM bug. In addition, most reported bug comes from people mis-using FFI, so I understand their answer. There was however another bug in the GC, but it's very hard for us to debug it if it's hidden after image corrupting bugs like the FreeType one here. 
So for that I made that change:

```
FreeTypeSettings>>startUp: resuming
    "resuming ifTrue:[ self updateFreeType ]"
```


saved, restarted the image, and ensured it was not corrupted (leak checker + swizzling in simulation).

3] Then I started the image in the simulator. Turns out the image start-up raises error if libgit cannot be loaded, and then the start-up script is not executed due to the exception. So I made that change:

```
LibGitLibrary>>startUp: isImageStarting
    "isImageStarting ifTrue: [ self uniqueInstance initializeLibGit2 ]"
```


4] Turns out ZnEasy does not work well in the simulator. So I preloaded this line `aJson := ZnEasy get: 'https://data.nasa.gov/resource/y77d-th95.json' asZnUrl`. into a Global variable. The rest of the script remains the same. I can finally run your script in the simulator! Usually we simulate Squeak image and all these preliminary steps are not required. But! It is still easier to reproduce this bug that most bugs I have to deal with for Android at work, at least I don't need to buy an uncommon device from an obscure chinese vendor to reproduce :-).

5] To shortcut simulation time, since the bug happened around the 60th save for me, I build a different script which snapshots the image to different image names. With a crash at snapshot 59 (only change file written to disk), image 57 was the latest non corrupted image. I then started the simulator (The StackSimulator since we are debugging a GC bug, not the Cog simulator, simulation is faster and simpler). I used the standard script available in the workspace of the Cog dev image built from the guidelines. [2]

```
| sis |
    sis := StackInterpreterSimulator newWithOptions: #(ObjectMemory Spur64BitMemoryManager).
    "sis desiredNumStackPages: 8." "Speeds up scavenging when simulating.  Set to e.g. 64 for something like the real VM."
    "sis assertValidExecutionPointersAtEachStep: false." "Set this to true to turn on an expensive assert that checks for valid stack, frame pointers etc on each bytecode.  Useful when you're adding new bytecodes or exotic execution primitives."
    sis openOn: 'Save57.image'.
    sis openAsMorph; run
```

I then let the simulator simulate, went swimming for 1h, and came back 1h30 later (with commute time). The bug happened in the simulator at save 90, I don't know how long it took to reproduce, but < 1h30. Then I had an assertion failure in the compactor:
 `self assert: (self validRelocationPlanInPass: finalPass) = 0`.
Good! From there I debugged using lemming debugging (technique described in [3], Section 3.2). When the assertion has failed, simulation is the clone. I went up in the debugger to the point where the clone was made, and restarted the same GC approximately 40 times during debugging because once the heap is corrupted you cannot know anymore what the problem is, but you need to trigger the problem to understand. 40 lemmings over that cliff :-) Good lemmings.

Then I quickly figured out that the GC was performing two successive compactions, and that the second compaction is broken right at the start (tries to move objects upward). Then I looked at the glue code in-between the 2 compactions, and yeah, in the case where the first compaction has compacted everything, the variables are incorrectly set for the second compaction. I tried fixing the variables but it's not that easy, so instead I just aborted compaction in that case (See VMMaker.oscog-cb.2595).

6] I then compiled a VM from the sources to check Slang translator would not complain, it did not. I then built a stack VM (Cog VM seems to be broken on tip of tree due on-going work for ARMv8 support) and run your script again. I was able to run the 400 iterations without crash. Bug seems to be fixed! 

@Eliot now needs to fix tip of tree, generate the code and produce new VMs. ARMv8 support is quite exciting though, giving that MacBooks do not support 32 bits any more and that the next Macbooks are rumoured to be on ARMv8. One wouldn't want to run the VM in a virtual box intel image :-).

Alistair, let me know if you have questions. I hope you can work with the simulator as efficiently as we can. If you've not seen it, there's this screencast where I showed how I used the simulator to debug JIT bugs [4]. Audio is not very good because my spoken English sucks, but it shows the main ideas.

[1] https://www.researchgate.net/publication/336422106_Lazy_pointer_update_for_low_heap_compaction_pause_times

[2] http://www.mirandabanda.org/cogblog/build-image/

[3] https://www.researchgate.net/publication/328509577_Two_Decades_of_Smalltalk_VM_Development_Live_VM_Development_through_Simulation_Tools

[4] https://clementbera.wordpress.com/2018/03/07/sista-vm-screencast/