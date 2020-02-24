---
title: Pharo - My Issues Summary
categories: [Pages, Pharo]
---

# Open Issues / PRs


### Remove the need for storing the callbackStack in image

PR: [https://github.com/pharo-project/threadedFFI-Plugin/pull/17](https://github.com/pharo-project/threadedFFI-Plugin/pull/17)


### LGitId>>hexString doesn't work with threaded FFI and is susceptible to buffer overrun

Issue: [https://github.com/pharo-project/pharo/issues/5379](https://github.com/pharo-project/pharo/issues/5379)  
PR: [https://github.com/pharo-vcs/libgit2-pharo-bindings/pull/31](https://github.com/pharo-vcs/libgit2-pharo-bindings/pull/31)


### Iceberg checkout shows incorrect list of changes

Issue: [https://github.com/pharo-project/pharo/issues/5383](https://github.com/pharo-project/pharo/issues/5383)


### Iceberg method history shows only oldest commit

Issue: [https://github.com/pharo-project/pharo/issues/5578](https://github.com/pharo-project/pharo/issues/5578)  
PR: [https://github.com/pharo-vcs/iceberg/pull/1341](https://github.com/pharo-vcs/iceberg/pull/1341)


### git commit crashes the VM

Issue: [https://github.com/pharo-project/pharo/issues/5592](https://github.com/pharo-project/pharo/issues/5592)  
PR: [https://github.com/pharo-vcs/libgit2-pharo-bindings/pull/32](https://github.com/pharo-vcs/libgit2-pharo-bindings/pull/32)  
Feenk: [https://github.com/feenkcom/gtoolkit/issues/796](https://github.com/feenkcom/gtoolkit/issues/796)

The latest status is in the Feenk issue.


### FileDoesNotExistException on existing files on Windows

Issue: [https://github.com/pharo-project/pharo/issues/3571](https://github.com/pharo-project/pharo/issues/3571)  
PR: [https://github.com/pharo-project/opensmalltalk-vm/pull/66](https://github.com/pharo-project/opensmalltalk-vm/pull/66)



# Merged Issues / PRs

### VM memory corruption during GC

Issue: [https://github.com/OpenSmalltalk/opensmalltalk-vm/issues/444](https://github.com/OpenSmalltalk/opensmalltalk-vm/issues/444)


### Base image contains invalid source pointers

Issue: [https://github.com/pharo-project/pharo/issues/4967](https://github.com/pharo-project/pharo/issues/4967)  
PR: [https://github.com/pharo-project/pharo/pull/5120](https://github.com/pharo-project/pharo/pull/5120)


### WaitfreeQeue>>nextOrNilSuchThat: memory leak

Issue: [https://github.com/pharo-project/pharo/issues/3198](https://github.com/pharo-project/pharo/issues/3198)  
PR: [https://github.com/pharo-project/pharo/pull/3205](https://github.com/pharo-project/pharo/pull/3205)
