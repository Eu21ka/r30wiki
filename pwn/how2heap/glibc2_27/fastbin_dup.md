---
title: fastbin dup
description: fastbin中的double free漏洞
published: true
date: 2022-09-26T08:18:39.471Z
tags: pwn, 堆利用
editor: markdown
dateCreated: 2022-09-26T08:18:39.471Z
---

# 源码
how2heap中的源码如下
```c
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>

int main()
{
	setbuf(stdout, NULL);

	printf("This file demonstrates a simple double-free attack with fastbins.\n");

	printf("Fill up tcache first.\n");
	void *ptrs[8];
	for (int i=0; i<8; i++) {
		ptrs[i] = malloc(8);
	}
	for (int i=0; i<7; i++) {
		free(ptrs[i]);
	}

	printf("Allocating 3 buffers.\n");
	int *a = calloc(1, 8);
	int *b = calloc(1, 8);
	int *c = calloc(1, 8);

	printf("1st calloc(1, 8): %p\n", a);
	printf("2nd calloc(1, 8): %p\n", b);
	printf("3rd calloc(1, 8): %p\n", c);

	printf("Freeing the first one...\n");
	free(a);

	printf("If we free %p again, things will crash because %p is at the top of the free list.\n", a, a);
	// free(a);

	printf("So, instead, we'll free %p.\n", b);
	free(b);

	printf("Now, we can free %p again, since it's not the head of the free list.\n", a);
	free(a);

	printf("Now the free list has [ %p, %p, %p ]. If we malloc 3 times, we'll get %p twice!\n", a, b, a, a);
	a = calloc(1, 8);
	b = calloc(1, 8);
	c = calloc(1, 8);
	printf("1st calloc(1, 8): %p\n", a);
	printf("2nd calloc(1, 8): %p\n", b);
	printf("3rd calloc(1, 8): %p\n", c);

	assert(a == c);
}
```
# 解析
## 填充tcache
正如上面第9行所说的，这是一个fastbin中double free的一个演示demo。接着我们往下看11到18行，如下所示：
```c
 printf("Fill up tcache first.\n");
  void *ptrs[8];
  for (int i=0; i<8; i++) {
    ptrs[i] = malloc(8);
  }
  for (int i=0; i<7; i++) {
    free(ptrs[i]);
  }
```
了解过tcache的小伙伴可以知道，自glibc2.26版本开始，加入了TCache机制。所以这里我们需要先将tcache中对应bins填满，然后才能将后续的chunk free入fastbin中。这里用了一个数组指针来保存`malloc()`返回的每个堆地址，用for循环free前7个堆块来填充tcache。

> 上面第3行的for循环里可以看到申请了8个堆块，虽然我们填充只需要7个堆块，但是我们需要一个使用中的堆块来阻止最后一个的tcache chunk与top chunk合并。


## 申请受害堆块
```c
printf("Allocating 3 buffers.\n");
int *a = calloc(1, 8);
int *b = calloc(1, 8);
int *c = calloc(1, 8);
```
这三个堆块就是后门将要用来模拟double free的堆块，但是这里用的不是`malloc()`而是`calloc()`。原因是`calloc()`不会调用tcache中的空闲堆块，这样保证了tcache中对应的bins一直处于一个满的状态，使我们后续free掉的chunk能够第一时间地被放入fastbin中。
这里我修改了第一个`calloc()`为`malloc()`，并且执行，这时tcache中如下图所示：

![h2p_2.27_fbdup1.jpg](/how2heap/glibc2.27/fastbin_dup/h2p_2.27_fbdup1.jpg)

## 触发double free
```c
printf("Freeing the first one...\n");
free(a);

printf("If we free %p again, things will crash because %p is at the top of the free list.\n", a, a);
// free(a);

printf("So, instead, we'll free %p.\n", b);
free(b);

printf("Now, we can free %p again, since it's not the head of the free list.\n", a);
free(a);
```
到这一步就是触发double free了，可以看到他先free掉了chunk a，然后他说如果继续释放chunk a的话就会导致程序崩溃。这是因为这时chunk a是这条bin链表的头结点，所以当我们再次释放chunk a时就会被glibc发现，触发程序崩溃（具体原理参考源码）。所以我们需要在再次释放chunk a之前先释放chunk b，这样头结点就变为chunk b，当我们再次释放chunk a时就可以骗过glibc，成功二次释放chunk a。
在39行下断点，查看fastbin链表和堆块，如下图所示：

![h2p_2.27_fbdup2.jpg](/how2heap/glibc2.27/fastbin_dup/h2p_2.27_fbdup2.jpg)
![h2p_2.27_fbdup3.jpg](/how2heap/glibc2.27/fastbin_dup/h2p_2.27_fbdup3.jpg)

可以看到在执行二次释放chunk a时头节点是chunk b（高地址的chunk），其fd指针指向chunk a，chunk a的fd指针内容为空。然后我们再步过，看看执行`free(a)`的效果，如下图所示：

![h2p_2.27_fbdup4.jpg](/how2heap/glibc2.27/fastbin_dup/h2p_2.27_fbdup4.jpg)
![h2p_2.27_fbdup5.jpg](/how2heap/glibc2.27/fastbin_dup/h2p_2.27_fbdup5.jpg)

可以看到chunk a被成功二次释放，而且其fd指针指向chunk b。

## 触发堆块重叠
```c
printf("Now the free list has [ %p, %p, %p ]. If we malloc 3 times, we'll get %p twice!\n", a, b, a, a);
a = calloc(1, 8);
b = calloc(1, 8);
c = calloc(1, 8);
printf("1st calloc(1, 8): %p\n", a);
printf("2nd calloc(1, 8): %p\n", b);
printf("3rd calloc(1, 8): %p\n", c);
```
这里我们看到他申请了三个堆块，但是我们之前其实只释放了两个chunk，其中chunk a被释放了两次，调试看看会出现什么情况。我们先在44行下断点，看看分配第三个chunk之前内存中的情况，如下图所示：

![h2p_2.27_fbdup6.jpg](/how2heap/glibc2.27/fastbin_dup/h2p_2.27_fbdup6.jpg)
![h2p_2.27_fbdup7.jpg](/how2heap/glibc2.27/fastbin_dup/h2p_2.27_fbdup7.jpg)

可以看到fastbin链表里只剩下一个节点（chunk a），执行44行之前我们明明申请了两个chunk，为什么heap查看只查看到一个chunk呢？那是因为chunk a还在fastbin中，所以heap查看任然显示free状态，实际上那块内存已经被分配给了第一个申请的chunk。
> 上诉说的chunk a指的是源码中第21行的chunk a

我们再步过，执行第44行，看看`free(c)`后内存中的情况，如下图所示：

![h2p_2.27_fbdup8.jpg](/how2heap/glibc2.27/fastbin_dup/h2p_2.27_fbdup8.jpg)![h2p_2.27_fbdup9.jpg](/how2heap/glibc2.27/fastbin_dup/h2p_2.27_fbdup9.jpg)

可以看出fastbin中的链表已经没有节点，之前的chunk a也显示正在使用，并且a和c指向的同一个chunk。

## 运行结果
```c
eureka@pc:~/Public/pwn/how2heap/glibc_2.27$ ./a.out 
This file demonstrates a simple double-free attack with fastbins.
Fill up tcache first.
Allocating 3 buffers.
1st calloc(1, 8): 0x55b170b80360
2nd calloc(1, 8): 0x55b170b80380
3rd calloc(1, 8): 0x55b170b803a0
Freeing the first one...
If we free 0x55b170b80360 again, things will crash because 0x55b170b80360 is at the top of the free list.
So, instead, we'll free 0x55b170b80380.
Now, we can free 0x55b170b80360 again, since it's not the head of the free list.
Now the free list has [ 0x55b170b80360, 0x55b170b80380, 0x55b170b80360 ]. If we malloc 3 times, we'll get 0x55b170b80360 twice!
1st calloc(1, 8): 0x55b170b80360
2nd calloc(1, 8): 0x55b170b80380
3rd calloc(1, 8): 0x55b170b80360
```
# 总结
fastbin的double free触发起来其实很简单，只要在二次释放受害chunk之前先释放一个其它的chunk，这样就可以将同一个chunk多次放入fastbin链表中。其造成的直接后果就是使某些变量共用一个内存空间，但也远远不知如此，其它更多的利用将在后面介绍。
> 这里说的”使某些变量共用一个内存空间“而不是”两个变量“的原因是理论上一个chunk可以被我们多次释放，只要在这过程中穿插一个”其它chunk“即可。