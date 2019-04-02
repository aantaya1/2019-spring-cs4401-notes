---
title:  "JIT Compiler"
date:   2019-04-01 23:00:00
categories: information
layout: post
---

At one point in lecture we discussed how modern hardware doesn't allow
the memory spaces to be both writable and executable at the same time. I looked
into this a little bit more and started researching into 'Writing XOR Executable'
(W xor X) memory protection policy, which doesn't allow for the same memory
space to be both writable and executable. While researching that I
stumbled upon the JIT compiler...

[article](https://eli.thegreenplace.net/2013/11/05/how-to-jit-an-introduction)
