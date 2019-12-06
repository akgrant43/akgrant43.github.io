---
layout: post
title: Pharo - Tuning Memory Management
categories: [Pharo, VM]
---

If you want to get started in tuning Pharo's memory management some mandatory reading is:

Clément's [Tuning the Pharo garbage collector](https://clementbera.wordpress.com/2017/03/12/tuning-the-pharo-garbage-collector/).

A couple of updates since it was written:

- It looks like the default Eden size has been increased from 3.8MB to 6.5MB (`Smalltalk vm parameterAt: 44`).
- The default Growth Headroom is now 16MB.

