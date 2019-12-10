# Buffer-OverFlow前置知识
> 这是本人在做本实验时了解的前置知识，有不对或者不清晰的内容希望能在评论区提出，会认真改正和学习。欢迎交流学习。😬
>
> **目录**
>
> - [Memory Layout 内存结构](#k1)  
> -  [Virtual Address & Physical Address 虚拟地址和物理地址](#k2)  
> -  [Stack Layout  栈结构](#k3)
>  - [Memory Layout 内存结构](#k1)  
> -  [Virtual Address & Physical Address 虚拟地址和物理地址](#k2)  
> -  [Stack Layout  栈结构](#k3)

## Memory Layout

<p id="k1">^_^</p>   
### 概述
 在这张图中，介绍了内存的存储结构，分别是：
- stack：栈
- heap：堆
- BSS Segment：Block Started by Symbol
- Data Segment：数据段
- Text Segment：代码段

### 从代码中学习
```c
...
// data stored in Data Segment
int x = 100;
int main(){
	// data stored on stack (Local Variable)
	int a=2;
	float b = 2.5;
	// data stored in BSS Segment
	static y;
	//allocate memory on heap
	int *ptr = (int*)malloc(2*sizeof(int));
	// value 5 and 6 stored on heap
	ptr[0]=5;
	ptr[1]=6；
	// deallocate memory on heap
	free(ptr);
	return 1;
}
```

1. stack 栈中存储的是**局部变量**(不包括static变量)
 - 可读可写
 - 存储代码中的局部变量
 - 栈的生存期取决于代码块，代码块运行就会分配空间但代码块结束就会自动收回空间。
2. heap 堆
 - 可读可写
 - 存储程序运行期间动态分配的空间，一般为malloc/realloc函数
 - 堆从分配一直存在，一直到执行free函数释放。进程创建就存在，进程死亡就消失
3. BSS Segment 
 - 可读可写
 - 存储未被初始化的**全局**变量或者**static**变量
 - 未被初始化的值会存储到这里，并且会有**默认值为0**。因为操作系统在其中放入了0，OS并不想让未初始化的值获取一个随机的值。
 - 进程创建就存在，进程死亡就消失
4. Data Segment
 - 可读可写
 - 存储未被初始化的全局变量或者static变量
 - 进程创建就存在，进程死亡就消失
5. Text Segment
 - 用于存放指令，运行代码的一块内存空间
 - 空间大小一般在运行之前就已经确定
 - 在这其中可能包含一些只读的常数变量，比如字符串常量。

在上述代码中，已经有了详细的注释每个数据存储到了哪里。但是仍有几点要说明一下。
- 首先，虽然在堆中分配了动态存储空间，但是ptr仍然是局部变量，所以应该放在stack中。
- Data Segment和BSS Segment中存储的变量主要区别就是是否初始化。
- 全局变量存储在`全局数据区`

## Virtual Address & Physical Address

<p id="k2">^_^</p>   
> 由于本人操作系统学的不是很好，所以这里只能简单说一下实验可能会涉及到的知识。后续我会多了解这方面的内容放在博客上。

我们知道，物理地址(Physical Address)是unique的，基于你的物理存储器RAM。

如果一个process在使用一部分，另一个人在同一时间就不可以使用，否则会产生矛盾。

但当讨论process中的内存地址时，就会涉及到Virtual Address，虚拟地址。

每一个Process都有一个自己专属的Virtual Addrss,比如如图中V1、V2的空间大小均为0~4G，它们都可以存在这样一个地址0x5000。虚拟地址和物理地址之间存在一个映射机制，来把虚拟地址映射到物理地址中，这个过程由OS完成。

尽管他们使用了相同的虚拟地址，但是映射到了不同的物理地址。

> 具体的细节我会在后续学习中继续了解，并且整理到博客中。也许后续会在这里放一个链接

这是就存在一个问题，如果V1中的一块虚拟地址在映射不能找到物理地址，这个就成为`valid address`，如果有指令想要从这里获取数据，程序可能会crash掉，必须能映射到物理地址才能访问这一块虚拟地址。

## Stack Layout 

<p id="k1">^_^</p>   
本实验时buffer-Overflow，所以有必要了解一下栈的结构。说到栈，就必须要提及`function`函数，所以我们可以从一个函数中来看。

### 栈结构
```c
void func(int a,int b)
{
	int x,y;
	x = a+b;
	y = a-b;
}
```
当调用`func`时，会把这些数据压入到stack中，如图所示，`High Address`代表栈底，a和b是形式参数，先被压入栈中（**注意C语言形参压栈的顺序是按照形参从右向左的顺序压栈的，原因可以自行百度或者参考[这里](https://www.cnblogs.com/findumars/p/5303022.html)**）

接下来压入 `Return Address`和`Previous Frame Pointer`，接下来压入函数的局部变量x和y（x和y的入栈顺序一般与声明的顺序相同，一般和编译器有关）

其中`Return Address`用来存储调用后的下一条指令的地址，以便执行结束后返回。

另外，从栈底到`Previous Frame Pointer`这部分在函数调用后是固定的，fixed，不可修改的，而且挨在一起。但是这里和下面的局部变量之间可能会存在一个操作系统给的`gap`，也就是地址可能不是相连的。

接下来在介绍更深入的内容之前（觍着脸这么说，因为我也只了解了皮毛，甚至还不懂更不用说讲清楚），首先来看一下这个程序所要做的事情，然后从汇编角度来看一看。

`x=a+b;`，在程序中，想要把a、b相加得到x，在OS中需要转换为汇编语言代码（最终为机器语言）来执行，就必须要知道a和b的地址，那么问题就来了，我们怎么知道这里的a和b的地址呢？

这里可以知道一种思路就是使用偏移量`offset`，但是想使用offset偏移量就必须有一个基址`base`。但是run这个程序的时候，我们并不知道stack是从哪里开始的，因此不可以把stack开始的SP(Stack Point)作为base。**这里引入了一个base point**，在运行的时候规定了一个寄存器`ebp`，在运行过程中这个值是可以确定的，这里指向`Previous Frame Pointer`。（后面会解释为什么）

那么根据我们前面的图示，这是一个32bit机，那么可以得到形参的地址：
- a : ebp+8
- b : ebp+12

同理我们可以知道x的地址，如果是紧挨着ebp的话，x : ebp-4。

对应的汇编代码：
```x86asm
movl 12(%ebp),%eax // 把ebp-12复制到eax也就是b中
movl 8(%ebp),%eax 
addl %edx,%eax  // 相加，值会存在eax中
movl %eax,-8(%ebp) // eax移到ebp-8中
```

由上述代码可以知道，实际上ebp和x之间可能存在一个gap，并且compiler知道这个gap是多少，因此也可以知道x、y的值。

***Local Variables是用户自己的，什么顺序入栈都是可以的；但是Argument不可以，它是为了其他人调用，所以必须遵从编译器的标准，否则会出错***

### c语言函数执行过程
```c
int fun(int a, int b);
int m=10;
int main()
{
	int i=4;
	int j=5;
	m = fun(i,j);
	return 0;
}
int fun(int a, int b)
{
	int c=0;
	c = a+b;
	return c;
}
```
C程序运行的核心是函数的执行和调用。 

### 函数调用链