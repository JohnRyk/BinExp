# Binary Exploitation 二进制程式开发入门 （基于linux环境）

> ## Introduction
我用了一个[vagrant file](Vagrantfile)来在我的virtual box中准备虚拟环境，在你的机器上直接用就好了：
1. `vagrant up`
2. `vagrant ssh`

注意： vagrant默认是使用virtualbox的，如果想用其它的需要自行安装插件和修改Vagrantfile


## Topics

1. **[Lecture 1.](Lecture1/README.md)**
     * C 程式的内存布局
     * 关于ELF binaries
     * 纵观函数调用发生时栈的状况
     * 从组合语言的角度看：函数的调用和返回
     * $ebp和$esp的概念
     * 可执行的内存区域

1. **[Lecture 1.5.](Lecture1.5/README.md)**
    * Linux是如何找到二进制文件实用程序的
    * 利用linux $PATH 变量进行一些simple的exploit

1. **[Lecture 2.](Lecture2/README.md)**
    * 什么是栈溢出？
    * ASLR基础知识，避开“栈保护”
    * Shellcodes                                            
    * Buffer overflow:                                          
        *  修改程序的执行流程，使其return到其它的地方
        *  把shellcode注入到buffer中，然后获得shell

1. **[Lecture 3.](Lecture3/README.md)**
    * 在ASLR开启的情况下注入shellcode
        * 环境变量

1. **[Lecture 3.5](Lecture3.5/README.md)**
    * 返回到libc的攻击方式
    * 从“不可执行的栈区”中搞到shell
    * 在实施“ret2libc”攻击时栈区的组织情况

1. **[Lecture 4.](Lecture4/)**
    * 这里包含一些小练习，可以尝试一下

1. **[Lecture 5.](Lecture5/README.md)**
    * 什么是格式化字符串漏洞？
    * 检查栈中的内容
    * 写入栈区
    * 写入任意内存地址

1. **[Lecture 6.](Lecture6/README.md)**
    * GOT                               
    * 覆盖GOT入口
    * 利用格式化字符串漏洞 spwan shell

1. **[Lecture 7.](Lecture7/README.md)**
    * 堆
    * Arena, Bins, Chunks. 
    * 释放后使用漏洞利用 Use after free exploit.
    * 双重释放漏洞利用 Double free exploit.
