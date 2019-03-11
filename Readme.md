# 最简协程实现	

参考Linux内核x86_64位体系结构中进程切换原理，实现的最简协程。

## 编译

键入make命令，可以编译出一个main程序。运行main程序获得结果。

## 原理

### 1.相关结构体

```c
typedef struct stack_frame {
	uint64_t r15;
	uint64_t r14;
	uint64_t r13;
	uint64_t r12;
	uint64_t rbx;
	uint64_t rbp;
	uint64_t ret;
}frame_t;

typedef void (*co_func)(void *);
typedef struct co_struct {
	uint64_t rsp;
	void *stack;
    int id;
	int exit;
	co_func func;
	void *data;
	struct co_struct *next;
}co_t;
```

`frame_t` 定义新建协程的初始化栈的结构。

`co_t` 定义一个协程，rsp字段用于保存协程的栈顶指针，stack保存协程的栈底指针，id表示协程id，exit表示协程退出，func是协程的处理函数，data是给func的参数，next指向下一个协程。

### 2.初始化

进程运行时，默认只有一个主线程，主线程的地址空间及栈由操作系统负责初始化了。

那这个主线程的栈，也同时会作为init协程的栈。

```c
// 初始协程, 标识主线程
static co_t init = {0,0,0,0,0,NULL};

//current 标识当前协程co_t
co_t *current=&init;
```

定义一个init协程，这里只需要把co_t的所有字段初始化为空即可，不需要初始化rsp及stack等字段，init协程的id=0。

current标识当前的协程，在初始化时，把current设置为init协程，那么在进程的main函数指向时，就可以使用current来读取协程id。

### 3.协程切换

协程使用类似Linux内核进程切换的接口：`switch_to(co_t *prev, co_t *next);`

```asm
.globl switch_to
switch_to:
	/*
	 * Save callee-saved registers
	 * This must match the order in inactive_task_frame
	 */
	pushq	%rbp
	pushq	%rbx
	pushq	%r12
	pushq	%r13
	pushq	%r14
	pushq	%r15

	/* switch stack */
	movq	%rsp, 0(%rdi)
	movq	0(%rsi), %rsp

	/* restore callee-saved registers */
	popq	%r15
	popq	%r14
	popq	%r13
	popq	%r12
	popq	%rbx
	popq	%rbp

	jmp	__switch_to
```

switch_to中先保存rbp到r15寄存器到当前栈上，当前栈也是prev协程的栈。

当前栈指的是：

- 如果当前是init协程，其当前栈就行进程中主线程的栈。
- 如果当前不是init协程，其当前栈就是协程自身的栈(co_t::stack)。

切换栈：

- 如果当前是init协程，则把rsp保证到init协程的`co_t::rsp`字段。
- 如果当前不是init协程，则把rsp保存到当前协程的`co_t::rsp`字段

然后把next协程的栈恢复。

- 如果下一个要运行的协程是init协程，则从init协程的`co_t::rsp`字段找到该协程的栈。
- 如果下一个要运行的协程不是init协程，则从next协程的`co_t::rsp`字段找到该协程的栈。

最后从next协程的栈上，恢复r15到rbp寄存器。

跳入\_\_switch_to。

```c
void __switch_to(co_t *prev, co_t *next)
{
    //赋值current, 切换当前协程
	current = next;
    //如果前一个协程执行完毕，则释放前一个协程的数据
	if(prev->exit) {
        co_t *c = &init;
        while(c->next != prev) c = c->next;
        c->next = prev->next;
		free(prev);
	}
}
```

`__switch_to`主要是执行一些清理工作，并修改current指向的协程。

切换到next协程后，就完全不会再使用prev协程的栈了，于是可以判断prev协程是否执行完毕，执行完毕，可以完全释放prev协程的结构及栈。

#### \_\_switch_to的返回

- 对应新创建的协程，\_\_switch_to返回后，会进入__new函数。
- 非新创建的协程，\_\_switch_to返回后，会返回到schedule()函数中。

为什么呢？可以参考新建协程部分的内容。

### 4.调度

```c
int schedule()
{
    /*
     * 选择下一个协程
     * 参考Linux内核的话，可以定义协程队列，并对每个协程定义优先级，
     * 在选择时，可以选择优先级高的协程先执行。
     * 这里最简处理。
    **/
	co_t *next = current->next;
    if(!next)
        next = &init;
    //协程切换
    switch_to(current, next);
    return (init.next != NULL);
}
```

最简单的是从选择current协程的下一个协程开始运行，执行switch_to切换到next协程。

schedule函数的返回值：1,还有其他协程，0,协程全部执行完毕。通过返回值可以确定main函数是否该退出了。

### 5.创建新协程

```c
static void __new()
{
    //调用协程函数
    current->func(current->data);
    //通过exit字段标识协程执行完毕
    current->exit = 1;
    //调度，切换到下一个协程
    schedule();
}

int cocreate(int stack_size, co_func f, void *d)
{
    static int co_id = 1;
    frame_t *frame;
    //分配新的协程co_t,并加入init队列中
	co_t *co = malloc(sizeof(co_t) + stack_size);
	co->stack = (void *)(co + 1);
    co->stack += stack_size;
    co->id = co_id++;
    co->exit = 0;
    co->func = f;
    co->data = d;
    co->next = init.next;
    init.next = co;
    
    /*
     * 这里的整个协程的核心
     * 要初始化新创建的栈，并初始化切换到新协程时要执行的函数
    **/
    frame = (frame_t *)co->stack;
    frame--;
    memset(frame, 0, sizeof(frame_t));
    frame->ret = (uint64_t)__new;  /* 核心中的核心 */
    co->rsp = (uint64_t)frame;
    return 0;
} 
```

通过cocreate创建新协程。只需要指定（栈大小，协程函数，参数）这三个参数即可。

1. 分配co_t结构及栈。
2. 初始化co_t结构，id字段使用持续递增的唯一id。
3. 加入协程链表，表头是init协程。
4. 初始化新协程的栈。主要是初始化rbp到r15的寄存器及frame->ret。

`frame->ret`是新创建协程开始运行的地方。

主要考虑，通过switch_to开始切换到新建的协程时的运行过程。

switch_to开始先保存rbp-r15寄存器到当前协程栈上，然后把rsp放到prev协程的co_t::rsp字段。然后把新建协程的co_t::rsp字段的值作为当前栈，然后弹出到r15-rbp寄存器。从源码可以看到新建协程的栈被清空了，所有r15-rbp都被初始化为0，此时新建协程的栈上还保留一个地址，就是frame->ret的值。

然后通过jmp指令跳入\_\_switch_to，当该函数返回时，就会弹出栈上的地址到rip，开始执行。可以看到\_\_switch_to返回后即开始执行\_\_new函数。

也即，开始执行新的协程。

### 6.协程id

```c
//返回当前协程id
int coid()
{
    return current->id;
}
```

通过coid返回协程的id，类似pthread_self。

### 7.main函数模型

```c
void main()
{
    cocreate(128*1024, f, NULL);
    while(schedule()) ;
}
```

在main函数中主要通过cocreate创建足够的协程后，通过while循环不断的进行调度即可，当schedule返回0时，意味着全部协程执行完了，进程退出。

## 更进一步

### 1.系统调用

可以把`read,write,sendmsg,connect,sleep`等阻塞的系统调用，全部实现为非阻塞版本，当这些系统调用返回EAGAIN时，立即调用schedule函数，调度到下一个协程上执行。

那么这些系统调用就会变成轮训方式，直到不再返回EAGAIN为止。是通过执行其他协程的方式来等待某文件描述符可以读，不再返回EAGAIN。

### 2.调度队列

可以实现特定优先级方式的调度队列，而不只是一个单链表。

### 3.其他体系结构

可以参考Linux内核中其他体系结构的switch_to的代码。