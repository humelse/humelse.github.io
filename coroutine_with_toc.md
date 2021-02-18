- [ 协程原理](#head1)
	- [ 定义](#head2)
	- [ 特点](#head3)
	- [ 协程实现的底层支持](#head4)
	- [**1 获取当前执行的上下文 int getcontext(ucontext_t \*ucp)**](#head5)
	- [**2 设置上下文信息makecontext**](#head6)
	- [**3 设置当前执行的上下文setcontext**](#head7)
	- [**4 交换上下文swapcontext**](#head8)
- [ 协程实现](#head9)
	- [ 从云风协程库理解协程的机制](#head10)
		- [ 协程调度器数据结构](#head11)
		- [ 协程数据结构](#head12)
		- [ 具体实现](#head13)
			- [ 首先是创建一个schedule:](#head14)
			- [ 创建一个协程:](#head15)
			- [ 设置协程必要信息](#head16)
			- [ 启动协程](#head17)
			- [ 让出执行](#head18)
			- [ 恢复](#head19)
# <span id="head1"> 协程原理</span>

[TOC]

## <span id="head2"> 定义</span>

协程（coroutine），是一种轻量级的用户态线程，操作系统对协程无感知。实现的是协作式调度（非抢占式调度），即协程切换由当前协程控制，主动让出CPU（例如当前协程在等待异步网络IO时）。

通常情况下，一个线程包含多个协程。

## <span id="head3"> 特点</span>

1. 协程是一个并发运行的多任务系统，一般由一个操作系统线程驱动；

2. 协程任务元数据资源占用比操作系统线程更低，且任务切换开销小；

3. 协程是任务间协作式调度，即某一任务主动放弃执行后进而调度另外一任务投入运行。

## <span id="head4"> 协程实现的底层支持</span>

glibc提供了四个函数给用户实现上下文的切换。

```text
int getcontext(ucontext_t *ucp);
int setcontext(const ucontext_t *ucp);
void makecontext(ucontext_t *ucp, void (*func)(), int argc, ...);
int swapcontext(ucontext_t *oucp, const ucontext_t *ucp);
```

glibc提高的功能类似早期setjmp和longjmp。本质上是保存当前的执行上下文到一个变量中，然后去做其他事情。在某个时机再切换回来。从上面函数的名字中，我们大概能知道，这些函数的作用。我们先看一下表示上下文的数据结构（x86架构）。

```text
   typedef struct ucontext_t {
        unsigned long int uc_flags;
        // 下一个执行上下文，执行完本上下文，就知道uc_link的上下文
        struct ucontext_t *uc_link;
        // 信号屏蔽位
        sigset_t          uc_sigmask;
        /*
        栈信息
        typedef struct
        {
        void *ss_sp;
        int ss_flags;
        size_t ss_size;
        } stack_t
        */
        stack_t           uc_stack;
        // 平台相关的上下文数据结构
        mcontext_t        uc_mcontext;
        ...
    } ucontext_t;
```

我们看到ucontext_t是对上下文实现一个更高层次的封装。真正的上下文由mcontext_t结构体表示。比如在x86架构下。他的定义是

```text
typedef struct
  {
      /*
        typedef int greg_t;
        typedef greg_t gregset_t[19]
        gregs是保存寄存器上下文的
    */
    gregset_t gregs;
    fpregset_t fpregs;
    unsigned long int oldmask;
    unsigned long int cr2;
  } mcontext_t;
```

整个布局如下

![img](/Users/zhangjunmei/work/%E5%8D%8F%E7%A8%8B.assets/v2-edf831867265b5841a58471a2b823550_1440w.jpg)



我们了解了基本的数据结构，然后开始分析一开始提到的四个函数。



## <span id="head5">**1 获取当前执行的上下文 int getcontext(ucontext_t \*ucp)**</span>

getcontext是把当前执行的上下文保存到ucp中。我们看看他大致的实现。他是用汇编实现的。首先看一下开始执行getcontext函数的时候的栈布局。

![img](/Users/zhangjunmei/work/%E5%8D%8F%E7%A8%8B.assets/v2-25467ea4c7ed16cfc83f084027b190f4_1440w.jpg)

```text
movl    4(%esp), %eax
```

把getcontext函数入参的地址赋值给eax。即ucp指向的地址。

```text
    // oEAX是eax字段在ucontext_t结构中的位置，这里就是把ucontext_t中eax的值置为0
    movl    $0, oEAX(%eax)
    // 同上
    movl    %ecx, oECX(%eax)
    movl    %edx, oEDX(%eax)
    movl    %edi, oEDI(%eax)
    movl    %esi, oESI(%eax)
    movl    %ebp, oEBP(%eax)
    // 把esp指向的内存的内容赋给eip字段，这时候esp指向的内存保存的值是返回地址的值。即getcontext函数的下一条指令
    movl    (%esp), %ecx
    movl    %ecx, oEIP(%eax)
    /*
     把esp+4（保存第一个入参的内存的地址）对应的地址（而不是这个地址里存的值）赋给esp。

     正常的函数执行流程是主函数压参，call指令压eip，然后调用子函数，
     子函数压ebp，设置新的esp。返回的时候子函数，恢复esp，ebp。然后弹出eip。回到主函数。

     这里模拟正常函数的调用过程。执行本上下文的eip时，相当于从一个子函数中返回，
     这时候的栈顶应该是esp+4,即跳过eip和恢复ebp的过程。
    */
    leal    4(%esp), %ecx       /* Exclude the return address.  */
    movl    %ecx, oESP(%eax)
    movl    %ebx, oEBX(%eax)

    xorl    %edx, %edx
    movw    %fs, %dx
    movl    %edx, oFS(%eax)
```

整个代码下来，对照一开始的结构体。对号入座。这里提一下子函数的调用过程一般是
1 主函数入栈参数
2 call 执行子函数压入eip
3 子函数保存ebp，设置新的esp
4 恢复ebp和esp
5 ret 弹回eip返回到主函数
6 主函数恢复栈，即清除1中的入栈的参数

继续

```text
    // 取得ucontext结构体的uc_sigmask字段的地址
    leal    oSIGMASK(%eax), %edx
    // ecx清0
    xorl    %ecx, %ecx
    // 准备调用系统调用，设置系统调用的入参，ebx是第一个参数，ecx是第二个，edx是第三个
    movl    $SIG_BLOCK, %ebx
    // 调用系统调用sigprocmask，SIG_BLOCK是表示设置屏蔽信号，见sigprocmask函数解释，eax保存系统调用的调用号
    movl    $__NR_sigprocmask, %eax
    // 通过中断触发系统调用
    int $0x80
```

这里是设置信号屏蔽的逻辑。我们看看该函数的声明。

```text
// how操作类型（这里是设置屏蔽信号），设置信号屏蔽位为set，保存旧的信号屏蔽位oldset
int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);
```

所以根据上面的代码，翻译过来就是。

```text
int sigprocmask(SIG_BLOCK, 0, &ucontext.uc_sigmask);
```

即保存旧的信号屏蔽信息。
getcontext函数大致逻辑就是上面。主要做了两个事情。
1 保存上下文。
2 保存旧的信号屏蔽信息

## <span id="head6">**2 设置上下文信息makecontext**</span>

makecontext是设置上下文的某些字段的信息

```text
// ucontext_t结构体的地址
movl    4(%esp), %eax
// 函数地址，即协程的工作函数，类似线程的工作函数
movl    8(%esp), %ecx
// 设置ucontext_t的eip字段的值为函数的值
movl    %ecx, oEIP(%eax)
// 赋值ucontext_t.uc_stack.ss_sp（栈顶）给edx
movl    oSS_SP(%eax), %edx
// oSS_SIZE为栈大小，这里设置edx指向栈底
addl    oSS_SIZE(%eax), %edx
```

这时候的布局如下。

![img](/Users/zhangjunmei/work/%E5%8D%8F%E7%A8%8B.assets/v2-790f7acb53208ed6fc6bfbc4359b8e94_1440w.jpg)

```text
    movl    12(%esp), %ecx
    movl    %ecx, oEBX(%eax)
```

保存makecontext的第三个参数（表示参数个数），到eax。

```text
    // 取负
    negl    %ecx
    // edx - ecx * 4 - 4。ecx * 4代表ecx个参数需要的空间，再减去4是保存oLINK的值（ucontext_t.ucontext的值）
    leal    -4(%edx,%ecx,4), %edx
    // 恢复ecx的值
    negl    %ecx
    // 栈顶减4，即可以存储多一个数据，用于保存L(exitcode)的地址，见下面的L(exitcode)
    subl    $4, %ed
    // 保存栈顶地址到ucontext_t
    movl    %edx, oESP(%eax)
    // 把ucontext_t.uc_link的内存复制到栈中的第一个元素
    movl    oLINK(%eax), %eax
    // edx + ecx * 4 + 4指向保存ucontext_t.ucontext的值的内存地址。即保存ucontext_t.ucontext到该内存里
    movl    %eax, 4(%edx,%ecx,4)
    // ecx（参数个数）为0则跳到2,说明不需要复制参数
    jecxz   2f
// 循环复制参数
1:    movl    12(%esp,%ecx,4), %eax
    movl    %eax, (%edx,%ecx,4)
    decl    %ecx
    jnz 1b
    // 把L(exitcode)的地址压入栈。L(exitcode)的内容下面继续分析
    movl    $L(exitcode), (%edx)
    // makecontext返回
    ret
```

这时候的栈布局如下

![img](/Users/zhangjunmei/work/%E5%8D%8F%E7%A8%8B.assets/v2-9a2460bfe43c0af9bf0a314adf0129a0_1440w.jpg)



从上面的代码中我们知道，makecontext函数主要的功能是

1 设置协程的工作函数地址到上下文（ucontext_t）中。

2 在用户设置的栈上保存一些信息，并且设置栈顶指针的值到上下文中。



## <span id="head7">**3 设置当前执行的上下文setcontext**</span>

setcontext是设置当前执行上下文。

```text
movl    4(%esp), %eax
```

把当前需要执行的上下文（ucontext_t）赋值给eax。

```text
    xorl    %edx, %edx
    leal    oSIGMASK(%eax), %ecx
    movl    $SIG_SETMASK, %ebx
    movl    $__NR_sigprocmask, %eax
    int $0x80
```

这里是用getcontext里保存的信息，设置信号屏蔽位。

```text
    // 设置fs寄存器
    movl    oFS(%eax), %ecx
    movw    %cx, %fs

    // 根据上下文设置栈顶，这个栈顶的值就是在makecontext里设置的（见上面的图）
    movl    oESP(%eax), %esp
    // 把eip压入栈，setcontext返回的时候，从eip开始执行。eip在makecontext中设置，即工作函数的地址
    movl    oEIP(%eax), %ecx
    // 把工作函数的地址入栈
    pushl   %ecx
```

这时候的栈布局

![img](/Users/zhangjunmei/work/%E5%8D%8F%E7%A8%8B.assets/v2-8d0c1b929028f2817955612b5710fcc1_1440w.jpg)

```text
    // 根据上下文设置其他寄存器
    movl    oEDI(%eax), %edi
    movl    oESI(%eax), %esi
    movl    oEBP(%eax), %ebp
    movl    oEBX(%eax), %ebx
    movl    oEDX(%eax), %edx
    movl    oECX(%eax), %ecx
    movl    oEAX(%eax), %eax
    // setcontext返回
    ret
```

然后setcontext函数返回。ret指令会把当前栈顶的元素出栈，赋值给eip。即下一条要执行的指令的地址。我们从上图可以知道，栈顶这时候指向的元素是上下文的工作函数的地址。所以setcontext返回后，执行设置的上下文的工作函数。
这时候的栈布局

![img](/Users/zhangjunmei/work/%E5%8D%8F%E7%A8%8B.assets/v2-cbe2865ccff5a9d10880f82501df607b_1440w.jpg)



当工作函数执行完之后，同样，栈顶的元素出栈，成为下一个eip。即L(exitcode)地址对应的指令会在工作函数执行完后执行。下面我们分析L(exitcode)。



```text
L(exitcode):
    // 工作函数执行完了，他的入参也不需要了，释放栈空间。栈布局见下图
    leal    (%esp,%ebx,4), %esp
```

这时候的栈布局

![img](/Users/zhangjunmei/work/%E5%8D%8F%E7%A8%8B.assets/v2-b97f0f4cab296e98df50bdf2567bd2ae_1440w.jpg)



接着

```text
    // 这时候的栈顶指向ucontext_t.uc_link的值，即下一个要执行的协程。
    cmpl    $0, (%esp) 
    // 如果没有要执行的协程。则跳到2正常退出  
    je  2f          /* If it is zero exit.  */
    // 否则继续setcontext，入参是上图esp指向的ucontext_t.uc_link
    call    HIDDEN_JUMPTARGET(__setcontext)
    // setcontext返回后会从新的eip开始执行，如果执行下面的指令说明setcontext执行出错了。调用exit退出
    jmp L(call_exit)

2:
    /* Exit with status 0.  */
    xorl    %eax, %eax
```

## <span id="head8">**4 交换上下文swapcontext**</span>

swapcontext函数把当前执行的上下文保存到第一个参数中，然后设置第二个参数为当前执行上下文。

```text
    // 把第一个参数的地址赋值给eax
    movl    4(%esp), %eax
    movl    $0, oEAX(%eax)
    // 保存当前执行上下文
    movl    %ecx, oECX(%eax)
    movl    %edx, oEDX(%eax)
    movl    %edi, oEDI(%eax)
    movl    %esi, oESI(%eax)
    movl    %ebp, oEBP(%eax)
    movl    %ebx, oEBX(%eax)

    // esp指向的内存保存了swapcontext函数下一条指令的地址，保存到上下文的eip字段中
    movl    (%esp), %ecx
    movl    %ecx, oEIP(%eax)
    // 保存栈到上下文。模拟正常函数的调用过程。见getcontext的分析
    leal    4(%esp), %ecx
    movl    %ecx, oESP(%eax)

    // 保存fs寄存器
    xorl    %edx, %edx
    movw    %fs, %dx
    movl    %edx, oFS(%eax)
```

swapcontext首先是保存当前执行上下文到第一个参数中。

```text
// 把swapcontext的第二个参数赋值给ecx
movl    8(%esp), %ecx
// 把旧的信号屏蔽位信息保存到swapcontext的第一个参数中，设置信号屏蔽位为swapcontext的第二个参数中的值
leal    oSIGMASK(%eax), %edx
leal    oSIGMASK(%ecx), %ecx
movl    $SIG_SETMASK, %ebx
movl    $__NR_sigprocmask, %eax
int $0x80
```

然后设置新的执行上下文

```text
    // 设置fs寄存器
    movl    oFS(%eax), %edx
    movw    %dx, %fs
    // 设置栈顶
    movl    oESP(%eax), %esp
    // 即将执行的上下文的eip压入栈，swapcontext函数返回的时候从这个开始执行（工作函数）
    movl    oEIP(%eax), %ecx
    pushl   %ecx
    // 设置其他寄存器
    movl    oEDI(%eax), %edi
    movl    oESI(%eax), %esi
    movl    oEBP(%eax), %ebp
    movl    oEBX(%eax), %ebx
    movl    oEDX(%eax), %edx
    movl    oECX(%eax), %ecx
    movl    oEAX(%eax), %eax
```

四个函数分析完了，主要的工作是对寄存器的一些保存和设置，实现任意跳转。

最后我们看一下例子。

```text
#include <ucontext.h>
#include <stdio.h>
#include <stdlib.h>

static ucontext_t uctx_main, uctx_func1, uctx_func2;

#define handle_error(msg) \
do { perror(msg); exit(EXIT_FAILURE); } while (0)

static void
func1(void)
{
	if (swapcontext(&uctx_func1, &uctx_func2) == -1)
		handle_error("swapcontext");
}

static void
func2(void)
{
	if (swapcontext(&uctx_func2, &uctx_func1) == -1)
		handle_error("swapcontext");
}

int
main(int argc, char *argv[])
{
    char func1_stack[16384];
    char func2_stack[16384];
    // 保存当前的执行上下文
    if (getcontext(&uctx_func1) == -1)
    	handle_error("getcontext");
    // 设置新的栈
    uctx_func1.uc_stack.ss_sp = func1_stack;
    uctx_func1.uc_stack.ss_size = sizeof(func1_stack);
    // uctx_func1对应的协程执行完执行uctx_main
    uctx_func1.uc_link = &uctx_main;
    // 设置协作的工作函数
    makecontext(&uctx_func1, func1, 0);
    // 同上
    if (getcontext(&uctx_func2) == -1)
    	handle_error("getcontext");
    uctx_func2.uc_stack.ss_sp = func2_stack;
    uctx_func2.uc_stack.ss_size = sizeof(func2_stack);
    // uctx_func2执行完执行uctx_func1
    uctx_func2.uc_link = (argc > 1) ? NULL : &uctx_func1;
    makecontext(&uctx_func2, func2, 0);
    // 保存当前执行上下文到uctx_main，然后开始执行uctx_func2对应的上下文
    if (swapcontext(&uctx_main, &uctx_func2) == -1)
    	handle_error("swapcontext");

    printf("main: exiting\n");
    exit(EXIT_SUCCESS);
}
```

所以整个流程是uctx_func2->uctx_func1->uctx_main
最后执行

```text
printf("main: exiting\n");
exit(EXIT_SUCCESS);
```

然后退出。

在Mac上编译运行：

`gcc -g -D _XOPEN_SOURCE -o myCtx myContext.c`

# <span id="head9"> 协程实现</span>

## <span id="head10"> 从云风协程库理解协程的机制</span>

风云的协程库是使用C语言实现的使用共享栈的一个非对称协程库，即调用者和被调用者的关系是固定的，协程A调用B，则B完成后必定返回到A。因为云风不希望使用他的协程库的人太考虑栈的大小，并且认为，进行上下文切换的大多时候，栈的使用实际并不大，所以使用共享栈每次进行上下文切换时拷贝的开销其实可以接受。另外，该库的上下文切换使用的是glibc的ucontext。

### <span id="head11"> 协程调度器数据结构</span>

```code
// 记录协程公共数据的结构体
struct schedule {
    // 协程的公共栈
    char stack[STACK_SIZE];
    // 主上下文，不是协程的
    ucontext_t main;
    // 已使用的个数
    int nco;
    // 最大容量
    int cap;
    // 记录哪个协程在执行
    int running;
    // 协程信息
    struct coroutine **co;
};
```

### <span id="head12"> 协程数据结构</span>

和进程一样，协程可以用一个结构体来表示。看看协程的表示。

```text
// 协程的表示
struct coroutine {
    // 协程任务函数
    coroutine_func func;
    // 用户数据，执行func的时候传入
    void *ud;
    // 保存执行前的上下文
    ucontext_t ctx;
    // 所属schedule 
    struct schedule * sch;
    // 当前栈的最大容量
    ptrdiff_t cap;
    // 当前栈的栈大小（已使用的大小）
    ptrdiff_t size;
    // 协程状态
    int status;
    // 协程的栈顶指针
    char *stack;
};
```

### <span id="head13"> 具体实现</span>

#### <span id="head14"> 首先是创建一个schedule:</span>

```text
// 创建一个调度器，准备开始执行协程
struct schedule * 
coroutine_open(void) {
    struct schedule *S = malloc(sizeof(*S));
    // 协程个数
    S->nco = 0;
    // 最大协程数
    S->cap = DEFAULT_COROUTINE;
    // 哪个协程在跑，初始化时还没有协程在跑
    S->running = -1;
    // 分申请了一个结构体指针数组，指向协程结构体的信息
    S->co = malloc(sizeof(struct coroutine *) * S->cap);
    memset(S->co, 0, sizeof(struct coroutine *) * S->cap);
    return S;
}
```

内存视图：

![](/Users/zhangjunmei/work/%E5%8D%8F%E7%A8%8B.assets/v2-8092681ec5014f9d3640bfb5e56baa54_1440w.jpg)

#### <span id="head15"> 创建一个协程:</span>

```text
// 申请一个表示协程的结构体
struct coroutine *_co_new(struct schedule *S , coroutine_func func, void *ud) {
    // 在堆上申请空间
    struct coroutine * co = malloc(sizeof(*co));
    co->func = func;
    co->ud = ud;
    co->sch = S;
    co->cap = 0;
    co->size = 0;
    co->status = COROUTINE_READY;
    co->stack = NULL;
    return co;
}
// 新建一个协程
int coroutine_new(struct schedule *S, coroutine_func func, void *ud) {
    // 申请一个协程结构体
    struct coroutine *co = _co_new(S, func , ud);
    // 协程数达到上限，扩容
    if (S->nco >= S->cap) {
        int id = S->cap;
        // 扩容两倍
        S->co = realloc(S->co, S->cap * 2 * sizeof(struct coroutine *));
        // 初始化空闲的内存
        memset(S->co + S->cap , 0 , sizeof(struct coroutine *) * S->cap);
        S->co[S->cap] = co;
        S->cap *= 2;
        ++S->nco;
        return id;
    } else {
        int i;
        // 有些slot可能是空的，这里从最后一个索引开始找，没找到再从头开始找
        for (i=0;i<S->cap;i++) {
            int id = (i+S->nco) % S->cap;
            // 找到可用的slot，记录协程信息
            if (S->co[id] == NULL) {
                S->co[id] = co;
                // 记录已使用的slot个数，即协程数
                ++S->nco;
                return id;
            }
        }
    }
    assert(0);
    return -1;
}
```

新的内存结构：

![](/Users/zhangjunmei/work/%E5%8D%8F%E7%A8%8B.assets/v2-10f6c7731a69a48d94b1cd647abf6602_1440w.jpg)

这样就完成了协程的创建。接下来就是执行协程。

#### <span id="head16"> 设置协程必要信息</span>

```text
struct coroutine *C = S->co[id]
```

首先拿到当前需要执行协程的结构体。id是创建协程的时候返回的。接着保存当前执行的上下文。

```text
// 保存当前执行的上下文到ctx
getcontext(&C->ctx);
```

getcontext函数在之前的文章分析过，他主要是保存当前执行的上下文，即getcontext函数下一条执行的地址和寄存器等信息。接着设置协程的信息。

```text
// 设置协程执行时的栈信息，真正的esp在makecontext里会修改成ss_sp+ss_size-一定的大小（用于存储额外数据的）
C->ctx.uc_stack.ss_sp = S->stack;
C->ctx.uc_stack.ss_size = STACK_SIZE;
// 记录下一个协程，即执行完执行他
C->ctx.uc_link = &S->main;
// 记录当前执行的协程
S->running = id;
// 协程开始执行
C->status = COROUTINE_RUNNING;
// 协程执行时的入参
uintptr_t ptr = (uintptr_t)S;
```

继续设置协程的信息。

```text
// 设置协程（ctx）的任务函数和入参，makecontext的入参是32位，在64位系统上有问题，所以兼容处理一下，把64位分为高低地址两部分
makecontext(&C->ctx, (void (*)(void)) mainfunc, 2, (uint32_t)ptr, (uint32_t)(ptr>>32));
```

makecontext是设置协程的任务函数是mainfunc函数（即设置eip寄存器为mainfunc的地址），后面的几个参数是执行mainfunc的入参。设置完了，开始执行。

#### <span id="head17"> 启动协程</span>

```text
// 保存当前上下文到main，然后切换到ctx对应的上下文执行，即执行上面设置的mainfunc，执行完再切回这里
swapcontext(&S->main, &C->ctx);
```

swapcontext函数首先保存当前执行的上下文到main字段，然后切换到ctx中执行。<u>**这样就启动了一个协程**</u>。假设协程的工作函数如下:

```text
static void foo(struct schedule * S, void *ud) {
    struct args * arg = ud;
    int start = arg->n;
    int i;
    for (i=0;i<5;i++) {
        printf("coroutine %d : %d\n",coroutine_running(S) , start + i);
        coroutine_yield(S);
    }
}
```

#### <span id="head18"> 让出执行</span>

协程执行到一个地方，执行coroutine_yield函数让出执行权。实现协程的切换，我们看看coroutine_yield的实现。

```text
// 协程主动让出执行权，切换到main
void coroutine_yield(struct schedule * S) {
    int id = S->running;
    assert(id >= 0);
    struct coroutine * C = S->co[id];
    assert((char *)&C > S->stack);
    _save_stack(C,S->stack + STACK_SIZE);
    C->status = COROUTINE_SUSPEND;
    // 当前协程已经让出执行权，当前没有协程执行
    S->running = -1;
    // 切换到main执行
    swapcontext(&C->ctx , &S->main);
}
```

其中最重要的是_save_stack函数。从前面的代码中我们知道，协程执行的时候使用的是一个公共的栈，即所有协程共享的。那么如果协程让出执行权后，其他协程执行时就会覆盖栈里的信息。所以让出执行权之前，协程有个很重要的事情要做，那就是保存当前栈的信息，以便再次执行时，能恢复栈的上下文。我们看看_save_stack的实现。

```text
// 保存当前栈信息，top是协程的栈顶最大值
static void
_save_stack(struct coroutine *C, char *top) {
    // dummy用于计算出当前的esp，即栈顶地址, 这里很巧妙了，第一个变量的地址就是栈顶地址
    char dummy = 0;
    assert(top - &dummy <= STACK_SIZE);
    // top-&dummy算出协程当前的栈上下文有多大，如果比当前的容量大，则需要扩容
    if (C->cap < top - &dummy) {
        free(C->stack);
        C->cap = top-&dummy;
        C->stack = malloc(C->cap);
    }
    // 记录当前的栈大小
    C->size = top - &dummy;
    // 复制公共栈的数据到私有栈
    memcpy(C->stack, &dummy, C->size);
}
```

这个函数的实现比较巧妙。假设当前的栈布局如下:

![](/Users/zhangjunmei/work/%E5%8D%8F%E7%A8%8B.assets/v2-4cf9418ac65bab42276743868c04968b_1440w.jpg)

从图中我们可以知道dummy变量的地址之前的（即高地址到dummy地址部分）是当前协程用到的栈空间（栈的上下文）。而这个栈是公共的栈，即其他协程也会使用。那么当前协程让出执行权后，需要保存这部分上下文，否则他就被覆盖了。做法就是在堆上申请一块空间（如果还没有或者大小不够的话）。然后保存公共栈里的上下文。这样其他协程执行的时候就可以覆盖里面的数据了。布局如下。

![img](/Users/zhangjunmei/work/%E5%8D%8F%E7%A8%8B.assets/v2-7195dd1e6b40534acaf50118dedbfa96_1440w.jpg)



#### <span id="head19"> 恢复</span>

然后等到该协程再被执行时，我们看看是怎么恢复这个栈上下文的。

```text
        memcpy(S->stack + STACK_SIZE - C->size, C->stack, C->size);
        S->running = id;
        C->status = COROUTINE_RUNNING;
        swapcontext(&S->main, &C->ctx);
```

就把把私有栈C->stack开始，大小为C->size的字节复制到公共栈中。达到恢复上下文的目的

![img](/Users/zhangjunmei/work/%E5%8D%8F%E7%A8%8B.assets/v2-dbb76832c863627ed6aeb84ac848d386_1440w.jpg)



然后调用swapcontext继续执行协程，即从上次挂起的地方继续执行。

总结：本文大致分析了一个协程的创建，执行，挂起，恢复执行的过程。





