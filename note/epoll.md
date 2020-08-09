## [epoll]( https://man7.org/linux/man-pages/man7/epoll.7.html )

IO多路复用, 同时监控多个文件描述符.

- ep_insert
```
ep_insert
    ->ep_item_poll
        -> file->f_op->poll (file结构体上的poll接口)
            ->poll_wait
                -> ep_ptable_queue_proc
                    - 把对应的epitem添加到需要监控的文件的等待队列中.
                    - 当文件有事件需要报告时会唤醒在等待队列中等待的进程,
                    在唤醒的时候回调`ep_poll_callback`. (参见: __wake_up_common)
```

- ep_poll_callback
```
ep_poll_callback
    - poll的回调函数, 当对应的file descripter有事件要报告时回调, 把对应的epitem添加到ready-list中.
    - 同时唤醒在之前在poll_wait时因为没有事件而进入等待状态的进程. (参见: __wake_up_common)
```

- epoll_wait
```
epoll_wait
    -> do_epoll_wait
        -> ep_poll
            -> 拿ready-list
            -> 如果ready-list为空, 还没事件, 进入等待状态
            -> 否则 ep_send_events
                -> ep_send_events_proc
                    - 取出ready-list中的item, 逐个poll把返回的事件放到用户空间的缓存中.
                    - 如果是`LevelTrigger`模式, item还要重新放回ready-list中,
                    这样下次调`epoll_wait`的时候就还会去poll该item. 因为:
                        1. `epoll_wait`的调用者不一定会一次性把数据读完;
                        2. 如果item所管理的文件是阻塞式的, 那么调用者尝试一次性读完有就会被阻塞.
```

- ep_send_events_proc
```c
// fs/eventpoll.c
static __poll_t ep_send_events_proc(struct eventpoll *ep, struct list_head *head,
			       void *priv)
{
	struct ep_send_events_data* esed = priv;
	__poll_t revents; // x__
	struct epitem * epi, * tmp;
	struct epoll_event __user *uevent = esed->events; // x__
	struct wakeup_source * ws;
	poll_table pt;

	init_poll_funcptr(&pt, NULL);
	esed->res = 0;

	 // We can loop without lock because we are passed a task private list.
	 // Items cannot vanish during the loop because ep_scan_ready_list() is
	 // holding "mtx" during this call.
	lockdep_assert_held(&ep->mtx);

	list_for_each_entry_safe(epi, tmp, head, rdllink) {
		if (esed->res >= esed->maxevents)
			break;

		 // Activate ep->ws before deactivating epi->ws to prevent
		 // triggering auto-suspend here (in case we reactive epi->ws
		 // below).
		 //
		 // This could be rearranged to delay the deactivation of epi->ws
		 // instead, but then epi->ws would temporarily be out of sync
		 // with ep_is_linked().
		ws = ep_wakeup_source(epi);
		if (ws) {
			if (ws->active)
				__pm_stay_awake(ep->ws);
			__pm_relax(ws);
		}

		list_del_init(&epi->rdllink);

		 // If the event mask intersect the caller-requested one,
		 // deliver the event to userspace. Again, ep_scan_ready_list()
		 // is holding ep->mtx, so no operations coming from userspace
		 // can change the item.
		revents = ep_item_poll(epi, &pt, 1);
		if (!revents)
			continue;

		if (__put_user(revents, &uevent->events) ||
		    __put_user(epi->event.data, &uevent->data)) {
			list_add(&epi->rdllink, head);
			ep_pm_stay_awake(epi);
			if (!esed->res)
				esed->res = -EFAULT;
			return 0;
		}
		esed->res++;
		uevent++;
		if (epi->event.events & EPOLLONESHOT)
			epi->event.events &= EP_PRIVATE_BITS;
		else if (!(epi->event.events & EPOLLET)) {
			 // If this file has been added with Level
			 // Trigger mode, we need to insert back inside
			 // the ready list, so that the next call to
			 // epoll_wait() will check again the events
			 // availability. At this point, no one can insert
			 // into ep->rdllist besides us. The epoll_ctl()
			 // callers are locked out by
			 // ep_scan_ready_list() holding "mtx" and the
			 // poll callback will queue them in ep->ovflist.
			list_add_tail(&epi->rdllink, &ep->rdllist);
			ep_pm_stay_awake(epi);
		}
	}

	return 0;
}

```

- f_op->poll
```c
static const struct file_operations eventpoll_fops = {
#ifdef CONFIG_PROC_FS
	.show_fdinfo	= ep_show_fdinfo,
#endif
	.release	= ep_eventpoll_release,
	.poll		= ep_eventpoll_poll,
	.llseek		= noop_llseek,
};
```


### 关于poll
- file->f_op->poll()
```c
struct file_operations {
    // ...
    __poll_t (*poll) (struct file*, struct poll_table_struct*);
    // ...
};
```
file有一个poll的等待队列, 当文件有事件要报告时, 会唤醒等待队列中的item.
调用poll的时候会往等待队列中注册一个Item.

- 以`port_fops`的poll为例:
```c
// drivers/char/virtio_console.c
static const struct file_operations port_fops {
    // ...
    .poll = port_fops_poll
    // ...
};

static __poll_t port_fops_poll(struct file* filp, poll_table* wait)
{
	struct port* port;
	__poll_t ret; // x__

	port = filp->private_data;
	poll_wait(filp, &port->waitqueue, wait);

	if (!port->guest_connected) {
		// Port got unplugged
		return EPOLLHUP;
	}
	ret = 0;
	if (!will_read_block(port))
		ret |= EPOLLIN | EPOLLRDNORM;
	if (!will_write_block(port))
		ret |= EPOLLOUT;
	if (!port->host_connected)
		ret |= EPOLLHUP;

	return ret;
}

// include/linux/poll.h
static inline void poll_wait(struct file * filp, wait_queue_head_t * wait_address, poll_table* p)
{
	if (p && p->_qproc && wait_address)
		p->_qproc(filp, wait_address, p);
}
```

```c
// kernel/sched/wait.c
/*
#define wake_up_interruptible(x)»···__wake_up(x, TASK_INTERRUPTIBLE, 1, NULL)

__wake_up
    -> __wake_up_common_lock
        -> __wake_up_common
*/
static int __wake_up_common(struct wait_queue_head* wq_head, unsigned int mode,
			int nr_exclusive, int wake_flags, void* key,
			wait_queue_entry_t* bookmark)
{
	wait_queue_entry_t * curr, * next;
	int cnt = 0;

	lockdep_assert_held(&wq_head->lock);

	if (bookmark && (bookmark->flags & WQ_FLAG_BOOKMARK)) {
		curr = list_next_entry(bookmark, entry);

		list_del(&bookmark->entry);
		bookmark->flags = 0;
	} else
		curr = list_first_entry(&wq_head->head, wait_queue_entry_t, entry);

	if (&curr->entry == &wq_head->head)
		return nr_exclusive;

	list_for_each_entry_safe_from(curr, next, &wq_head->head, entry) {
		unsigned flags = curr->flags;
		int ret;

		if (flags & WQ_FLAG_BOOKMARK)
			continue;

		ret = curr->func(curr, mode, wake_flags, key); //
		if (ret < 0)
			break;
		if (ret && (flags & WQ_FLAG_EXCLUSIVE) && !--nr_exclusive)
			break;

		if (bookmark && (++cnt > WAITQUEUE_WALK_BREAK_CNT) &&
				(&next->entry != &wq_head->head)) {
			bookmark->flags = WQ_FLAG_BOOKMARK;
			list_add_tail(&bookmark->entry, &next->entry);
			break;
		}
	}

	return nr_exclusive;
}

```
