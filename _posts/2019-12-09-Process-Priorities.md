---
layout: post
title: Pharo - Process Priorities
categories: [Pharo, Processes]
---

Like most Smalltalk impelementations, Pharo supports 'green threads', meaning that multiple smalltalk threads can be active at once, but they are all run from a single operating system thread, i.e. only one CPU is active at a time.

Pharo processes are pre-emptive, i.e. a higher priority process will always be given the CPU before a lower priority processes.

Currently the scheduling of processes at the same priority is largely non-deterministic because `Smalltalk vm processPreemptionYields` is set to true.  `processPreemptionYields` means that any time a process is pre-empted by a higher priority process it is moved to the back of the queue (at its priority).  Since timers and input events, amongst other things, are typically processed with high priority processes, this means that the timing and number of interruptions a lower priority process will encounter is unpredictible, in turn resulting in the scheduling of that process amongst its priority peers being also unpredicitible.

But what happens when a process priority is lowered?

Executing the following code in a playground (which evaluates code at priority 40):

```
msgs := OrderedCollection new.
launchProc := [
	lowProc := [ 
		msgs add: 'lowProc at 42'.
			 ] forkAt: 42.
	highProc := [ 
		msgs add: 'highProc at 45'.
		Processor activeProcess priority: 41.
		msgs add: 'highProc at 41'.
			 ] forkAt: 45.
	 ] forkAt: 47.
String streamContents: [ :stream |
	msgs do: [ :msg |
		stream 
			<< msg;
			cr ] ].
```

We might reasonably expect the output to be:

```
highProc at 45
lowProc at 42
highProc at 41
```

However it is actually:

```
highProc at 45
highProc at 41
lowProc at 42
```

Even after lowering its priority `highProc` continues to execute.

Looking at `Process>>priority:`:

```
priority: anInteger
	"Set the receiver's priority to anInteger."

	(anInteger between: Processor lowestPriority and: Processor highestPriority)
		ifTrue: [ priority := anInteger ]
		ifFalse: [ self error: 'Invalid priority: ' , anInteger printString ]
```

The method checks that the new priority is valid, but otherwise simply sets the instance variable.

This isn't enough to force the VM to check whether there is a higher priority process ready to run.

Just as a little background: The VM uses an instance of `ProcessorScheduler`, held in the global variable `Processor`, to schedule Pharo processes.  Ignoring some complexity to handle debugging, the currently active process is held in `activeProcess`, and all the processes waiting for the CPU are held in `quiescentProcessLists`.

To get the expected result we need to force the VM to check for a context switch, which we can do with:

```
	prioritySempahore := Semaphore new.
	[prioritySempahore signal] fork.
	prioritySempahore wait.
```

Modifying our original script:

```
msgs := OrderedCollection new.
prioritySempahore := Semaphore new.
launchProc := [
	lowProc := [ 
		msgs add: 'lowProc at 42'.
			 ] forkAt: 42.
	highProc := [ 
		msgs add: 'highProc at 45'.
		Processor activeProcess priority: 41.
		[prioritySempahore signal] fork.
		prioritySempahore wait.
		msgs add: 'highProc at 41'.
			 ] forkAt: 45.
	 ] forkAt: 47.
String streamContents: [ :stream |
	msgs do: [ :msg |
		stream 
			<< msg;
			cr ] ].
```

now gives the expected result:

```
highProc at 45
lowProc at 42
highProc at 41
```
