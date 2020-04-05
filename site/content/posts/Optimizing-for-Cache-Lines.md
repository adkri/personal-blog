---
title: "Optimizing for Cache Lines"
date: 2020-04-05T13:38:56-04:00
draft: false
tags: ["c"]

---


In this post, we will see how we can write code optimized for cpu cache lines. If this topic is new to you, this post is your friend.



So recently I have been learning about designing data systems, and the first things I learned about were data structures. In these articles the common theme was designing structures while thinking about how they were going to be accessed, and in all them I heard about cache lines.

Even though I took a data structures class and systems class, I seem to have missed the part about cache lines.

However, before we start talking about the let's start with a simple example.

I have an **2d integer array** of say 25,000,000 elements (5000 rows x5000 cols).



```c
int n_rows = 10000;	
int n_cols = 10000;

int** arr = (int**)(malloc(n_rows * sizeof(int*)));
for (int i = 0; i < n_rows; i++)
    arr[i] = (int*)(malloc(n_cols * sizeof(int)));

```

We have two ways we can iterate over them:

1. Iterate each row and then iterate over columns

```c
 for (int i = 0; i < n_rows; i++)
        for (int j = 0; j < n_cols; j++)
            arr[i][j] = i * j;
```

2. Iterate each column and then iterate over rows

```c
for (int j = 0; j < n_cols; j++)
        for (int i = 0; i < n_rows; i++)
            arr[i][j] = i * j;

```



Although the difference might look miminal, the second one took 4x more time than the first one on my machine.

```bash
1. iteration took 0.320760 seconds
2. iteration took 1.440252 seconds
```

I will provide the code I used at the bottom.

So why does it make such a big difference? Well it all goes back to how the cpu works.

First let us look at the memory hierarchy from the CPU, their speeds and their size.

| CPU Registers |
|---------------|
| L1 Cache      |       <1ns, ~10KB
|---------------|
| L2 Cache      |       ~3ns, ~100KB
|---------------|
| L3 Cache      |       ~10ns, ~1MB
|---------------|
| Main Memory   |       ~100ns, ~10GB
|---------------|
| Hard Disk     |       ~1ms, >100GB


So we see that the furthur up the data is stored, the faster the CPU can read it.
The computer caches data to L1, L2 and L3 so it does not have to make the trip to the RAM to get data.

When we load bytes if it is not present in the cache, the computer will bring those memory from the lower memory to higher up in the hierarchy. Usually the memory modules can tranfer 8 bytes of memory at a time. 
The L1, L2, L3 have the memory organized in cachelines which are typically 32-64 bytes in size.
And the memory modules brings bytes to fill these cachelines.

So when we write programs, we want them to have sequential access pattern. This is leads to 'spatial locality'. Spacial locality means that if we are reading a piece of memory, then the next piece of memory we read is also already in the cacheline.

For example, if we are working with arrays of integers,

Integers are usually 4 bytes.
If we assume the cacheline to be 32-bytes, then we can have 8 integers of the array at the same time in the cacheline. This is possible because arrays use sequential blocks of memory.
So if we accessing it linearly we only have to update the cacheline after every 8th element.


Array -> [ 10, 20 , 30, 40 , 50 , 60 , 70 , 80 , 90, 100, 110, 120, 130 ]

Then when we access the first element, the computer actually brings the whole 32 bytes to the cacheline.
32 byte CacheLine -> [10, 20, 30, 40, 50, 60, 70, 80]
Then for the next 7 accesses of the array, the CPU just looks at cacheline, and the reads are very fast.
Then on the 9th access, we have a cache miss, so then the processor needs to go down on the memory hierarchy, possibly to main memory to get the next cachelines.
32 byte CacheLine -> [ 90, 100, 110, 120, 130]
This takes around ~90ns, compared to ~1ns when it was in the cacheline. But then successive accesses are also fast.

So going back to our 2d array example above, why is it slow? Well in the array, each row was a contiguous block of memory, but different rows are in different places in memory. And if we jumo access different locations in memory, everytime it will invalidate the cachelines.

Memory Access for 1)
Row 1: 1 -> 2 -> 3 -> 4 -> ... -> 50000
Row 2: 1 -> 2 -> 3 -> 4 -> ... -> 50000

Memory Access for 2)
Row 1: 1 ->  Row 2: 1 -> ...... -> Row 5000: 1
Row 1: 2 ->  Row 2: 2 -> ...... -> Row 5000: 2

The first one is optimized as it uses the cachelines, where on the second method, it updates the cacheline but then on the next read, it has to update cacheline again for second row. So it never really utilizes the cachelines.

So in conclusion, when writing systems where memory access times are very important, think about how to structure your data structures to possibly fit on the cachelines, and algorithms to use sequential access.

Thank you for reading.

```c
#include<stdlib.h>
#include<stdio.h>
#include<sys/time.h>


int main() {
    int n_rows = 10000;
    int n_cols = 10000;

    int** arr = (int**)(malloc(n_rows * sizeof(int*)));
    for (int i = 0; i < n_rows; i++)
        arr[i] = (int*)(malloc(n_cols * sizeof(int)));

    struct timeval start, end;

    gettimeofday(&start, NULL);
    for (int i = 0; i < n_rows; i++)
        for (int j = 0; j < n_cols; j++)
            arr[i][j] = i * j;
    gettimeofday(&end, NULL);

    double secs = (double)(end.tv_usec - start.tv_usec) / 1000000 + (double)(end.tv_sec - start.tv_sec); 
    printf("iteration took %f seconds\n", secs);

    gettimeofday(&start, NULL);
    for (int j = 0; j < n_cols; j++)
        for (int i = 0; i < n_rows; i++)
            arr[i][j] = i * j;
    gettimeofday(&end, NULL);

    secs = (double)(end.tv_usec - start.tv_usec) / 1000000 + (double)(end.tv_sec - start.tv_sec); 
    printf("iteration took %f seconds\n", secs);

    return 0;
}

```

References:
https://stackoverflow.com/questions/3928995/how-do-cache-lines-work
https://en.wikipedia.org/wiki/CPU_cache#CACHE-LINES
http://daslab.seas.harvard.edu/classes/cs265/index.html#faq
