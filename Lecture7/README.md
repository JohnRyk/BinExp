# Understanding the Heap



> One Heap to malloc them all, One Heap to free them, One Heap to coalesce, and in the memory bind them...



阅读本章之前，假设你已经了解C中的 `malloc` 和 `free` 。这一章节介绍关于 Heap exploitation 以及 heap 是如何被程序所管理的。这一章节还涉及到了在 `glibc` 中，是如何实现管理 heap 的。



我计划在这章节中介绍如下这些内容：

 1. 堆和动态内存管理函数？

    `malloc` & `free` 函数族

 2. `sbrk` & `mmap`

 3. Arena

 4. Bins

 5. Chunks

 6. 通过一个简单的exploit来看 `fastbins` 是有多好糊弄的。



What I am planning to cover in this lecture:

1. Heap and dynamic memory management functions?
    * `malloc` & `free` fundamentally.
2. `sbrk` & `mmap`
2. Arena
3. Bins
4. Chunks
5. Simple exploit demonstrating how fooling **`fastbins`** are.



### Heap



Heap 是内存中的一部分空间。它被用作存储使用 `*alloc` 函数族动态创建的变量。和Stack上的变量不一样，虽然Stack上的变量空间也是在程序代码运行的时候动态创建的，但Stack上的变量空间都是在程序代码中写死的所以不算动态内存分配。

在这个段中创建的内存属性都是 global 的，这意味着程序中的任何 函数/线程 都可以共享这块内存空间。这块内存空间是通过指针处置的。 如果你对指针不熟悉，可以先看看这个 [this guide]() 

The memory created in this segment is global, in terms that any function/thread in the program can share that memory. They are handled with pointers. If you are not too handy with pointers, you can refer [this guide]() before going forward.

下面这张图展示了应用程序和内存管理系统调用及其组件之间的关系。

The following diagram shows how the application performs memory management calls and which of its components are system (in)/dependent.



![image-20210208151952132](README.assets/image-20210208151952132.png)





`brk()` 系统调用函数可以通过增加 程序的 break location (brk) 从 kernel 获取内存空间（非0初始化的）

初始化状态时，heap的起始指针 ( `start_brk` ) 和 heap 的结尾指针 ( `brk` ) 指向同一个位置。

在程序运行时， `start_brk` 指向 BSS 段的结尾位置（假设ASLR关闭的情况下）

可以通过  `sbrk()`  系统调用来获得程序的 `start_brk` 的值。（需要传递一个参数 `0` ）



`brk` obtains memory (non zero initialized) from kernel by increasing program break location (brk). Initially start (`start_brk`) and end of heap segment (`brk`) would point to same location. 

`start_brk` points to the segment in memory that points to the end of the BSS(in case the ASLR is turned off) during the program run.

 The value for programs `start_brk` can be obtained using `sbrk()` system call, by passing the argument `0` in it.



来看看 系统调用 `sbrk` 是如何工作的：

```C
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>

int main()
{
    printf("Current program pid %d\n", getpid());
    printf("Current break location %p\n", sbrk(0));
    //used to increment and decrement program break location
    brk(sbrk(0) + 1024);
    printf("Current break location %p\n", sbrk(0));
    getchar();
    return 0;
}
```

```bash
Current program pid 10316
Current break location 0x9380000    <----- Current brk
Current break location 0x9380400
```



下面是查看进程的内存空间情况：

And this is how the process looks like in memory.

```bash
$ cat /proc/10316/maps/
08048000-08049000 r-xp 00000000 00:27 45                                 /vagrant/Lecture7/brk_test
08049000-0804a000 r--p 00000000 00:27 45                                 /vagrant/Lecture7/brk_test
0804a000-0804b000 rw-p 00001000 00:27 45                                 /vagrant/Lecture7/brk_test
0935f000-09381000 rw-p 00000000 00:00 0                                  [heap]     <------------------- Heap segment
...
```



来解析一下输出内容：

`0935f000-09381000` 是这个段的虚拟地址范围。

`rw-p` 是 标志 Flags（Read，Write，Non-Xecute，Private）

`00000000`  是文件偏移量 - 因为这个进程不是由任何其它文件所映射出来的，所以这里偏移量是0

`00:00`	是 Majro/Minor 设备号 - 同上，因为这个进程不是由任何其它文件所映射出来的，所以这里是0

`0` 这个是 inode 号 - 同上，因为这个进程不是由任何其它文件所映射出来的，所以这里是0



`0935f000-09381000` is Virtual address range for this segment,
`rw-p` is Flags (Read, Write, Non-Xecute, Private)
`00000000` is File offset – Since its not mapped from any file, its zero here
`00:00` is Major/Minor device number – Since its not mapped from any file, its zero here
`0` is inode number – Since its not mapped from any file, its zero here



我们知道了 `sbrk` 和 `brk` 被用作 get/set 程序的 break 的偏移量。另一方面 `mmap` 是用来从kernel获取内存，然后将它





Since `sbrk` and `brk` are used to get/set the offset of program break on the other hand `mmap` is used to get the memory from the kernel to add it to the heap and update the program `brk`.
The functions that we have discussed so far are able to manage the heap memory. Now we will clear out the picture of `Heap` a bit.



### Heap 的组织结构：



`glibc` 中针对 heap 实现了多种不同的分配方案。它可以帮助我们更方便地操纵 heap。这些不同的分配方案为：Arenas，Bins，Chunks.

Heap have different allocation units implemented by `glibc` that helps for easy heap manipulation. These different allocation units are Arenas, Bins, Chunks.



#### Arenas:



为了更好地处理多线程应用，`glibc 的 malloc` 允许同一时间内存中有超过一个区域活动。

因此，不同的线程可以访问内存中不同的区域，而不会相互影响。这些被分配的内存区域统称作 `arenas` 。

其中，有一个 `主 arena` 它对应应用程序的初始 heap 。在 `malloc` 的代码中有一个静态变量指向这个 arena，其余的每一个 。

In order to efficiently handle multi-threaded applications, `glibc's malloc` allows for more than one region of memory to be active at a time. Thus, different threads can access different regions of memory without interfering with each other. These regions of memory are collectively called `arenas`.
There is one arena, the `main arena`, that corresponds to the application's initial heap. There's a static variable in the `malloc` code that points to this arena, and each arena has a next pointer to link additional arenas.
To understand, the physical heap (allocated to VA) is division-ed into arenas. The main arena starts just after the program break `start_brk`. Arena contains collections of bins in it.




#### Bins:
They are the collection of free memory allocation units called `chunks`. There are 4 different types of bins present in one arena specific to requirement. And each bin contains, allocation data structure that keeps track of free chunks. ** No arena (basically the bins in the arena) keeps track of allocated chunks **. Each arena have specific count of specific bin in it.

The four types of bins are:
1. **Fast**.
    There are 10 fast bins. Each of these bins maintains a single linked list. Addition and deletion happen from the front of this list (LIFO manner). Each bin has chunks of the same size. The 10 bins each have chunks of sizes: 16, 24, 32, 40, 48, 56, 64 bytes etc. No two contiguous free fast chunks coalesce together.
2. **Unsorted**.
    When small and large chunks are free'd they're initially stored in a this bin. There is only 1 unsorted bins.
3. **Small**.
    The normal bins are divided into "small" bins, where each chunk is the same size, and "large" bins, where chunks are a range of sizes. When a chunk is added to these bins, they're first combined with adjacent chunks to "coalesce" them into larger chunks. Thus, these chunks are never adjacent to other such chunks (although they may be adjacent to fast or unsorted chunks, and of course in-use chunks). Small and large chunks are doubly-linked so that chunks may be removed from the middle (such as when they're combined with newly free'd chunks).
4. **Large**.
    A chunk is "large" if its bin may contain more than one size. For small bins, you can pick the first chunk and just use it. For large bins, you have to find the "best" chunk, and possibly split it into two chunks (one the size you need, and one for the remainder).



#### Chunks:

Chunks are the fundamental allocation unit in bins. The memory in the heap is divided into chunks of various sizes depends on where they are allocated (in which bin). Each chunk includes meta-data about how big it is (via a size field in the chunk header), and thus where the adjacent chunks are. When the chunk is free'd, the memory that used to be application data is re-purposed for additional arena-related information, such as pointers within linked lists, such that suitable chunks can quickly be found and re-used when needed. The size of chunk is always in multiple of 8, that allows last three bits to be used as flags.

These three flags are:
- **A**
    Allocated Arena - the main arena uses the application's heap. Other arenas use mmap'd heaps. To map a chunk to a heap, you need to know which case applies. If this bit is 0, the chunk comes from the main arena and the main heap. If this bit is 1, the chunk comes from mmap'd memory and the location of the heap can be computed from the chunk's address.
- **M**
    MMap\'d chunk - this chunk was allocated with a single call to mmap and is not part of a heap at all.
- **P**
    Previous chunk is in use - if set, the previous chunk is still being used by the application, and thus the prev_size field is invalid. Note - some chunks, such as those in fastbins (see below) will have this bit set despite being free'd by the application. This bit really means that the previous chunk should not be considered a candidate for coalescing - it's "in use" by either the application or some other optimization layered atop malloc's original code :-)

The chunk may look like the following:
![chunk](./chunk.png)

### Allocation of small/medium bins:
`glibc` malloc uses first fit algorithm for the allocation of chunks in small/large bins. In this implementation as the name suggest, the first suitable free location of memory which is capable of handling the new request size will split according to the requirement and will be allocated to the new request.
Let see what's going on `use after free exploit`

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main()
{
    printf("If a chunk is free and large enough, malloc will select this chunk.\n");
    printf("This can be exploited in a use-after-free situation.\n");

    char* a = malloc(512);
    char* b = malloc(256);
    char* c;

    printf("1st malloc(512): %p\n", a);
    printf("2nd malloc(256): %p\n", b);
    strcpy(a, "this is A!");
    printf("first allocation %p points to %s\n", a, a);

    printf("Freeing the first one...\n");
    free(a);

    printf("We don't need to free anything again. As long as we allocate less than 512, it will end up at %p\n", a);

    printf("So, let's allocate 500 bytes\n");
    c = malloc(500);
    printf("3rd malloc(500): %p\n", c);
    printf("And put a different string here, \"this is C!\"\n");
    strcpy(c, "this is C!");
    printf("3rd allocation %p points to %s\n", c, c);
    printf("first allocation %p points to %s\n", a, a);
    printf("If we reuse the first allocation, it now holds the data from the third allocation.");
}
```
Run the program and notice that, the pointer `c` and pointer `a` points to the same location.
With small and large chunks/bins there is a hope for use after free exploit. In which the pointer which is freed can be exploited even after it is freed.





### Fast bin allocation:

As I told earlier that fatsbins are maintained as single linked list. When I mention maintains "fastbins are maintained" then I am talking about, the free chunks. **Always remember bins point to free chunks** only not to the allocated chunks. It is the responsibility of programmmer to take care of allocated chunks and free then when they are not in use.
When a chunk is freed, it added to the head of the fast bin list and when it is allocated the head node chunk is removed from the list and is up for allocation.
If not properly maintained fastbins can be exploited to run `double free` exploits. In which programmer by mistake frees a memory twice and the attacker can leverage it to do something malicious.
Look at the following code.
```C
#include <stdio.h>
#include <stdlib.h>
int main()
{
    void *a, *b, *c, *d, *e, *f;
    a = malloc(32);
    b = malloc(32);
    c = malloc(32);

    printf("%p %p %p\n", a, b, c);
    free(a);                                //fastbin head -> a > tail
    //To avoid double free correction or fasttop check, free something other than a
    free(b);                                //fastbin head b -> a > tail
    free(a);                                //fastbin head a -> b -> a > tail

    d = malloc(32);                         //fastbin head -> b -> a -> tail first a popped
    e = malloc(32);                         //fastbin head -> a -> tail first b popped
    f = malloc(32);                         //fastbin head -> tail second a popped
    printf("%p %p %p", d, e, f);
    return 0;
}
```
This will make pointers `d` and `f` points to same memory location. This is called double free exploit.
