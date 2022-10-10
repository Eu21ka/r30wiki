---
title: 调用printf函数
description: 这节主要介绍用汇编来写一个调用printf函数的程序
published: true
date: 2022-10-10T10:30:02.442Z
tags: 汇编语言
editor: markdown
dateCreated: 2022-10-10T07:01:49.434Z
---

# 打印字符串
## alive.asm
```asm
section .data
    msg1 db  "Hello, World",10,0
    msg1Len equ $-msg1-1
    msg2    db  "Alive and Kicking!",10,0
    msg2Len equ $-msg2-1
    radius  dq  357
    pi  dq  3.14

section .bss
section .text
    global main

main:
    push rbp        ;函数序言
    mov rbp, rsp    ;函数序言
    
    mov rax, 1
    mov rdi, 1
    mov rsi, msg1
    mov rdx, msg1Len
    syscall

    mov rax, 1
    mov rdi, 1
    mov rsi, msg2
    mov rdx, msg2Len
    syscall

    mov rsp, rbp
    pop rbp
    mov rax, 60
    mov rdi, 0
    syscall
```

## alive.asm的makefile
```makefile
#makefile for alive.asm
alive: alive.o
        gcc -o alive alive.o -no-pie
alive.o: alive.asm
        nasm -f elf64 -g -F dwarf alive.asm -l alive.lst
```

## alive程序分析
先看到`alive.asm`的第3行，这里定义了`msg1Len`这个变量，`equ`是个汇编伪指令，作用是将符号名和一个表达式或者一个任意文本连接起来，这里将`msg1Len`和表达式`$-msg1-1`连接了起来。


