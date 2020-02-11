---
layout: post
title: Pharo 8 - Programmatic Refactoring
categories: [Pharo, Iceberg, Pharo8]
---

[Refactorings](https://github.com/pharo-open-documentation/pharo-wiki/blob/master/General/Refactorings.md) describes how to refactor code using the GUI.  But if you want to programmatically refactor the API is either not intuitive (to me) or a little inconsistent.

So here's a few notes that did the job for me.  There are probably better ways to go about this...

## Move a method to another package

```smalltalk
(SycMoveMethodsToPackageCommand
	for: aCollectionOfMethods
	to: aDestinationPackage) 
		execute
```


## Move a class to another package

```smalltalk
aClass category: aPackageNameString
```

The destination package name can include tags, e.g. `'Collections-Atomic-Base'`.


## Rename a class or trait

```smalltalk
(SycRenameClassCommand new 
	targetClass: aClass;
	newName: aNewNameSymbol)
		execute
```

