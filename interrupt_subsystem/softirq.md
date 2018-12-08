## **softirq机制**

对于中断处理而言，linux将其分成了两个部分, 一个叫做中断handler(top half), 是全程关闭中断的, 主要处理中断的一些实时性任务, 另外一部分是deferable task(bottom half), 处理不那么紧急需要处理的事情。在执行bottom half的时候，是开中断的。bottom half的机制主要有softirq, tasklet和workqueue, 这三者有本质的区别：workqueue运行在process context，而softirq和tasklet运行在interrupt context.

softirq和hardirq是相对应的一个概念, 因此可以依据hardirq的机制来理解softirq, 不过softirq是一个纯软件的概念, 不需要任何硬件来参与, 故而没有和硬件相关的一些概念.比如hwirq id, irq_chip等等.

**1.softirq number**

softirq number是和hard irq中逻辑中断号相对应的一个概念, 用来标示一个softirq. 具体的softirq number定义如下:

```
路径: include/linux/interrupt.h
enum
{
        HI_SOFTIRQ=0,
        TIMER_SOFTIRQ,
        NET_TX_SOFTIRQ,
        NET_RX_SOFTIRQ,
        BLOCK_SOFTIRQ,
        IRQ_POLL_SOFTIRQ,
        TASKLET_SOFTIRQ,
        SCHED_SOFTIRQ,
        HRTIMER_SOFTIRQ, /* Unused, but kept as tools rely on the
                            numbering. Sigh! */
        RCU_SOFTIRQ,    /* Preferable RCU should always be the last softirq */

        NR_SOFTIRQS
};
```
softirq number是静态分配的, 这一点和hardirq 逻辑中断号的动态映射不同. 
* HI_SOFTIRQ 用于高优先级的tasklet.
* TIMER_SOFTIRQ 用于software timer.
* TASKLET_SOFTIRQ 用于低优先级的tasklet.

**2.what is softirq desc**

和hardirq一样, softirq也存在软中断描述符.相对于hardirq desc, softirq desc相当简单, 只有一个callback 函数. 具体如下:

```
路径: include/linux/interrupt.h
struct softirq_action
{
        void    (*action)(struct softirq_action *);
};

路径: kernel/softirq.c
static struct softirq_action softirq_vec[NR_SOFTIRQS] __cacheline_aligned_in_smp;
```
内核为每一个softirq定义了一个softirq_action结构, 用一个静态分配的结构体结构体数组softirq_vec来表示, 实现了softirq number和softirq desc的关联. 如果触发了某个softirq就调用相应的callback函数.

**3.what is softirq register**

由于softirq是一个纯软件的概念, 当触发了一个softirq时, 也需要一个"register"来记录该事件发生了, 以便在将来的某个时候可以调用某个处理函数. 在softirq中用irq_cpustat_t结构来模拟softirq register, 具体定义如下:

```
路径: arch/arm64/include/asm/hardirq.h
typedef struct {
	unsigned int __softirq_pending;
	unsigned int ipi_irqs[NR_IPI];
} ____cacheline_aligned irq_cpustat_t;

路径: kernel/softirq.c
#ifndef __ARCH_IRQ_STAT
irq_cpustat_t irq_stat[NR_CPUS] ____cacheline_aligned;
EXPORT_SYMBOL(irq_stat);
#endif
```
由上面定义的irq_stat的数组可知, 内核为每一个cpu都定义了一个irq_cpustat_t结构, 用于记录每一个cpu触发的softirq.

**4.how register softirq?**

softirq的注册是通过调用open_softirq()函数来实现的, 系统启动后, 在执行start_kernel()函数会调用softirq_init()函数注册tasklet, 但是最终调用函数是open_softirq(). 下面来看看该函数的实现:

```
路径: kernel/softirq.c
void open_softirq(int nr, void (*action)(struct softirq_action *))
{
        softirq_vec[nr].action = action;
}
```
由此可见: 注册softirq灰常简单, 就是设置对应softirq的回调函数.

**5.how trigger softirq?**

触发一个softirq就是设置softirq register. 内核提供了两个接口来触发softirq: raise_softirq()和raise_softirq_irqoff(). raise_softirq()是在raise_softirq_irqoff()的基础上添加了关闭本地cpu中断的功能. 具体如下:
```
路径: kernel/softirq.c
void raise_softirq(unsigned int nr)
{
        unsigned long flags;

        local_irq_save(flags);
        raise_softirq_irqoff(nr);
        local_irq_restore(flags);
}
void __raise_softirq_irqoff(unsigned int nr)
{
        trace_softirq_raise(nr);
        or_softirq_pending(1UL << nr);
}

路径: include/linux/interrupt.h
#define or_softirq_pending(x)  (local_softirq_pending() |= (x))

路径: include/linux/irq_cpustat.h
#ifndef __ARCH_IRQ_STAT
extern irq_cpustat_t irq_stat[];		/* defined in asm/hardirq.h */
#define __IRQ_STAT(cpu, member)	(irq_stat[cpu].member)
#endif

/* arch independent irq_stat fields */
#define local_softirq_pending() \
	__IRQ_STAT(smp_processor_id(), __softirq_pending)
```
由上面的代码可知, 所谓触发softirq就是设置irq_cpustat_t的成员__softirq_pending相应bit.

**5.when excute softirq action?**

softirq 的action在两种情况下会执行:
* 中断的上半部(top half)执行完成时, 会调用 __do_softirq()函数来执行本cpu上所有待决的softirq.
* 如果在第一种执行的时间超过2ms或者有更高优先级的softirq或者执行次数超过了10次,  则唤醒ksoftirqd线程来执行.

在中断top half执行完成后, 会调用irq_exit()函数来处理bottom half, 具体代码如下:
```
路径: 
void irq_exit(void)
{
#ifndef __ARCH_IRQ_EXIT_IRQS_DISABLED
        local_irq_disable();
#else
        WARN_ON_ONCE(!irqs_disabled());
#endif

        account_irq_exit_time(current);
        preempt_count_sub(HARDIRQ_OFFSET); // 设置相应标志, 退出IRQ context.
        if (!in_interrupt() && local_softirq_pending()) // 系统不处于IRQ context且有待决的softirq
                invoke_softirq();

        tick_irq_exit();
        rcu_irq_exit();
        trace_hardirq_exit(); /* must be last! */
}
```

```
路径: kernel/softirq.c
static inline void invoke_softirq(void)
{
        if (!force_irqthreads) { // 是否启用中断线程化
#ifdef CONFIG_HAVE_IRQ_EXIT_ON_IRQ_STACK
                /*
                 * We can safely execute softirq on the current stack if
                 * it is the irq stack, because it should be near empty
                 * at this stage.
                 */
                __do_softirq();
#else
                /*
                 * Otherwise, irq_exit() is called on the task stack that can
                 * be potentially deep already. So call softirq in its own stack
                 * to prevent from any overrun.
                 */
                do_softirq_own_stack();
#endif
        } else {
                wakeup_softirqd(); // 唤醒softirqd线程
        }
}
```

```
asmlinkage __visible void __softirq_entry __do_softirq(void)
{
        unsigned long end = jiffies + MAX_SOFTIRQ_TIME;
        unsigned long old_flags = current->flags;
        int max_restart = MAX_SOFTIRQ_RESTART;
        struct softirq_action *h;
        bool in_hardirq;
        __u32 deferred;
        __u32 pending;
        int softirq_bit;

        /*
         * Mask out PF_MEMALLOC s current task context is borrowed for the
         * softirq. A softirq handled such as network RX might set PF_MEMALLOC
         * again if the socket is related to swap
         */
        current->flags &= ~PF_MEMALLOC;

        pending = local_softirq_pending(); // 获取本cpu的_softirq_pending值
        deferred = softirq_deferred_for_rt(pending);
        account_irq_enter_time(current); 
        __local_bh_disable_ip(_RET_IP_, SOFTIRQ_OFFSET);
        in_hardirq = lockdep_softirq_start(); // 返回当前进程的hardirq_context并清除该标志, 设置当前进程的softirq_context标志加1, 表示进入softirq 上下文.

restart:
        /* Reset the pending bitmask before enabling irqs */
        set_softirq_pending(deferred); // 清除pengding
        __this_cpu_write(active_softirqs, pending); // 将获取的pending记录到active_softirqs中.

        local_irq_enable();  // 开启本cpu中断.

        h = softirq_vec;
        
        while ((softirq_bit = ffs(pending))) {
                 unsigned int vec_nr;
                 int prev_count;
 
                 h += softirq_bit - 1;
 
                 vec_nr = h - softirq_vec; // 计算softirq number
                 prev_count = preempt_count(); //获取current进程是否可抢占的状态
 
                 kstat_incr_softirqs_this_cpu(vec_nr); // 更新统计量
 
                 trace_softirq_entry(vec_nr);
                 h->action(h);	// 执行softirq 处理函数
                 trace_softirq_exit(vec_nr);
                 if (unlikely(prev_count != preempt_count())) { // current进程是否可抢占的状态发生了改变
                         pr_err("huh, entered softirq %u %s %p with preempt_count %08x, exited with %08x?\n"
                                vec_nr, softirq_to_name[vec_nr], h->action,
                                prev_count, preempt_count());
                         preempt_count_set(prev_count); // 设置回原来的状态.
                 }
                 h++;
                 pending >>= softirq_bit;
         }
 
         __this_cpu_write(active_softirqs, 0); //清除active_softirqs中设置的pending标志.
         rcu_bh_qs();
         local_irq_disable(); // 关闭本地cpu的中断
 
         pending = local_softirq_pending(); // 获取本地cpu的_softirq_pending值, 防止在开机中断后, 又发生了 hardirq, 从而再次进入该函数.
         deferred = softirq_deferred_for_rt(pending);
 
         if (pending) {
                 if (time_before(jiffies, end) && !need_resched() &&
                     --max_restart)
                         goto restart; // 如果满足进入该函数的时间小于2ms, 没有更高优先级的softirq需要调度且处理次数小于10次, 就重新开始处理.
         }
 
         if (pending | deferred) 
                 wakeup_softirqd(); // softirq交给本cpu的ksoftirqd进行处理.
         lockdep_softirq_end(in_hardirq);// 将当前进程的softirq_context标志减1, 同时恢复hardirq_context的值.
         account_irq_exit_time(current);
         __local_bh_enable(SOFTIRQ_OFFSET); // 使能bootom half
         WARN_ON_ONCE(in_interrupt());
         tsk_restore_flags(current, old_flags, PF_MEMALLOC);
 }
```


### **总结**

整个中断过程分为top half和bottom half两个部分, top half主要处理实时性任务, 而bottom half主要处理一些实行要求不是很高的任务. 在top half过程中本cpu的中断是关闭的, 而在bottom half总本地cpu的中断是开启. 一般top half处理比较快, 这样就提高了系统的实时性. bottom half 可以采用softirq形式, 其处理函数通常情况下在top half处理完成后即开始处理. 但是当bottom half处理时间大于2ms或者有更高优先级的softirq需要调度或者处理次数超过了10次, 就转给ksoftirqd内核线程处理.

### **参考博客**
[linux kernel的中断子系统之（八）：softirq](http://www.wowotech.net/linux_kenrel/soft-irq.html)
[Linux中断（interrupt）子系统之五：软件中断（softIRQ）](https://blog.csdn.net/droidphone/article/details/7518428)
