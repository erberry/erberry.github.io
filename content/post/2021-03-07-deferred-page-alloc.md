---
title: "Deferred Page Alloc"
date: 2021-03-07T10:38:14+08:00
draft: false
---

## 你要的内存真的是你的吗
在linux上c或者c++常会用到堆内存分配函数malloc，malloc接受一个需要分配空间大小的参数，然后返回分配好的内存空间地址，通常会判断它的返回值是否为NULL，如果为NULL代表内存分配失败。但是有些情况下即使返回了非NULL，当向新的内存地址中写入数据时会遭遇失败（oom或者其他错误）。

原因是linux的copy on write（cow）策略，只有写入时才会真正分配内存。malloc返回的都是虚拟内存地址，当写入时才会分配物理内存。

linux会有一个零页。malloc时分配的内存指向零页。

## 内存预分配还管用吗
思考：一般分配内存后会立即使用（写入）；当希望做一些空间预分配，减少后续多次分配带来的性能消耗时（类似自己管理内存），底层其实也没有真正为你分配内存，这不是很尴尬吗？当然可以通过分配后写入些什么（比如memset为0）来保证真正分配了物理内存。


笔记参考以下文章和讨论：

https://stackoverflow.com/questions/911860/does-malloc-lazily-create-the-backing-pages-for-an-allocation-on-linux-and-othe

https://en.m.wikipedia.org/wiki/Copy-on-write

https://www.win.tue.nl/~aeb/linux/lk/lk-9.html
