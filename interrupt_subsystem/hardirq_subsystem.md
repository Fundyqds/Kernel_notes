## **中断控制子系统** ##

### **一 相关结构体** ###

* **irq_domain**

```
struct irq_domain {
        struct list_head link;
        const char *name;
        const struct irq_domain_ops *ops;
        void *host_data;
        unsigned int flags;

        /* Optional data */
        struct fwnode_handle *fwnode;
        enum irq_domain_bus_token bus_token;
        struct irq_domain_chip_generic *gc;
#ifdef  CONFIG_IRQ_DOMAIN_HIERARCHY
        struct irq_domain *parent;
#endif

        /* reverse map data. The linear map gets appended to the irq_domain */
        irq_hw_number_t hwirq_max;
        unsigned int revmap_direct_max_irq;
        unsigned int revmap_size;
        struct radix_tree_root revmap_tree;
        unsigned int linear_revmap[];
};
```
link: 链表结构.
name: 名称.
ops: 回调函数集合.
host_data: 私有数据指针.
bus_token:
fwnode: 所属的firmware device node handler, 里面包含了该设备节点的类型.
gc:
parent: 指向父 domain.
hwirq_max: 该控制器支持的最大hw irq 数目. 
revmap_direct_max_irq: 支持最大直接映射的hw irq id.
revmap_size: 线性映射表的size.
revmap_tree: radix tree, 用于hw irq id的 radix tree map. 
linear_revmap: 线性映射的table.
 
 * **irq_domain_ops**
 
```
 struct irq_domain_ops {
        int (*match)(struct irq_domain *d, struct device_node *node,
                     enum irq_domain_bus_token bus_token);
        int (*select)(struct irq_domain *d, struct irq_fwspec *fwspec,
                      enum irq_domain_bus_token bus_token);
        int (*map)(struct irq_domain *d, unsigned int virq, irq_hw_number_t hw);
        void (*unmap)(struct irq_domain *d, unsigned int virq);
        int (*xlate)(struct irq_domain *d, struct device_node *node,
                     const u32 *intspec, unsigned int intsize,
                     unsigned long *out_hwirq, unsigned int *out_type);

#ifdef  CONFIG_IRQ_DOMAIN_HIERARCHY
        /* extended V2 interfaces to support hierarchy irq_domains */
        int (*alloc)(struct irq_domain *d, unsigned int virq,
                     unsigned int nr_irqs, void *arg);
        void (*free)(struct irq_domain *d, unsigned int virq,
                     unsigned int nr_irqs);
        void (*activate)(struct irq_domain *d, struct irq_data *irq_data);
        void (*deactivate)(struct irq_domain *d, struct irq_data *irq_data);
        int (*translate)(struct irq_domain *d, struct irq_fwspec *fwspec,
                         unsigned long *out_hwirq, unsigned int *out_type);
#endif
};
```
match(): 用于判断一个指定的interrupt controller（node参数）是否和一个irq domain匹配（d参数）, 如果匹配的话, 返回1.
select():  用于在domain匹配时, 进行一定的规则检测从而确定某个hw irq是否属于该domain.
map(): 用于创建hw irq id和irq number之间的映射关系.
unmap(): 作用和map()相反.
xlate(): 
alloc(): 用于进行一些相关结构的分配以及进行相应的结构的关联, 比如:irqchip, 
free():
active():
deactivate(): 
translate(): 用于将dts中配置的中断号依据中断控制器特点转化为根gic所对应的hw irq id以及获取中断触发类型.


* **irq_desc**
```
struct irq_desc {
        struct irq_common_data  irq_common_data;
        struct irq_data         irq_data;
        unsigned int __percpu   *kstat_irqs;
        irq_flow_handler_t      handle_irq;
#ifdef CONFIG_IRQ_PREFLOW_FASTEOI
        irq_preflow_handler_t   preflow_handler;
#endif
        struct irqaction        *action;        /* IRQ action list */
        unsigned int            status_use_accessors;
        unsigned int            core_internal_state__do_not_mess_with_it;
        unsigned int            depth;          /* nested irq disables */
        unsigned int            wake_depth;     /* nested wake enables */
        unsigned int            irq_count;      /* For detecting broken IRQs */
        unsigned long           last_unhandled; /* Aging timer for unhandled count */
        unsigned int            irqs_unhandled;
        atomic_t                threads_handled;
        int                     threads_handled_last;
        raw_spinlock_t          lock;
        struct cpumask          *percpu_enabled;
        const struct cpumask    *percpu_affinity;
#ifdef CONFIG_SMP
        const struct cpumask    *affinity_hint;
        struct irq_affinity_notify *affinity_notify;
#ifdef CONFIG_GENERIC_PENDING_IRQ
        cpumask_var_t           pending_mask;
#endif
#endif
        unsigned long           threads_oneshot;
        atomic_t                threads_active;
        wait_queue_head_t       wait_for_threads;
#ifdef CONFIG_PM_SLEEP
        unsigned int            nr_actions;
        unsigned int            no_suspend_depth;
        unsigned int            cond_suspend_depth;
        unsigned int            force_resume_depth;
#endif
#ifdef CONFIG_PROC_FS
        struct proc_dir_entry   *dir;
#endif
#ifdef CONFIG_SPARSE_IRQ
        struct rcu_head         rcu;
        struct kobject          kobj;
#endif
        int                     parent_irq;
        struct module           *owner;
        const char              *name;
} ____cacheline_internodealigned_in_smp;
```
irq_common_data:
irq_data: 指向该中断描述符的irq_data. 
kstat_irqs: irq的统计信息.
handle_irq: 上层的irq_events处理函数, 会进行一些信息的更新以及调用中断处理函数. 在分配irq_desc结构时由根gic的alloc()函数设定.
preflow_handler: 在调用flow_handler()之前处理的一些函数. 应该是钩子函数.
action: 中断处理函数链表.
status_use_accessors: 中断描述符的状态.

name: 
* **irq_chip**
```
struct irq_chip {
        struct device   *parent_device;
        const char      *name;
        unsigned int    (*irq_startup)(struct irq_data *data);
        void            (*irq_shutdown)(struct irq_data *data);
        void            (*irq_enable)(struct irq_data *data);
        void            (*irq_disable)(struct irq_data *data);

        void            (*irq_ack)(struct irq_data *data);
        void            (*irq_mask)(struct irq_data *data);
        void            (*irq_mask_ack)(struct irq_data *data);
        void            (*irq_unmask)(struct irq_data *data);
        void            (*irq_eoi)(struct irq_data *data);

        int             (*irq_set_affinity)(struct irq_data *data, const struct cpumask *dest, bool force);
        int             (*irq_retrigger)(struct irq_data *data);
        int             (*irq_set_type)(struct irq_data *data, unsigned int flow_type);
        int             (*irq_set_wake)(struct irq_data *data, unsigned int on);

        void            (*irq_bus_lock)(struct irq_data *data);
        void            (*irq_bus_sync_unlock)(struct irq_data *data);

        void            (*irq_cpu_online)(struct irq_data *data);
        void            (*irq_cpu_offline)(struct irq_data *data);

        void            (*irq_suspend)(struct irq_data *data);
        void            (*irq_resume)(struct irq_data *data);
        void            (*irq_pm_shutdown)(struct irq_data *data);

        void            (*irq_calc_mask)(struct irq_data *data);

        void            (*irq_print_chip)(struct irq_data *data, struct seq_file *p);
        int             (*irq_request_resources)(struct irq_data *data);
        void            (*irq_release_resources)(struct irq_data *data);

        void            (*irq_compose_msi_msg)(struct irq_data *data, struct msi_msg *msg);
        void            (*irq_write_msi_msg)(struct irq_data *data, struct msi_msg *msg);
 
        int             (*irq_get_irqchip_state)(struct irq_data *data, enum irqchip_irq_state which, bool *state);
        int             (*irq_set_irqchip_state)(struct irq_data *data, enum irqchip_irq_state which, bool state);
 
        int             (*irq_set_vcpu_affinity)(struct irq_data *data, void *vcpu_info);
 
        void            (*ipi_send_single)(struct irq_data *data, unsigned int cpu);
        void            (*ipi_send_mask)(struct irq_data *data, const struct cpumask *dest);
 
        unsigned long   flags;
 };
```
name: 中断控制器的名称. 
irq_startup(): start up 指定的irq domain上的HW interrupt ID。如果不设定的话，default会被设定为enable函数
irq_shutdown(): shutdown 指定的irq domain上的HW interrupt ID。如果不设定的话，default会被设定为disable函数
irq_enable(): enable指定的irq domain上的HW interrupt ID。如果不设定的话，default会被设定为unmask函数
irq_disable(): disable指定的irq domain上的HW interrupt ID。
irq_ack(): 和具体的硬件相关，有些中断控制器必须在Ack之后（清除pending的状态）才能接受到新的中断。
irq_mask(): mask指定的irq domain上的HW interrupt ID
irq_mask_ack(): mask并ack指定的irq domain上的HW interrupt ID。
irq_unmask(): mask指定的irq domain上的HW interrupt ID
irq_eoi(): 有些interrupt controler（例如GIC）提供了这样的寄存器接口，让CPU可以通知interrupt controller，它已经处理完一个中断
irq_set_affinity(): 在SMP的情况下，可以通过该callback函数设定CPU affinity
irq_retrigger(): 重新触发一次中断，一般用在中断丢失的场景下。如果硬件不支持retrigger，可以使用软件的方法。
irq_set_type(): 设定指定的irq domain上的HW interrupt ID的触发方式，电平触发还是边缘触发
irq_set_wake():和电源管理相关，用来enable/disable指定的interrupt source作为唤醒的条件。
irq_bus_lock():有些interrupt controller是连接到慢速总线上（例如一个i2c接口的IO expander芯片），在访问这些芯片的时候需要lock住那个慢速bus（只能有一个client在使用I2C bus）
irq_bus_sync_unlock(): unlock慢速总线
irq_suspend(): 电源管理相关的callback函数.
irq_resume ():
irq_pm_shutdown():
irq_calc_mask(): TODO
irq_print_chip(): /proc/interrupts中的信息显示.

* **irq_data**
```
struct irq_data {
        u32                     mask;
        unsigned int            irq;
        unsigned long           hwirq;
        struct irq_common_data  *common;
        struct irq_chip         *chip;
        struct irq_domain       *domain;
#ifdef  CONFIG_IRQ_DOMAIN_HIERARCHY
        struct irq_data         *parent_data;
#endif
        void                    *chip_data;
};
```
mask:
irq: 逻辑中断号, irq number.
hwirq: 硬件中断id, 和irq number存在一定的映射关系.
common: 
chip: 所属的irq_chip.
domain: 所属的irq_domain.
parent_data: 指向父节点的, 和该irq相关的irq_data.
chip_data: 

### **二 中断subsytem初始化**
* **什么是中断控制器**
	中断控制器就是在一个计算机系统中专门用来管理I/O中断的器件, 它的功能是接收外部中断源的中断请求, 并对中断请求进行处理后再向CPU发出中断请求, 然后则由CPU响应中断并进行处理. 在CPU的响应中断的过程中, 中断控制器仍然负责管理外部中断源的中断请求，从而实现中断的嵌套与禁止, 而如何对中断进行嵌套和禁止则与中断控制器的工作模式与状态有关.
	
	对于arm处理器, ARM公司为其设计了通用的中断控制器GIC(generic interrupt controller). 目前已经迭代到第4代, 但是目前主流arm处理用的还是gic-v3, 比如qcom公司的sdm845, sdm660等等. 
	
* **硬件中断号和逻辑中断号**

	**硬件中断号**(HW interrupt id):  硬件中断号是对于interrupt controller而言, interrupt controller 利用HW interrupt controller id来表示一个外设中断, 并且该中断号是对于interrupt controller来说, 还可以用于进行一定的硬件配置.
	
	**逻辑中断号**(irq number): 逻辑中断号是对于cpu而言的, 是一个虚拟的中断号, 和硬件没有关系.主要用于对硬件进行隔离, 逻辑中断号和中断描述符是关联的. 通过逻辑中断号可以唯一确定中断描述符, 从而找到相应的中断处理函数.
	
* **初始化流程**
	本文基于linux-4.9内核来叙述arm gic注册过程, 来叙述中断子系统的初始化.
	
	在系统运行到start_kernel函数时, 会调用early_irq_init()和init_IRQ()函数进行初始化中断子系统. 具体代码如下:
	
```
路径:init/main.c

asmlinkage __visible void __init start_kernel(void)
{
......
	early_irq_init();
	init_IRQ();
......
}
```
	
	在early_irq_init()函数中, 主要提前预分配一部分irq_desc, 并置位了allocated_irqs的相应bit, 并以irq number为index将分配的irq_desc插入到radix tree中. 系统中所有的irq_desc都是通过radix tree来管理的. (还有一种方式是通过数组来管理系统中所有的irq_desc, 也是以irq number为index). 具体代码如下:
	
	
```
路径: kernel/irq/irqdesc.c
int __init early_irq_init(void)
{
        int i, initcnt, node = first_online_node;
        struct irq_desc *desc;

        init_irq_default_affinity();

        /* Let arch update nr_irqs and return the nr of preallocated irqs */
        initcnt = arch_probe_nr_irqs();
        printk(KERN_INFO "NR_IRQS:%d nr_irqs:%d %d\n", NR_IRQS, nr_irqs, initcnt);

        if (WARN_ON(nr_irqs > IRQ_BITMAP_BITS))
                nr_irqs = IRQ_BITMAP_BITS;

        if (WARN_ON(initcnt > IRQ_BITMAP_BITS))
                initcnt = IRQ_BITMAP_BITS;

        if (initcnt > nr_irqs)
                nr_irqs = initcnt;

        for (i = 0; i < initcnt; i++) {
                desc = alloc_desc(i, node, 0, NULL, NULL);
                set_bit(i, allocated_irqs);
                irq_insert_desc(i, desc);
        }
        return arch_early_irq_init();
}
```

	在init_IRQ()函数中, 主要调用了irqchip_init()进行系统中部分interrupt controller的初始化工作. 传入到of_irq_init()函数中的参数__irqchip_of_table是在编译的时候链接到 __irqchip_of_table段中, 用IRQCHIP_DECLARE宏申明的. 具体代码如下:


```
路径: arch/arm64/kernel/irq.c
void __init init_IRQ(void)
{
        irqchip_init();
        if (!handle_arch_irq)
                panic("No interrupt controller found.");
}

路径: drivers/irqchip/irqchip.c
void __init irqchip_init(void)
{
        of_irq_init(__irqchip_of_table);
        acpi_probe_device_table(irqchip); // 暂时未弄清楚其作用.
}
```

在of_irq_init()函数中, 主要是遍历__irqchip_of_table数组, 依据设备树确定interrupt controller的父子关系, 然后首先初始化root interrupt controller, 然后初始化其子interrupt controller. 

```
void __init of_irq_init(const struct of_device_id *matches)
{
        const struct of_device_id *match;
        struct device_node *np, *parent = NULL;
        struct of_intc_desc *desc, *temp_desc;
        struct list_head intc_desc_list, intc_parent_list;

        INIT_LIST_HEAD(&intc_desc_list);
        INIT_LIST_HEAD(&intc_parent_list);

        for_each_matching_node_and_match(np, matches, &match) {
                if (!of_find_property(np, "interrupt-controller", NULL) ||
                                !of_device_is_available(np))
                        continue;

                if (WARN(!match->data, "of_irq_init: no init function for %s\n",
                         match->compatible))
                        continue;

                /*
                 * Here, we allocate and populate an of_intc_desc with the node
                 * pointer, interrupt-parent device_node etc.
                 */
                desc = kzalloc(sizeof(*desc), GFP_KERNEL);
                if (WARN_ON(!desc)) {
                        of_node_put(np);
                        goto err;
                }

                desc->irq_init_cb = match->data;
                desc->dev = of_node_get(np);
                desc->interrupt_parent = of_irq_find_parent(np);
                if (desc->interrupt_parent == np)                                                                                  
                         desc->interrupt_parent = NULL;
                 list_add_tail(&desc->list, &intc_desc_list);
         }
 
         /*
          * The root irq controller is the one without an interrupt-parent.
          * That one goes first, followed by the controllers that reference it,
          * followed by the ones that reference the 2nd level controllers, etc.
          */
         while (!list_empty(&intc_desc_list)) {
                 /*
                  * Process all controllers with the current 'parent'.
                  * First pass will be looking for NULL as the parent.
                  * The assumption is that NULL parent means a root controller.
                  */
                 list_for_each_entry_safe(desc, temp_desc, &intc_desc_list, list) {
                         int ret;
 
                         if (desc->interrupt_parent != parent)
                                 continue;
 
                         list_del(&desc->list);
 
                         of_node_set_flag(desc->dev, OF_POPULATED);
 
                         pr_debug("of_irq_init: init %s (%p), parent %p\n",
                                  desc->dev->full_name,
                                  desc->dev, desc->interrupt_parent);
                         ret = desc->irq_init_cb(desc->dev,
                                                 desc->interrupt_parent);
                         if (ret) {
                                 of_node_clear_flag(desc->dev, OF_POPULATED);
                                 kfree(desc);
                                 continue;
                         }
 
                         /*
                          * This one is now set up; add it to the parent list so
                          * its children can get processed in a subsequent pass.
                          */
                         list_add_tail(&desc->list, &intc_parent_list);
                 }
 
                 /* Get the next pending parent that might have children */
                 desc = list_first_entry_or_null(&intc_parent_list,
                                                 typeof(*desc), list);
                 if (!desc) {
                         pr_err("of_irq_init: children remain, but no parents\n");
                         break;
                 }
                 list_del(&desc->list);
                 parent = desc->dev;
                 kfree(desc);
         }
 
         list_for_each_entry_safe(desc, temp_desc, &intc_parent_list, list) {
                 list_del(&desc->list);
                 kfree(desc);
         }
 err:
         list_for_each_entry_safe(desc, temp_desc, &intc_desc_list, list) {
                 list_del(&desc->list);
                 of_node_put(desc->dev);
                 kfree(desc);
         }
 }
```
接下来看一下gic的初始化流程. 具体代码如下:
```
路径: drivers/irqchip/irq-gic-v3.c
 static int __init gic_of_init(struct device_node *node, struct device_node *parent)
 {
         void __iomem *dist_base;
         struct redist_region *rdist_regs;
         u64 redist_stride;
         u32 nr_redist_regions;
         int err, i, ignore_irqs_len;
         u32 ignore_restore_irqs[MAX_IRQS_IGNORE] = {0};
 
         dist_base = of_iomap(node, 0); // 进行ioremap, 便于读取gic相关寄存器.
         if (!dist_base) {
                 pr_err("%s: unable to map gic dist registers\n",
                         node->full_name);
                 return -ENXIO;
         }
 
         err = gic_validate_dist_version(dist_base); // 检查gic的版本是否合法.
         if (err) {
                 pr_err("%s: no distributor detected, giving up\n",
                         node->full_name);
                 goto out_unmap_dist;
         }
 
         if (of_property_read_u32(node, "#redistributor-regions", &nr_redist_regions))
                 nr_redist_regions = 1;
 
         rdist_regs = kzalloc(sizeof(*rdist_regs) * nr_redist_regions, GFP_KERNEL);
         if (!rdist_regs) {
                 err = -ENOMEM;
                 goto out_unmap_dist;
         }
 
         for (i = 0; i < nr_redist_regions; i++) {
                 struct resource res;
                 int ret;
 
                 ret = of_address_to_resource(node, 1 + i, &res);
                 rdist_regs[i].redist_base = of_iomap(node, 1 + i);
                 if (ret || !rdist_regs[i].redist_base) {
                         pr_err("%s: couldn't map region %d\n",
                                node->full_name, i);
                         err = -ENODEV;
                         goto out_unmap_rdist;
                 }
                 rdist_regs[i].phys_base = res.start;
         }
 
         if (of_property_read_u64(node, "redistributor-stride", &redist_stride))
                 redist_stride = 0;
 
         err = gic_init_bases(dist_base, rdist_regs, nr_redist_regions,
                              redist_stride, &node->fwnode);
         if (err)
                 goto out_unmap_rdist;
 
         gic_populate_ppi_partitions(node);
         gic_of_setup_kvm_info(node);
 
         ignore_irqs_len = of_property_read_variable_u32_array(node,
                                 "ignored-save-restore-irqs",
                                 ignore_restore_irqs,
                                 0, MAX_IRQS_IGNORE);
         for (i = 0; i < ignore_irqs_len; i++)
                 set_bit(ignore_restore_irqs[i], irqs_ignore_restore);
 
         return 0;
 
 out_unmap_rdist:
         for (i = 0; i < nr_redist_regions; i++)
                 if (rdist_regs[i].redist_base)
                         iounmap(rdist_regs[i].redist_base);
         kfree(rdist_regs);
 out_unmap_dist:
         iounmap(dist_base);
         return err;
 }
 
 IRQCHIP_DECLARE(gic_v3, "arm,gic-v3", gic_of_init);
```
在gic_of_init()函数中, 首先进行了设备树的解析, 合法性检查, 然后调用了gic_init_bases()函数, 进行gic的硬件初始化, 代码如下:

```
路径: drivers/irqchip/irq-gic-v3.c
static int __init gic_init_bases(void __iomem *dist_base,
                                 struct redist_region *rdist_regs,
                                 u32 nr_redist_regions,
                                 u64 redist_stride,
                                 struct fwnode_handle *handle)
{
        u32 typer;
        int gic_irqs;
        int err;

        if (!is_hyp_mode_available())
                static_key_slow_dec(&supports_deactivate);

        if (static_key_true(&supports_deactivate))
                pr_info("GIC: Using split EOI/Deactivate mode\n");

        gic_data.fwnode = handle;
        gic_data.dist_base = dist_base;
        gic_data.redist_regions = rdist_regs;
        gic_data.nr_redist_regions = nr_redist_regions;
        gic_data.redist_stride = redist_stride;

        gicv3_enable_quirks();

        /*
         * Find out how many interrupts are supported.
         * The GIC only supports up to 1020 interrupt sources (SGI+PPI+SPI)
         */
        typer = readl_relaxed(gic_data.dist_base + GICD_TYPER);
        gic_data.rdists.id_bits = GICD_TYPER_ID_BITS(typer);
        gic_irqs = GICD_TYPER_IRQS(typer);
        if (gic_irqs > 1020)
                gic_irqs = 1020;
        gic_data.irq_nr = gic_irqs;

        gic_data.domain = irq_domain_create_tree(handle, &gic_irq_domain_ops,
                                                 &gic_data);
        gic_data.rdists.rdist = alloc_percpu(typeof(*gic_data.rdists.rdist));

        if (WARN_ON(!gic_data.domain) || WARN_ON(!gic_data.rdists.rdist)) {
                err = -ENOMEM;
                 goto out_free;
         }
 
         set_handle_irq(gic_handle_irq);
 
         if (IS_ENABLED(CONFIG_ARM_GIC_V3_ITS) && gic_dist_supports_lpis() &&
             !IS_ENABLED(CONFIG_ARM_GIC_V3_ACL))
                 its_init(handle, &gic_data.rdists, gic_data.domain);
 
         gic_smp_init();
         gic_dist_init();
         gic_cpu_init();
         gic_cpu_pm_init();
 
         return 0;
 
 out_free:
         if (gic_data.domain)
                 irq_domain_remove(gic_data.domain);
         free_percpu(gic_data.rdists.rdist);
         return err;
 }
```
在gic_init_bases()函数中主要依据gic_of_init()中解析出来的信息, 填充了gic_data结构体. 比如:fwnode, 支持irq数目, 所属的domain等等, 并且设置了异常处理时的handler和部分相关硬件初始化.
接下来看看将gic注册到中断子系统中的函数rq_domain_create_tree(), 具体代码如下:

```
路径: include/linux/irqdomain.h
static inline struct irq_domain *irq_domain_create_tree(struct fwnode_handle *fwnode,
                                         const struct irq_domain_ops *ops,
                                         void *host_data)
{
        return __irq_domain_add(fwnode, 0, ~0, 0, ops, host_data);
}

路径: kernel/irq/irqdomain.c
struct irq_domain *__irq_domain_add(struct fwnode_handle *fwnode, int size,
                                    irq_hw_number_t hwirq_max, int direct_max,
                                    const struct irq_domain_ops *ops,
                                    void *host_data)
{
        struct device_node *of_node = to_of_node(fwnode);
        struct irq_domain *domain;

        domain = kzalloc_node(sizeof(*domain) + (sizeof(unsigned int) * size),
                              GFP_KERNEL, of_node_to_nid(of_node));
        if (WARN_ON(!domain))
                return NULL;

        of_node_get(of_node);

        /* Fill structure */
        INIT_RADIX_TREE(&domain->revmap_tree, GFP_KERNEL);
        domain->ops = ops;
        domain->host_data = host_data;
        domain->fwnode = fwnode; 
        domain->hwirq_max = hwirq_max;
        domain->revmap_size = size;
        domain->revmap_direct_max_irq = direct_max;
        irq_domain_check_hierarchy(domain);

        mutex_lock(&irq_domain_mutex);
        list_add(&domain->link, &irq_domain_list);
        mutex_unlock(&irq_domain_mutex);

        pr_debug("Added domain %s\n", domain->name);
        return domain;
}
EXPORT_SYMBOL_GPL(__irq_domain_add);
```
函数rq_domain_create_tree()最终调用了__irq_domain_add()函数, 在该函数中主要进行domain结构相关成员的初始化, 最后将该domain加入到全局链表irq_domain_list. 系统中所有的irq domain都会挂到该链表中.

* **中断描述符分配**
在解析dts并注册设备的过程中, 完成了中断描述符的分配工作.依据在基于dts中设备的注册过程, 可知, 设备在注册的过程中, 会调用到of_platform_device_create_pdata()函数, 最终会调用到of_device_alloc()函数.
```
路径: drivers/of/platform.c
 static int of_platform_bus_create(struct device_node *bus,
                                   const struct of_device_id *matches,
                                   const struct of_dev_auxdata *lookup,
                                   struct device *parent, bool strict)
{
	......
	dev = of_platform_device_create_pdata(bus, bus_id, platform_data, parent);
	......
}
路径: drivers/of/platform.c
static struct platform_device *of_platform_device_create_pdata(
                                        struct device_node *np,
                                        const char *bus_id,
                                        void *platform_data,
                                        struct device *parent)
{
	......
	dev = of_device_alloc(np, bus_id, parent);
	......
}

路径: drivers/of/platform.c
struct platform_device *of_device_alloc(struct device_node *np,
                                  const char *bus_id,
                                  struct device *parent)
{
        struct platform_device *dev;
        int rc, i, num_reg = 0, num_irq;
        struct resource *res, temp_res;

        dev = platform_device_alloc("", -1);
        if (!dev)
                return NULL;

        /* count the io and irq resources */
        while (of_address_to_resource(np, num_reg, &temp_res) == 0)
                num_reg++;
        num_irq = of_irq_count(np);

        /* Populate the resource table */
        if (num_irq || num_reg) {
                res = kzalloc(sizeof(*res) * (num_irq + num_reg), GFP_KERNEL);
                if (!res) {
                        platform_device_put(dev);
                        return NULL;
                }

                dev->num_resources = num_reg + num_irq;
                dev->resource = res;
                for (i = 0; i < num_reg; i++, res++) {
                        rc = of_address_to_resource(np, i, res);
                        WARN_ON(rc);
                }
                if (of_irq_to_resource_table(np, res, num_irq) != num_irq)
                        pr_debug("not all legacy IRQ resources mapped for %s\n",
                                 np->name);
        }

        dev->dev.of_node = of_node_get(np);
        dev->dev.fwnode = &np->fwnode;
         dev->dev.parent = parent ? : &platform_bus;
 
         if (bus_id)
                 dev_set_name(&dev->dev, "%s", bus_id);
         else
                 of_device_make_bus_id(&dev->dev);
 
         return dev;
 }
 EXPORT_SYMBOL(of_device_alloc);
 ```
 ```
 int of_irq_to_resource_table(struct device_node *dev, struct resource *res,
                int nr_irqs)
{
        int i;

        for (i = 0; i < nr_irqs; i++, res++)
                if (!of_irq_to_resource(dev, i, res))
                        break;

        return i;
}
EXPORT_SYMBOL_GPL(of_irq_to_resource_table);

int of_irq_to_resource(struct device_node *dev, int index, struct resource *r)
{
        int irq = irq_of_parse_and_map(dev, index);

        /* Only dereference the resource if both the
         * resource and the irq are valid. */
        if (r && irq) {
                const char *name = NULL;

                memset(r, 0, sizeof(*r));
                /*
                 * Get optional "interrupt-names" property to add a name
                 * to the resource.
                 */
                of_property_read_string_index(dev, "interrupt-names", index,
                                              &name);

                r->start = r->end = irq;
                r->flags = IORESOURCE_IRQ | irqd_get_trigger_type(irq_get_irq_data(irq));
                r->name = name ? name : of_node_full_name(dev);
        }

        return irq;
}
EXPORT_SYMBOL_GPL(of_irq_to_resource);
```
```
路径:drivers/of/irq.c
 unsigned int irq_of_parse_and_map(struct device_node *dev, int index)
 {
         struct of_phandle_args oirq;
 
         if (of_irq_parse_one(dev, index, &oirq)) 
                 return 0;
 
         return irq_create_of_mapping(&oirq);
 }
 EXPORT_SYMBOL_GPL(irq_of_parse_and_map);
```
```
路径:kernel/irq/irqdomain.c
unsigned int irq_create_of_mapping(struct of_phandle_args *irq_data)
{
        struct irq_fwspec fwspec;

        of_phandle_args_to_fwspec(irq_data, &fwspec); //通过irq_data转化为fwspec结构. 
        return irq_create_fwspec_mapping(&fwspec);
}
EXPORT_SYMBOL_GPL(irq_create_of_mapping);
```

```
路径:kernel/irq/irqdomain.c
unsigned int irq_create_fwspec_mapping(struct irq_fwspec *fwspec)
{
        struct irq_domain *domain;
        struct irq_data *irq_data;
        irq_hw_number_t hwirq;
        unsigned int type = IRQ_TYPE_NONE;
        int virq;

        if (fwspec->fwnode) {
                domain = irq_find_matching_fwspec(fwspec, DOMAIN_BUS_WIRED);// 依据fwspec的fwnode找到匹配的domain.
                if (!domain)
                        domain = irq_find_matching_fwspec(fwspec, DOMAIN_BUS_ANY);
        } else {
                domain = irq_default_domain;
        }

        if (!domain) {
                pr_warn("no irq domain found for %s !\n",
                        of_node_full_name(to_of_node(fwspec->fwnode)));
                return 0;
        }

        if (irq_domain_translate(domain, fwspec, &hwirq, &type)) // 调用domain的回调函数translate解析fwspec, 获取hwirq和type.
                return 0;

        /*
         * WARN if the irqchip returns a type with bits
         * outside the sense mask set and clear these bits.
         */
        if (WARN_ON(type & ~IRQ_TYPE_SENSE_MASK))
                type &= IRQ_TYPE_SENSE_MASK;

        /*
         * If we've already configured this interrupt,
         * don't do it again, or hell will break loose.
         */
        virq = irq_find_mapping(domain, hwirq); // 通过hwirq在该domain的直接映射表, 线性映射表以及radix tree中查找是否已经分配了对应的irq desc. 
        if (virq) {
                 /*
                  * If the trigger type is not specified or matches the
                  * current trigger type then we are done so return the
                  * interrupt number.
                  */
                 if (type == IRQ_TYPE_NONE || type == irq_get_trigger_type(virq))
                         return virq;
 
                 /*
                  * If the trigger type has not been set yet, then set
                  * it now and return the interrupt number.
                  */
                 if (irq_get_trigger_type(virq) == IRQ_TYPE_NONE) {
                         irq_data = irq_get_irq_data(virq);
                         if (!irq_data)
                                 return 0;
 
                         irqd_set_trigger_type(irq_data, type); 
                         return virq;
                 }
 
                 pr_warn("type mismatch, failed to map hwirq-%lu for %s!\n",
                         hwirq, of_node_full_name(to_of_node(fwspec->fwnode)));
                 return 0;
         }
 
         if (irq_domain_is_hierarchy(domain)) {  // domain 是否具有层级性
                 virq = irq_domain_alloc_irqs(domain, 1, NUMA_NO_NODE, fwspec); //分配irq_desc, 调用父domain及该domain的alloc函数初始化irq_data成员, 最后执行hwirq到irq number的映射.
                 if (virq <= 0)
                         return 0;
         } else {
                 /* Create mapping */
                 virq = irq_create_mapping(domain, hwirq);  
                 if (!virq)
                         return virq;
         }
 
         irq_data = irq_get_irq_data(virq);	 // 获取对应irq number的irq_data.
         if (!irq_data) {
                 if (irq_domain_is_hierarchy(domain))
                         irq_domain_free_irqs(virq, 1);
                 else
                         irq_dispose_mapping(virq);
                 return 0;
         }
 
         /* Store trigger type */
         irqd_set_trigger_type(irq_data, type); // 将中断触发类型保存在irq_data的common成员的state_use_accessors中.
 
         return virq;
 }
 EXPORT_SYMBOL_GPL(irq_create_fwspec_mapping);
```
* **中断调用过程**

在系统发生中断时, 系统的pc指针根据系统当前的状态跳转到指定的中断向量执行, 会进入下面el1_irq入口执行, 最终会调用handle_arch_irq()函数.
```
路径: arch/arm64/kernel/entry.S
         .macro  irq_handler
         ldr_l   x1, handle_arch_irq
         mov     x0, sp
         irq_stack_entry
         blr     x1
         irq_stack_exit
         .endm
         
el1_irq:
        kernel_entry 1
        enable_dbg
#ifdef CONFIG_TRACE_IRQFLAGS
        bl      trace_hardirqs_off
#endif

        irq_handler

#ifdef CONFIG_PREEMPT
        ldr     w24, [tsk, #TSK_TI_PREEMPT]     // get preempt count
        cbnz    w24, 1f                         // preempt count != 0
        ldr     x0, [tsk, #TSK_TI_FLAGS]        // get flags
        tbz     x0, #TIF_NEED_RESCHED, 1f       // needs rescheduling?
        bl      el1_preempt
1:
#endif
#ifdef CONFIG_TRACE_IRQFLAGS
        bl      trace_hardirqs_on
#endif
        kernel_exit 1
ENDPROC(el1_irq)
```

函数handle_arch_irq()函数是一个函数指针, 依据体系架构不同, 设置为不同的handle_irq(), 具体设置函数如下:

```
void __init set_handle_irq(void (*handle_irq)(struct pt_regs *))
{
        if (handle_arch_irq)
                return;

        handle_arch_irq = handle_irq;
}
```
对根interrupt controller为gic

```
static asmlinkage void __exception_irq_entry gic_handle_irq(struct pt_regs *regs)
{
        u32 irqnr;

        do {
                irqnr = gic_read_iar(); // 获取hw irq id.

                if (likely(irqnr > 15 && irqnr < 1020) || irqnr >= 8192) {
                        int err;

                        uncached_logk(LOGK_IRQ, (void *)(uintptr_t)irqnr);
                        if (static_key_true(&supports_deactivate))
                                gic_write_eoir(irqnr);

                        err = handle_domain_irq(gic_data.domain, irqnr, regs);
                        if (err) {
                                WARN_ONCE(true, "Unexpected interrupt received!\n");
                                if (static_key_true(&supports_deactivate)) {
                                        if (irqnr < 8192)
                                                gic_write_dir(irqnr);
                                } else {
                                        gic_write_eoir(irqnr);
                                }
                        }
                        continue;
                }
                if (irqnr < 16) {
                        uncached_logk(LOGK_IRQ, (void *)(uintptr_t)irqnr);
                        gic_write_eoir(irqnr);
                        if (static_key_true(&supports_deactivate))
                                gic_write_dir(irqnr);
#ifdef CONFIG_SMP
                        /*
                         * Unlike GICv2, we don't need an smp_rmb() here.
                         * The control dependency from gic_read_iar to
                         * the ISB in gic_write_eoir is enough to ensure
                         * that any shared data read by handle_IPI will
                         * be read after the ACK.
                         */
                        handle_IPI(irqnr, regs);
#else
                        WARN_ONCE(true, "Unexpected SGI received!\n");
#endif
                        continue;
                }
        } while (irqnr != ICC_IAR1_EL1_SPURIOUS);
}

```

* **中断处理函数注册过程**
