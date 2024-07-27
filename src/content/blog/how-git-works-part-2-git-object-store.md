---
title: "How Git works: Part 2 - Git Objects"
date: 2024-06-16
author: Chandra Kiran G
slug: how-git-works-part-2-git-objects]
pubDatetime: 2024-06-16T12:18:31
modifiedDatetime: 2024-06-16T12:18:31
draft: true
tags:
  - git
  - git-internals
  - git-object-store
description: Git stores the content using Git objects. I'll try to explain how it works in this blog.
---

## Introduction

This is the second of the series of blogs I'm thinking to write on the Git internals. In the first one, I've tried to explain the Git directory structure. In this one, I'll try to explain Git objects.

As we saw in the first blog, Git stores the content in a folder called objects inside the .git folder. So all the changes we make in the repository are stored in this objects folder. These are called Git objects. Git uses content-based hashing to store the objects. Meaning, the objects are stored based on the content in the object. If the content is the same, the object will be the same. There are 4 types of objects in Git. Let's see what they are.

1. Blob
2. Tree
3. Commit
4. Tag

Every Git object has the following structure:

```
<object-type> <object-body-length>\0<object-body>
```

Git first hashes the content of the object using SHA-1 and this hash is used as the object name. The content of the object is then compressed using zlib and is stored in the objects folder with the hash as the name of the file.

Let's see what each of the Git object in detail.

## Blob

Each individual file in a Git repository is represented as a blob. A blob stands for Binary Large Object. The body of this object is the content of the file. The object type is 'blob'. Let's see an example.

```shell
$ echo "Hello World" > a.txt
$ git init
$ git add a.txt
$ git hash-object a.txt
557db03de997c86a4a028e1ebd3a1ceb225be238
```

The hash of the content of the file a.txt is hashed to 557db03de997c86a4a028e1ebd3a1ceb225be238. So the blob object will be stored at .git/objects/55/7db03de997c86a4a028e1ebd3a1ceb225be238. The content of the object will be:

```shell
$ cat .git/objects/55/7db03de997c86a4a028e1ebd3a1ceb225be238
xK��OR04b�H����/�I�A�I%
```

Oops! We forgot that the content is compressed using zlib. Let's decompress it.

```shell
$ sudo apt-get install qpdf -y
$ cat .git/objects/55/7db03de997c86a4a028e1ebd3a1ceb225be238 | zlib-flate -uncompress
blob 12Hello World
```

The content of the object is 'blob 12Hello World!'. The object type is 'blob'. The object body length is 12. The body of the object is 'Hello World'. Wait a minute! The body of the object is "Hello World" with length 11. Where did that 12 come from? That's because the content written using `echo` will have a newline character at the end. Also note that there is a null character between the body length and the body. That will not be shown here. We can inspect the content more clearly using hexdump.

```shell
$ cat .git/objects/55/7db03de997c86a4a028e1ebd3a1ceb225be238 | zlib-flate -uncompress | hexdump -C
00000000  62 6c 6f 62 20 31 32 00  48 65 6c 6c 6f 20 57 6f  |blob 12.Hello Wo|
00000010  72 6c 64 0a                                       |rld.|
00000014
```

The first 4 bytes in hex stands for 'blob' in ASCII (use [this](https://www.rapidtables.com/convert/number/hex-to-ascii.html) tool to verify). The next byte `20` is a space. The next byte `31` and `32` are the ASCII values of `1` and `2` respectively and represents the length 12. The next byte `00` is the null character. The next bytes are the contents of the file. The last byte `0a` is the newline character.

The advantage of this content based hashing is that if the content of the file is the same, the hash of the file will be the same. So the same file will be stored only once in the objects folder. This is a huge advantage in terms of storage. So how does Git differentiate between the files with the same content but different names? That's where the tree object comes into play. Let's see what that is.

## Tree

Tree object is used to store the directory structure of the repository. The way this stores the directory structure is similar to the UNIX directory structure. The content of the tree object is:

```shell
<file-mode> <file-name>\0<object-hash>
```

Let's see an example.

```shell
$ mkdir dir
$ touch dir/a.txt
$ echo "Hello World" > dir/a.txt
$ git add dir
$ git write-tree
14395e4d7c645304acbc6a94fb1ae20293af70af
```

The tree object is stored at .git/objects/14/395e4d7c645304acbc6a94fb1ae20293af70af. Let's see the content of the object.

```shell
$ git cat-file -p 14395e4d7c645304acbc6a94fb1ae20293af70af
100644 blob 557db03de997c86a4a028e1ebd3a1ceb225be238    a.txt
040000 tree 8984f01e2f4ad953a01facc3c192a763df4e79bf    dir
```

`git cat-file -p` is used to print the content of the object. It won't print the object's content itself but will only bring the body in a human readable format. Let's dissect this before analysing the hexdump of this object. The body of the tree object is:

```shell
<file-mode> <file-type> <object-hash> <file-name>
```

file-mode: 100644 is the filemode of a.txt which means that the file is a normal file. Git only stores whether the file is normal or executable and doesn't store all the permissions as in UNIX. Let's see how the filemode is calculated. The filemode is calculated as follows:

1. The first 3 bits represent the file type. 100 represents a normal file. 040 represents a directory. 120 represents a symbolic link.
2. The next 3 bits represent the file permission. 644 represents the file permission of a regular file (rw-r--r--). 755 represents an executable file (rwxr-xr-x).

file-type: This represents the type of the file. "blob" represents a file. "tree" represents a directory. This enables the Git tree structure to recursively store the directory structure.

object-hash: This is the hash of the object. This can be a hash of a blob object or a tree object.

file-name: This is the name of the file or directory.

Let's see the hexdump of the tree object to understand it more clearly.

```shell
$ cat .git/objects/14/395e4d7c645304acbc6a94fb1ae20293af70af | inflate | hexdump -C
00000000  74 72 65 65 20 36 33 00  31 30 30 36 34 34 20 61  |tree 63.100644 a|
00000010  2e 74 78 74 00 55 7d b0  3d e9 97 c8 6a 4a 02 8e  |.txt.U}.=...jJ..|
00000020  1e bd 3a 1c eb 22 5b e2  38 34 30 30 30 30 20 64  |..:.."[.840000 d|
00000030  69 72 00 89 84 f0 1e 2f  4a d9 53 a0 1f ac c3 c1  |ir...../J.S.....|
00000040  92 a7 63 df 4e 79 bf                              |..c.Ny.|
00000047
```

Phew! This looks a bit complicated. And trust me, it is. This is one of the complex object among all other. Let's break it down. As we saw earlier the first 4 bytes are object type. The first four bytes represent 'tree' in ASCII. The next byte `20` is a space. The next 2 bytes `36` and `33` are the ASCII values of `6` and `3` respectively, represents the length 63 of the object's body. The next byte `00` is the null character. The next bytes are the body of the tree object. Let's look into the object's body more carefully now.

```shell "31 30 30 36 34 34 20 61" "2e 74 78 74 00 55 7d b0  3d e9 97 c8 6a 4a 02 8e" "1e bd 3a 1c eb 22 5b e2  38"
$ cat .git/objects/14/395e4d7c645304acbc6a94fb1ae20293af70af | inflate | hexdump -C
00000000  74 72 65 65 20 36 33 00  31 30 30 36 34 34 20 61  |tree 63.100644 a|
00000010  2e 74 78 74 00 55 7d b0  3d e9 97 c8 6a 4a 02 8e  |.txt.U}.=...jJ..|
00000020  1e bd 3a 1c eb 22 5b e2  38 34 30 30 30 30 20 64  |..:.."[.840000 d|
00000030  69 72 00 89 84 f0 1e 2f  4a d9 53 a0 1f ac c3 c1  |ir...../J.S.....|
00000040  92 a7 63 df 4e 79 bf                              |..c.Ny.|
00000047
```

The highlighted bytes above correspond to the first entry of the tree, a.txt. Let's break it down.

1. `31 30 30 36 34 34` represent the file mode of a.txt. The ASCII values of these bytes are `100644`.
2. `20` is a space.
3. `61 2e 74 78 74` is the ASCII representation of `a.txt`, the file name.
4. `00` is the null character.
5. Now the next 20 bytes `55 7d b0 3d e9 97 c8 6a 4a 02 8e 1e bd 3a 1c eb 22 5b e2 38` represent the hash of the blob object a.txt.

These bytes are then followed by the next entry in the tree object.
