---
layout: post
title: "测试CPU cache的延迟以及局部性原理"
description: "test the influence of CPU cache"
category: test
tags: [CPU, Cache延迟]
---
{% include JB/setup %}

我们都知道，CPU的高速缓存具有局部性:

1. 空间局部性
2. 时间局部性

这两种局部性体现在：当访问某个地址时，它附近的地址很可能接下来被访问，所以CPU在取得一个地址数据的时候，会连它周围的数据也一起放入cache；当一个地址被访问时，它接下来再次被访问的概率很高，所以采用LRU替换算法来更新cache。  

使用以下代码来测试：

	#include <time.h>
	#include <stdio.h>
	#include <stdlib.h>
	#include <unistd.h>
	
	
	typedef unsigned int   u32int;
	typedef          int   s32int;
	typedef unsigned short u16int;
	typedef          short s16int;
	typedef unsigned char  u8int;
	typedef          char  s8int;
	
	#define LONG_SIZE 4
	#define PRIME_INC 514229
	//#define PRIME_INC 4096
	
	#define PAGE_SIZE  (4*1024)
	#define ONE_GIG  (1024*1024*1024)
	#define TWO_GIG  ((u32int)2 * ONE_GIG)
	#define ARRAY_SIZE  (ONE_GIG/LONG_SIZE)
	
	#define WORDS_PER_PAGE  (PAGE_SIZE / LONG_SIZE)
	#define ARRAY_MASK  (ARRAY_SIZE - 1)
	#define PAGE_MASK  (WORDS_PER_PAGE - 1)
	
	#define N 5
	typedef u32int (*next_func_t)(u32int pageOffset,u32int wordOffset,u32int pos);
当线性扫描地址时，由于CPU提前将数据块装载进入高速缓存，所以使得缓存命中高，性能最好。
	
	u32int linear_walk(u32int pageOffset,u32int wordOffset,u32int pos)
	{
	  return (pos+1) & ARRAY_MASK;
	}
	
当在一个页面的范围内扫描时，虽然是随机的，但是页面的范围有限，使得缓存命中率不会那么低，性能一般。

	u32int random_page_walk(u32int pageOffset,u32int wordOffset,u32int pos)
	{
	  return (pageOffset + ((pos + PRIME_INC) & PAGE_MASK)) & ARRAY_MASK;
	}
在这种情况下，由于访问地址是随机的（在整个2G的空间内），所以空间局部性不能起到作用，反而降低了性能，也就导致了非常低的性能。

	u32int random_heap_walk(u32int pageOffset,u32int wordOffset,u32int pos)
	{
	  return (pos + PRIME_INC) & ARRAY_MASK;
	}
	
	next_func_t nexts[3] = {linear_walk,random_page_walk,random_heap_walk};
	next_func_t next;
	
	int main()
	{
	  u32int *memory = malloc(sizeof(u32int) * (u32int)ARRAY_SIZE);
	  u32int i,walk_type=0,iter,pageOffset,wordOffset,limit,pos;
	  long long result;
	  clock_t start,end;
		
	  for(i=0;i<ARRAY_SIZE;i++){。、
	    memory[i] = 777;
	  }
	
	
	  for(walk_type = 0;walk_type <3;walk_type++){
	    next = nexts[walk_type];
	    for(iter = 0;iter < N;iter++){
	      start = clock();
	      for(pageOffset = 0; pageOffset <ARRAY_SIZE; pageOffset += WORDS_PER_PAGE){
	        for(wordOffset = pageOffset,limit = pageOffset + WORDS_PER_PAGE;wordOffset < limit;wordOffset++){
	          pos = next(pageOffset,wordOffset,pos);
	          result += memory[pos];
	        }
	      }
	      end = clock();
	      printf("%d - %.2fs walk_type:%d\n",iter,1.0*(end-start)/CLOCKS_PER_SEC,walk_type);
	    }
	  }
	
	  if(memory)
	    free(memory);
	}

测试平台为：`Lenovo Y400,core i7 3630QM,2.4GHZ,8G RAM`

数据如下：

	0 - 1.21s walk_type:0
	1 - 1.21s walk_type:0
	2 - 1.22s walk_type:0
	3 - 1.19s walk_type:0
	4 - 1.19s walk_type:0
	0 - 1.43s walk_type:1
	1 - 1.46s walk_type:1
	2 - 1.44s walk_type:1
	3 - 1.45s walk_type:1
	4 - 1.43s walk_type:1
	0 - 6.12s walk_type:2
	1 - 6.45s walk_type:2
	2 - 6.40s walk_type:2
	3 - 6.42s walk_type:2
	4 - 6.65s walk_type:2

结果分析：

1. 顺序访问性能最好，平均1.21s访问完整个1G内存。
2. 局部随机性的访问性能稍差，平均1.45s访问完整个1G内存。
3. 完全随机性访问性能最差，平均6.43s访问完整个1G内存。