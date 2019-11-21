## 中断概念
https://blog.csdn.net/zqixiao_09/article/details/50908125
## GIC

## 中断号

## 中断处理过程

## 常用的中断驱动API



## Linux 中断处理机制
https://blog.csdn.net/weixin_42092278/article/details/81989449
https://www.cnblogs.com/linfeng-learning/p/9512866.html
https://www.cnblogs.com/chen-farsight/p/6155503.html
### 顶、底半部划分原则
#### 底半部实现机制
##### a.软中断
软中断（softirq）是一种传统的底半部处理机制，它的执行时机通常是顶半部返回的时候，在 linux 内核中用 softirq_action 结构体来表示一个软中断，这个结构体包含软中断函数指针和传递给该函数的参数。softirq_action 定义在```include/linux/interrupt.h``` 文件中。其中的action函数指针就是对应的软中断服务函数。
```C
struct softirq_action
{
	void	(*action)(struct softirq_action *);
};
```
在内核 ```kernel/softirq.c``` 文件中一共定义了 10 个软中断,softirq_vec 是个全局数组，因此所有的 CPU(对于 SMP 系统而言)都可以访问到，每个 CPU 都有自己的触发和控制机制，并且只执行自己所触发的软中断。但是各个 CPU 所执行的软中断服务函数确是相同的，都是数组 softirq_vec 中定义的 action 函数。
```C
static struct softirq_action softirq_vec[NR_SOFTIRQS];
```
所以要使用软中断，可以用open_softirq() 函数来注册对应的软中断处理函数，函数原型如下：
```C
void open_softirq(int nr, void (*action)(struct softirq_action *)) /* 注册软中断 */
```
函数中nr参数代表的是软中断类型（linux内核中一共定义了 10 中软中断类型，如下），action 为对应的中断处理函数。
```C
enum
{
	HI_SOFTIRQ=0,
	TIMER_SOFTIRQ,
	NET_TX_SOFTIRQ,
	NET_RX_SOFTIRQ,
	BLOCK_SOFTIRQ,
	BLOCK_IOPOLL_SOFTIRQ,
	TASKLET_SOFTIRQ,
	SCHED_SOFTIRQ,
	HRTIMER_SOFTIRQ,
	RCU_SOFTIRQ,    /* Preferable RCU should always be the last softirq */

	NR_SOFTIRQS //计数
};
```
注册好软中断以后需要通过 raise_softirq() 函数触发， raise_softirq() 函数原型如下：
```C
void raise_softirq(unsigned int nr) /* 触发软中断 */
```
##### b.Tasklet
Tasklet定义....
其类型结构同样定义在在```include/linux/interrupt.h``` 文件中，如下所示：
```C
struct tasklet_struct
{
	struct tasklet_struct *next;    /* 下一个 tasklet */
	unsigned long state;            /* tasklet 状态 */
	atomic_t count;                 /* tasklet 引用技术 */
	void (*func)(unsigned long);    /* tasklet 执行函数 */
	unsigned long data;             /* 函数func 的参数 */
};
```
其中 func 函数就是tasklet 要执行的中断处理函数。如果要使用tasklet，须先定义一个tasklet，然后使用tasklet_init() 函数初始化tasklet，task_init() 函数原型如下:
```C
void tasklet_init(struct tasklet_struct *t,void (*func)(unsigned long), unsigned long data);
```
参数 t 为定义的 tasklet 结构体，func 为 tasklet 处理函数，data 为要传递给 func 函数的参数。也可以使用宏 DECLARE_TASKLET 来一次性完成 tasklet 的定义和初始化，定义如下：
```C
#define DECLARE_TASKLET(name, func, data) struct tasklet_struct name = { NULL, 0, ATOMIC_INIT(0), func, data }
```
tasklet使用模板:
```C
/* 定义 tasklet */
struct tasklet_struct xxx_tasklet;

/* tasklet 中断处理底半部 */
void xxx_tasklet_func(unsigned long data)
{
    /* 中断处理具体操作 */
}

/* 中断处理顶半部 */
irqreturn xxx_interrupt(int irq, void *dev_id)
{
    ...
    task_schedule(&xxx_tasklet);//调用tasklet_schedule()函数使系统在适当的时候调度运行底半部
    ...
　　 return IRQ_HANDLED；
}

/* 设备驱动模块 init */
int __init xxx_init(void)
{
    ...
    /* 初始化 xxx_tasklet */
    DECLARE_TASKLET(xxx_tasklet, xxx_tasklet_func, data);
    /* 申请设备中断 */
    result = request_irq(xxx_irq, xxx_interrupt, IRQF_DISABLED, "xxx", NULL);
    ...
    return 0;
}

/* 设备驱动模块exit */
void __exit xxx_exit(void)
{
    ...
    /* 释放中断 */
    free_irq(xxx_irq, NULL);
}

module_init(xxx_init); 
module_exit(xxx_exit);
```
##### c.Workqueue(工作队列)
workqueue 定义...
Linux 内核使用 work_struct 结构体表示一个工作，定义在 ```include/linux/workqueue.h``` 文件中，其结构如下：
```C
struct work_struct {
	atomic_long_t data;         /* 函数 func 的参数 */
	struct list_head entry;
	work_func_t func;           /* 工作队列 执行函数 */
#ifdef CONFIG_LOCKDEP
	struct lockdep_map lockdep_map;
#endif
};
```
工作队列与 tasklet 使用方法非常类似，先定义一个 work_struct 变量，然后使用宏定义 INIT_WORK 来初始化。INIT_WORK原型如下：
```C
#define INIT_WORK(_work, _func)     __INIT_WORK((_work), (_func), 0)
```
其中 _work 表示定义的 work_struct 变量，_func 为工作对应的处理函数。
也可以使用宏定义 ```#define DECLARE_WORK(n, f)``` 一次性完成工作的创建和初始化,其中 n 表示定义的work_struct 变量，f 为工作对应的处理函数。
Workqueue 使用模板:
```C
/* 定义工作 */
struct work_struct xxx_work;

/* work中断处理底半部 */
void xxx_work_func(unsigned long)
{
　　/* do something */
}

/* 中断处理顶半部 */
irqreturn_t xxx_interrupt(int irq, void *dev_id)
{
　　...
　　schedule_work(&xxx_work);
　　...
　　return IRQ_HANDLED;
}

/* 设备驱动模块 init */
int __init xxx_init(void)
{
    ...
    /* 初始化工作队列 */
　　INIT_WORK(&xxx_work,xxx_work_func);

    /* 申请设备中断 */
    result = request_irq(xxx_irq, xxx_interrupt, IRQF_DISABLED, "xxx", NULL);
    ...
    return 0;
}

/* 设备驱动模块exit */
void __exit xxx_exit(void)
{
    ...
    /* 释放中断 */
    free_irq(xxx_irq, NULL);
}

module_init(xxx_init);
module_exit(xxx_exit);
```


## 参考文档
* [Linux内核中断系统处理机制-详细分析](https://blog.csdn.net/weixin_42092278/article/details/81989449)
* [Linux内核的中断机制](https://www.cnblogs.com/linfeng-learning/p/9512866.html)