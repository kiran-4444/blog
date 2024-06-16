---
title: "How Git works: Part 1 - Git Folder Structure"
date: 2024-06-15
author: Chandra Kiran G
slug: how-git-works-part-1-git-folder-structure
pubDatetime: 2024-01-07T12:18:31
draft: false
tags:
  - git
  - git-internals
  - git-folder-structure
description: Git is a distributed version control system. We all know what Git is. We all know how to work with Git. But did you ever wonder how Git works? This is an attempt to put all my learning into a blog to make me better understand the concepts and to give you a chance to explore further.
---

### Introduction

This is the first of the series of blogs I'm thinking to write on the Git internals. I'll try to explain the Git directory structure in this one.

Git tracks and stores the changes in a folder called .git. Let's see what that contains.

```bash
$ git init

$ echo a.txt > a.txt

$ git add a.txt

$ tree .git
.git
├── branches
├── COMMIT_EDITMSG
├── config
├── description
├── FETCH_HEAD
├── HEAD
├── hooks
│   ├── applypatch-msg.sample
│   └──commit-msg.sample
├── index
├── info
│   └── exclude
├── logs
│   ├── HEAD
│   └── refs
│       ├── heads
│       │   └── main
│       └── remotes
│           └── origin
│               ├── HEAD
│               └── main
├── objects
│   ├── 2d
│   │   └── cbb273b464a2301e958341a26487df0b34b504
│   ├── info
│   └── pack
│       ├── pack-146d926a6f751188c9b9c942cf85c5ef6ac35c17.idx
│       └── pack-146d926a6f751188c9b9c942cf85c5ef6ac35c17.pack
├── ORIG_HEAD
├── packed-refs
├── REBASE_HEAD
└── refs
    ├── heads
    │   └── main
    ├── remotes
    │   └── origin
    │       ├── HEAD
    │       └── main
    └── tags
        └── v1.0.0

```

Let's try to explore each file and folder in the order of their priority.

## objects/ Folder

This is the most important folder. This is where Git stores the history of each file in the form of blobs, trees and commits. Git uses a technique called content-based hashing to store blobs (individual files) onto the disk. Meaning, two files with same content will have same entry in objects folder.

But how does it differentiate between those two files then? This is where tree object (analogous to folders) comes into the picture. While blobs are only stored based on the content, tree objects store the files's metadata i.e filename, file mode, etc. This can help Git differentiate files with same content. If you think about it a bit, this is a clever way to store history of files. Disk space is saved by just storing one blob entry for many file with same content.

And finally commit objects store a collection of trees and blobs. Each commit object store the exact state of a Git repository.

### objects/pack/ Folder

If you look at the objects folder above closely (ignore info and pack folders for now) there's a folder and not a file. We'd expect a single object file since we only added a.txt right? This is one of the place where Git smartly stores the objects in folders with folder names being the first two characters of the object's hash. This is to reduce the number of entries in objects folder and make it easier of operating system to iterate. With this technique, the maximum folders that can exist (excluding info and pack folders) under objects folder will be 256 (16 \* 16 combinations using first two hexadecimal characters) [^1] . These are called loose objects. When these loose objects grow more than a certain threshold (6700 by default) Git performs a pack action to store all similar files into a pack file under pack/ directory. This immensely reduces space further.

### objects/info/ Folder

This folder is supposed to store any additional information regarding the files stored. But I've always seen this folder empty.

## index File

This is where all our staging area information is stored. This information is used by various commands like `git commit`, `git status`, `git diff` etc.

## refs/ Folder

Stores the information about various branches and tags. Each file inside the folders heads/, tags/ stores the tip of the commit in a file with same name a branch/tag. For example if you have a branch named main with latest commit being 1ce5...c661a, the file refs/heads/main would store this commit. So when ever you switch branches, this is where Git looks to get the latest commit of that specific branch.

## HEAD File

This file typically points to the current branch's latest commit's ref file (symbolic link). For example if you are on main branch, the HEAD file would contain the following content:

```shell
$ cat .git/HEAD  
ref: refs/heads/main
```

This can also point to a specific commit directly and is called detached HEAD.

## logs/ Folder

This folder stores all the changes made to a specific branch. For example, the following is a simple logs/HEAD file on one of my local repository:

```shell
0000000000000000000000000000000000000000 ea2ffcf2e8acc01a37a5952ebe72d67beb99ab28 Luna <luna-acer@luna-acer.(none)> 1716878649 +0530    clone: from https://github.com/kiran-4444/rgit.git
ea2ffcf2e8acc01a37a5952ebe72d67beb99ab28 ecceb7ef60f7f3b922eacb4331afc86e42bb8ddd Luna <luna-acer@luna-acer.(none)> 1716956657 +0530    pull origin feat/branch: Fast-forward
ecceb7ef60f7f3b922eacb4331afc86e42bb8ddd ecceb7ef60f7f3b922eacb4331afc86e42bb8ddd Luna <luna-acer@luna-acer.(none)> 1716956690 +0530    checkout: moving from main to feat/branch
ecceb7ef60f7f3b922eacb4331afc86e42bb8ddd 96c30493440b57f6c9fbe3f2cfefbe81000ba34f Chandra Kiran G <chandrakiran.g19@gmail.com> 1717994685 +0530 pull: Fast-forward
96c30493440b57f6c9fbe3f2cfefbe81000ba34f 9596a9052e6bf0a4d85167108e09955dfe769ad2 Chandra Kiran G <chandrakiran.g19@gmail.com> 1717994773 +0530 pull: Fast-forward
9596a9052e6bf0a4d85167108e09955dfe769ad2 ecceb7ef60f7f3b922eacb4331afc86e42bb8ddd Chandra Kiran G <chandrakiran.g19@gmail.com> 1718433740 +0530 checkout: moving from feat/branch to main
ecceb7ef60f7f3b922eacb4331afc86e42bb8ddd ecceb7ef60f7f3b922eacb4331afc86e42bb8ddd Chandra Kiran G <chandrakiran.g19@gmail.com> 1718433742 +0530 checkout: moving from main to main

```

## hooks/ Folder

This folder stores some scripts that during run during various Git commands. For example the file commit-msg.sample  contains a script to show the default commit message template.

```shell
$ cat .git/hooks/commit-msg.sample  
#!/bin/sh
#
# An example hook script to check the commit log message.
# Called by "git commit" with one argument, the name of the file
# that has the commit message.  The hook should exit with non-zero
# status after issuing an appropriate message if it wants to stop the
# commit.  The hook is allowed to edit the commit message file.
#
# To enable this hook, rename this file to "commit-msg".

# Uncomment the below to add a Signed-off-by line to the message.
# Doing this in a hook is a bad idea in general, but the prepare-commit-msg
# hook is more suited to it.
#
# SOB=$(git var GIT_AUTHOR_IDENT | sed -n 's/^\(.*>\).*$/Signed-off-by: \1/p')
# grep -qs "^$SOB" "$1" || echo "$SOB" >> "$1"

# This example catches duplicate Signed-off-by lines.

test "" = "$(grep '^Signed-off-by: ' "$1" |
        sort | uniq -c | sed -e '/^[   ]*1[    ]/d')" || {
       echo >&2 Duplicate Signed-off-by lines.
       exit 1
}
```

## config File

This file stores the repository specific configurations.

## info/exclude File

This file stores the entries to that we don't want to track. This can especially be useful when we don't want to track a particular file/folder but also don't want to add it to .gitignore.

## branches/ Folder

This is a deprecated way of storing remote branch URLs. This info is now stored in config file.

## Conclusion

That's it for this one. I'll next write on how Git's object store works. Thanks!

## References

[^1]: [Why does git store objects in directories with the first two characters of the hash?](https://stackoverflow.com/questions/18731887/why-does-git-store-objects-in-directories-with-the-first-two-characters-of-the-h)
