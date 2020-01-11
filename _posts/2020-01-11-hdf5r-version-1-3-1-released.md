---
layout: post
title: "hdf5r version 1.3.1 released" 
date: 2020-01-11
categories: hdf5r-package
description: "New version of the R package hdf5r has been released."
published: true
---

I just pushed a maintenance release of hdf5r to CRAN. In this release, no user facing changes are occuring. 
It fixes 2 issues:
- Adding a missing package "formatR" to the "Suggests" lists as it is needed in the examples
- A new version of gcc is being used for R compilation, and one of the default flag settings has changed. 
  Now *-fno-common* is being used ([gcc flags](https://gcc.gnu.org/onlinedocs/gcc/Code-Gen-Options.html)). Due
  to this a few issues cropped up as I was initializing an array in a header file that was included multiple 
  times. 

Enjoy!
