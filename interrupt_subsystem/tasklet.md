## **tasklet机制**

```
路径: include/linux/interrupt.h
struct tasklet_struct
{
        struct tasklet_struct *next;  	// 指向下一个tasklet结构体, 构成链表.
        unsigned long state;		// 状态, TASKLET_STATE_SCHED表示该tasklet已经被调度到某个cpu上, TASKLET_STATE_RUN表示该tasklet已经在运行.
        atomic_t count;			// 使能tasklet与否的计数, count等于0那么该tasklet是处于enable的，如果大于0，表示该tasklet是disable的.
        void (*func)(unsigned long);	// callback 函数
        unsigned long data;		// callback 函数的参数
};
```

每个cpu都会维护一个链表，将本cpu需要处理的tasklet管理起来
```
路径: kernel/softirq.c
struct tasklet_head {
        struct tasklet_struct *head;
        struct tasklet_struct **tail;
};

static DEFINE_PER_CPU(struct tasklet_head, tasklet_vec);
static DEFINE_PER_CPU(struct tasklet_head, tasklet_hi_vec);

```
```
路径:include/linux/interrupt.h
static inline void tasklet_schedule(struct tasklet_struct *t)
{
        if (!test_and_set_bit(TASKLET_STATE_SCHED, &t->state))
                __tasklet_schedule(t);
}
```

```
路径: kernel/softirq.c
void __tasklet_schedule(struct tasklet_struct *t)
{
        unsigned long flags;

        local_irq_save(flags);
        t->next = NULL;
        *__this_cpu_read(tasklet_vec.tail) = t;
        __this_cpu_write(tasklet_vec.tail, &(t->next));
        raise_softirq_irqoff(TASKLET_SOFTIRQ);
        local_irq_restore(flags);
}
EXPORT_SYMBOL(__tasklet_schedule);
```

```
void __init softirq_init(void)
{
        int cpu;

        for_each_possible_cpu(cpu) {
                per_cpu(tasklet_vec, cpu).tail =
                        &per_cpu(tasklet_vec, cpu).head; // 初始化链表
                per_cpu(tasklet_hi_vec, cpu).tail =
                        &per_cpu(tasklet_hi_vec, cpu).head; // 初始化链表
        }

        open_softirq(TASKLET_SOFTIRQ, tasklet_action); // 设置softirq number为TASKLET_SOFTIRQ的handler: tasklet_action
        open_softirq(HI_SOFTIRQ, tasklet_hi_action);
}
```
```
 static __latent_entropy void tasklet_action(struct softirq_action *a)
 {
         struct tasklet_struct *list;
 
         local_irq_disable();
         list = __this_cpu_read(tasklet_vec.head); // 获取本cpu的链表
         __this_cpu_write(tasklet_vec.head, NULL); // 清空本cpu的链表头
         __this_cpu_write(tasklet_vec.tail, this_cpu_ptr(&tasklet_vec.head)); // 设置本cpu的链表尾指向链表头.
         local_irq_enable(); // 开启本cpu的中断.
 
         while (list) {
                 struct tasklet_struct *t = list;
 
                 list = list->next; // 获取tasklet.
 
                 if (tasklet_trylock(t)) { // 判断tasklet是否处于运行状态, 没有则置为run状态.
                         if (!atomic_read(&t->count)) { // 判断tasklet是否处于使能状态
                                 if (!test_and_clear_bit(TASKLET_STATE_SCHED,
                                                         &t->state)) // 剔除tasklet没有调度的情况
                                         BUG();
                                 t->func(t->data);
                                 tasklet_unlock(t); // 清除tasklet的run状态
                                 continue;
                         }
                         tasklet_unlock(t);
                 }
 
                 local_irq_disable(); // 关闭本地中断.
                 t->next = NULL; //
                 *__this_cpu_read(tasklet_vec.tail) = t; // 设置list head指向t.
                 __this_cpu_write(tasklet_vec.tail, &(t->next)); // 设置list tail执行t->next.
                 __raise_softirq_irqoff(TASKLET_SOFTIRQ); // 再次触发softirq，等待下一个执行时机 
                 local_irq_enable(); //  开启本cpu的中断.
         }
 }
```