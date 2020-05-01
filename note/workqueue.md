## Workqueue
- Documentation/core-api/workqueue.rst

- 用于异步执行

>A `work item` is a simple struct that holds a pointer to the function
that is to be executed asynchronously.  Whenever a driver or subsystem
wants a function to be executed asynchronously it has to set up a work
item pointing to that function and queue that work item on a
`workqueue`.

>Special purpose threads, called `worker_thread`, execute the functions
off of the queue, one after the other.  If no work is queued, the
worker threads become idle.  These worker threads are managed in so
called `worker-pools`.


```c
// 最终对外暴露的结构
// 用于将work_struct放到合适的worker_pool
struct workqueue_struct{
    strut list_head pwqs; // pool_workqueue.pwqs_node串成的链表
    struct pool_workqueue __percpu* cpu_pwqs; // perpu变量, 每个cpu都会有一个副本, 这样就可以实现和cpu的绑定__ 
};

struct pool_workqueue {
    struct worker_pool* pool;
    struct workqueue_struct* wq;

    struct list_head pwqs_node; // 用于在workqueue_struct.pwqs里面排队
};

struct worker_pool {
    struct list_head worklist;  // 需要处理的work (work_struct.entry串成的一个链表)
    struct list_head workers;   // 当前的worker (worker.entry串成的一个链表)
};

// 每个worker都会隶属于一个worker_pool,
// worker的具体运行逻辑在`int worker_thread(void* __worker__)`函数定义
struct worker {
    // on idle list while idle, on busy hash table while busy
    union {
        struct list_head entry; // 用于在worker_pool里面里面排队
        struct hlist_node hentry;
    };

    struct stask_struct* task;
    struct worker_pool pool; // 自己所属的worker_pool;
};

// workitem的定义
typedef void (* work_func_t)(struct work_struct work);
struct work_struct {
    atomic_long_t data;
    struct list_head entry; // 用于在worklist里面排队
    work_func_t func;
};

// kernel/workqueue.c
// 定义每个worker的运行逻辑
static int worker_thread(void* __worker__)
{
    struct worker* worker = __worker__;
    struct worker_pool* pool = worker->pool;

woke_up:

    if (!need_more_worker(pool)) // !list_empty(&pool->worklist) && !atomic_read(&pool->nr_running);
        goto sleep;

    do {
        // 每次从自己所属的worker_pool里面拿一个work
        struct work_struct work* =
            list_first_entry(&pool->worklist,
                struct work_struct, entry);

        process_one_work(worker, work);

    } while (keep_working(pool)); // !list_empty(&pool->worklist) && atomic_read(&pool->nr_running) <= 1

sleep:

    schedule();
    goto woke_up;
}


static void __queue_work(int cpu, struct workqueue_struct* wq,
            struct work_struct* work)
{
    struct pool_workqueue pwq;
    struct worker_pool* last_pool;
    struct list_head* worklist;

    if (req_cpu == WORK_CPU_UNBOUND) // 如果不需要绑定cpu, 则默认优先用当前所在的cpu
        cpu = wq_select_unbound_cpu(raw_smp_processor_id());

    // 拿指定cpu上的pool_workqueue

    // 拿worker_pool

    // 拿worklist

    insert_work(pwq, work, worklist, work_flags);
}
```


### worker_pool中worker的管理:
- 如果worker_pool中还有work, 则至少保持一个worker处于running状态.
- 如果worker运行过程中进入了挂起(suspend)状态, 则唤醒一个处于idle状态的worker来处理work.
- 如果有work需要执行, 并且处于running状态的worker多余一个, 则让多余的worker进入idle状态.
<http://kernel.meizu.com/linux-workqueue.html>
