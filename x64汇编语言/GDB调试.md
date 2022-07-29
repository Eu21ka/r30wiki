---
title: GDB调试
description: 学习gdb的基本使用，修改hello程序
published: true
date: 2022-07-29T14:57:33.368Z
tags: 汇编语言, x64
editor: markdown
dateCreated: 2022-07-29T14:57:33.367Z
---

# GDB
gdb是一个调试器，通过它可以逐步指令的同时，查看内存、寄存器和标志的变化。它是一个命令行程序，要想使用它就得熟悉其基本的指令。

## 调试hello
使用指令`gdb hello`调试之前写的hello程序，输入指令`list`，即可显示多行代码，再次输入则会显示后面的几行，如下图所示。
![x64汇编语言_gdb_1.jpg](/x64汇编语言_gdb_1.jpg){.align-center}
再输入指令`disassemble main`，则可以看到main标签下的反汇编代码，但是gcc默认配置的是`AT&T`语法，我们将使用更为熟悉的`intel`语法。所以使用指令`set disassembly-flavor intel`修改语法，或者在用户目录下（主目录）创建文件`.gdbinit`文件，将之前的set指令写进去，保存退出后，重新登录用户，再次使用gdb语法就更改成功了，如下图所示。
![x64汇编语言_gdb_2.jpg](/x64汇编语言_gdb_2.jpg){.align-center}
在上图中，最左侧一列是程序的指令所存储的内存地址，再右边一列是对应起始地址的偏移，最右边一列就是程序的反汇编代码。
在`hello.asm`中我们将`0x1`这个数传入的寄存器是`rax`，但是在上图中却变成了`eax`。其实通过上篇我们知道，`eax`就是`rax`的第32位寄存器。汇编器觉得用64位寄存器存储一个1位的数字`0x1`有点杀鸡用牛刀的感觉，所以就只使用了低32位寄存器，像上图中的`edi`和`edx`道理也是一样。