---
layout: post
title: Pharo 8 Dictionary Performance
categories: [Pharo, Pharo8]
---

Some recent work involved generating, saving and loading large dictionaries, up to 1.9M entries.  Performance of said dictoinary natuarally became a concern.

This post was going to be more detailed, but I've lost intermediate results, so now is a somewhat patchy summary.  Sorry.

The keys and values in the largest of the dictionaries were 64-bit SmallIntegers and hash strings, e.g.

```none
307176021913115381 -> 'MrTwuIg8rBa97LA9/vvpfQjjMUjRFEj/0HYExa1YmwM='
```

Using `#collisionStats` showed that of the 771K items in a subset of the dictionary, 216K had collisions during lookup.  Each time there was a collision, on average it had to iterate over 36,554 items in the array to find the key.

Adding a variant that allowed the size of the underlying storage to be increased, reducing the collision rate, significantly impacted the time it took to convert an array of associations in to a dictionary:

```
[ Dictionary newFrom: self scale: 1 ] timeToRun   ==>   0:00:01:55.139
[ Dictionary newFrom: self scale: 2 ] timeToRun   ==>   0:00:01:37.316
[ Dictionary newFrom: self scale: 4 ] timeToRun   ==>   0:00:00:02.463
[ Dictionary newFrom: self scale: 8 ] timeToRun   ==>   0:00:00:01.798
[ Dictionary newFrom: self scale: 16 ] timeToRun  ==>   0:00:00:01.779
```

where `Dictionary class>>newFrom:scale:` is defined as:

```smalltalk
newFrom: aDict scale: anInteger
	"Answer an instance of me containing the same associations as aDict.
	 Error if any key appears twice."
	| newDictionary |
	newDictionary := self new: aDict size * anInteger.
	aDict associationsDo:
		[:x |
		(newDictionary includesKey: x key)
			ifTrue: [self error: 'Duplicate key: ', x key printString]
			ifFalse: [newDictionary add: x]].
	^ newDictionary
```

Using a `PluggableDictionary` with the appropriate hashBlock (`[ :i | i hashMultiply]`) reduced the time down to 0.2 seconds.


`Dictionary>>collectionStats` is defined as:

```smalltalk
collisionStats
	"Answer a dictionary with some information about the receiver's collisions"
	| results missCount scanForwardCounts scanBackwardCounts start finish found index element haltOn |

	results := Dictionary new.
	results at: #size put: self size.

	missCount := 0.
	scanForwardCounts := OrderedCollection new.
	scanBackwardCounts := OrderedCollection new.
	self keysDo: [ :key |
		key = haltOn ifTrue: [ self halt ].
		"Do a basic lookup, based on #scanFor:"
		finish := array size.
		start := (key hash \\ finish) + 1.
		found := false.
		
		(array at: start) key = key ifFalse: 
			[ missCount := missCount + 1.
			index := start + 1.
			[ found not and: [ index <= array size ] ] whileTrue:
				[ ((element := array at: index) isNotNil and: [ element key = key ]) ifTrue: [ found := true ].
				index := index + 1 ].
			found ifTrue: 
				[ scanForwardCounts add: index - start ]
			ifFalse:
				[ index := 1.
				[ found not and: [ index < start ] ] whileTrue:
					[ ((element := array at: index) isNotNil and: [ element key = key ]) ifTrue: [ found := true ].
					index := index + 1 ].
				scanBackwardCounts add: index ] ]
		ifTrue:
			[ found := true ].
		self assert: found ].

	results 
		at: #arraySize put: array size;
		at: #missCount put: missCount;
		at: #scanForwardCounts put: scanForwardCounts;
		at: #scanBackwardCounts put: scanBackwardCounts.
	^ results
```