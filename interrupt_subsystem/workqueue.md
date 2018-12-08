
## **workqueue机制**

### **1. 数据结构**
```
struct worker_pool {
        spinlock_t              lock;    	// 自旋锁
        int                     cpu;		// 所属的cpu
        int                     node;           /* I: the associated node ID */
        int                     id;             	// pool ID
        unsigned int            flags;          /* X: flags */

        unsigned long           watchdog_ts;    /* L: watchdog timestamp */

        struct list_head        worklist;       /* L: list of pending works */
        int                     nr_workers;     /* L: total number of workers */

        /* nr_idle includes the ones off idle_list for rebinding */
        int                     nr_idle;        /* L: currently idle ones */

        struct list_head        idle_list;      /* X: list of idle workers */
        struct timer_list       idle_timer;     /* L: worker idle timeout */
        struct timer_list       mayday_timer;   /* L: SOS timer for workers */

        /* a workers is either on busy_hash or idle_list, or the manager */
        DECLARE_HASHTABLE(busy_hash, BUSY_WORKER_HASH_ORDER);
                                                /* L: hash of busy workers */

        /* see manage_workers() for details on the two manager mutexes */
        struct worker           *manager;       /* L: purely informational */
        struct mutex            attach_mutex;   /* attach/detach exclusion */
        struct list_head        workers;        /* A: attached workers */
        struct completion       *detach_completion; /* all workers detached */

        struct ida              worker_ida;     /* worker IDs for task name */

        struct workqueue_attrs  *attrs;         /* I: worker attributes */
        struct hlist_node       hash_node;      /* PL: unbound_pool_hash node */
        int                     refcnt;         /* PL: refcnt for unbound pools */

        /*
         * The current concurrency level.  As it's likely to be accessed
         * from other CPUs during try_to_wake_up(), put it in a separate
         * cacheline.
         */
        atomic_t                nr_running ____cacheline_aligned_in_smp;
 
         /*
          * Destruction of pool is sched-RCU protected to allow dereferences
          * from get_work_pool().
          */
         struct rcu_head         rcu;
 } ____cacheline_aligned_in_smp;
```

```
struct pool_workqueue {
        struct worker_pool      *pool;          // 指向所属的 worker_pool
        struct workqueue_struct *wq;            /* I: the owning workqueue */
        int                     work_color;     /* L: current color */
        int                     flush_color;    /* L: flushing color */
        int                     refcnt;         /* L: reference count */
        int                     nr_in_flight[WORK_NR_COLORS];
                                                /* L: nr of in_flight works */
        int                     nr_active;      /* L: nr of active works */
        int                     max_active;     /* L: max active works */
        struct list_head        delayed_works;  /* L: delayed works */
        struct list_head        pwqs_node;      /* WR: node on wq->pwqs */
        struct list_head        mayday_node;    /* MD: node on wq->maydays */

        /*
         * Release of unbound pwq is punted to system_wq.  See put_pwq()
         * and pwq_unbound_release_workfn() for details.  pool_workqueue
         * itself is also sched-RCU protected so that the first pwq can be
         * determined without grabbing wq->mutex.
         */
        struct work_struct      unbound_release_work;
        struct rcu_head         rcu;
} __aligned(1 << WORK_STRUCT_FLAG_BITS);
```

```
struct workqueue_struct {
        struct list_head        pwqs;           /* WR: all pwqs of this wq */
        struct list_head        list;           /* PR: list of all workqueues */

        struct mutex            mutex;          /* protects this wq */
        int                     work_color;     /* WQ: current work color */
        int                     flush_color;    /* WQ: current flush color */
        atomic_t                nr_pwqs_to_flush; /* flush in progress */
        struct wq_flusher       *first_flusher; /* WQ: first flusher */
        struct list_head        flusher_queue;  /* WQ: flush waiters */
        struct list_head        flusher_overflow; /* WQ: flush overflow list */

        struct list_head        maydays;        /* MD: pwqs requesting rescue */
        struct worker           *rescuer;       /* I: rescue worker */

        int                     nr_drainers;    /* WQ: drain in progress */
        int                     saved_max_active; /* WQ: saved pwq max_active */

        struct workqueue_attrs  *unbound_attrs; /* PW: only for unbound wqs */
        struct pool_workqueue   *dfl_pwq;       /* PW: only for unbound wqs */

#ifdef CONFIG_SYSFS
        struct wq_device        *wq_dev;        /* I: for sysfs interface */
#endif
#ifdef CONFIG_LOCKDEP
        struct lockdep_map      lockdep_map;
#endif
        char                    name[WQ_NAME_LEN]; /* I: workqueue name */

        /*
         * Destruction of workqueue_struct is sched-RCU protected to allow
         * walking the workqueues list without grabbing wq_pool_mutex.
         * This is used to dump all workqueues from sysrq.
         */
        struct rcu_head         rcu;

        /* hot fields used during command issue, aligned to cacheline */
        unsigned int            flags ____cacheline_aligned; /* WQ: WQ_* flags */
        struct pool_workqueue __percpu *cpu_pwqs; /* I: per-cpu pwqs */
        struct pool_workqueue __rcu *numa_pwq_tbl[]; /* PWR: unbound pwqs indexed by node */
}
```

```
struct worker {
        /* on idle list while idle, on busy hash table while busy */
        union {
                struct list_head        entry;  /* L: while idle */
                struct hlist_node       hentry; /* L: while busy */
        };

        struct work_struct      *current_work;  /* L: work being processed */
        work_func_t             current_func;   /* L: current_work's fn */
        struct pool_workqueue   *current_pwq; /* L: current_work's pwq */
        bool                    desc_valid;     /* ->desc is valid */
        struct list_head        scheduled;      /* L: scheduled works */

        /* 64 bytes boundary on 64bit, 32 on 32bit */

        struct task_struct      *task;          /* I: worker task */
        struct worker_pool      *pool;          /* I: the associated pool */
                                                /* L: for rescuers */
        struct list_head        node;           /* A: anchored at pool->workers */
                                                /* A: runs through worker->node */

        unsigned long           last_active;    /* L: last active timestamp */
        unsigned int            flags;          /* X: flags */
        int                     id;             /* I: worker id */

        /*
         * Opaque string set with work_set_desc().  Printed out with task
         * dump for debugging - WARN, BUG, panic or sysrq.
         */
        char                    desc[WORKER_DESC_LEN];

        /* used only by rescuers to point to the target workqueue */
        struct workqueue_struct *rescue_wq;     /* I: the workqueue to rescue */
};

```

```
struct work_struct {
        atomic_long_t data;	// 处理函数的参数
        struct list_head entry;	//
        work_func_t func;		// 处理函数
#ifdef CONFIG_LOCKDEP
        struct lockdep_map lockdep_map;
#endif
};

```