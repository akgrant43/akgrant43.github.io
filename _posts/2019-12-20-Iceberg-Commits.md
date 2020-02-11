---
layout: post
title: Pharo 8 - Iceberg Commits
categories: [Pharo, Iceberg, Pharo8]
---

I'm currently trying to understand how to commit code using Iceberg (programatically, the UI is fine).

The starting point for me is:

```
IceTipCommitBrowser>>doCommit: aCollection message: aString pushing: aBoolean
	self model
		commit:
			(IceTipCommitAction new
				diff: self model workingCopyDiff;
				items: aCollection;
				message: aString;
				yourself)
		then: [ self verifyNeedsRefreshOrClose.
			aBoolean
				ifTrue: [ (IceTipPushAction new repository: self model entity) execute ] ]
```

This is the method called when pressing the `Commit` button in the UI commit dialog.

`aCollection` is a Set of `IceNode`s, all the nodes that are selected in the diff dialog.  The value of the node is an `IceOperation`.  Those that are modified will have a value of `IceModification`, those unchanged will have `IceNoModification`, etc.

`aString` is the commit message.

`aBoolean` flags whether to push the changes to the remote branch.

The `model` is an `IceTipCachedModel` on a `IceTipRepositoryModel`.
