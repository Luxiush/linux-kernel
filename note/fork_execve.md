## 进程的创建和执行
- 新的进程通过克隆当前进程而建立.

### fork, vfork, clone
```c
// kernel/fork.c
SYSCAL_DEFINE0(fork)
{
    return _do_fork(SIGCHLD, 0, 0, NULL, NULL, 0);
}

SYSCAL_DEFINE0(vfork)
{
    return _do_fork(CLONE_VFORK | CLONE_VM | SIGCHLD, 0,
            0, NULL, NULL, 0);
}

SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
        int `__user` *, parent_tidptr,
        unsigned long, tls,
        int `__user` *, child_tidptr)
{
    return _do_fork(clone_flags, newsp, 0, parent_tidptr, child_tidptr, tls);
}
```

- 三者都调到了`_do_fork`, 主要通过`clone_flags`来控制需要复制哪些东西.

#### 关于`clone_flags`
```c
// include/uapi/linux/sched.h
#define CSIGNAL            0x000000ff        /* signal mask to be sent at exit */
#define CLONE_VM           0x00000100        /* set if VM shared between processes */
#define CLONE_FS           0x00000200        /* set if fs info shared between processes */
#define CLONE_FILES        0x00000400        /* set if open files shared between processes */
#define CLONE_SIGHAND      0x00000800        /* set if signal handlers and blocked signals shared */
#define CLONE_PTRACE       0x00002000        /* set if we want to let tracing continue on the child too */
#define CLONE_VFORK        0x00004000        /* set if the parent wants the child to wake it up on mm_release */
#define CLONE_PARENT       0x00008000        /* set if we want to have the same parent as the cloner */
#define CLONE_THREAD       0x00010000        /* Same thread group? */
#define CLONE_NEWNS        0x00020000        /* New mount namespace group */
#define CLONE_SYSVSEM      0x00040000        /* share system V SEM_UNDO semantics */
#define CLONE_SETTLS       0x00080000        /* create a new TLS for the child */
#define CLONE_PARENT_SETTID 0x00100000       /* set the TID in the parent */
#define CLONE_CHILD_CLEARTID 0x00200000      /* clear the TID in the child */
#define CLONE_DETACHED     0x00400000        /* Unused, ignored */
#define CLONE_UNTRACED     0x00800000        /* set if the tracing process can't force CLONE_PTRACE on this clone */
#define CLONE_CHILD_SETTID 0x01000000        /* set the TID in the child */
#define CLONE_NEWCGROUP    0x02000000        /* New cgroup namespace */
#define CLONE_NEWUTS       0x04000000        /* New utsname namespace */
#define CLONE_NEWIPC       0x08000000        /* New ipc namespace */
#define CLONE_NEWUSER      0x10000000        /* New user namespace */
#define CLONE_NEWPID       0x20000000        /* New pid namespace */
#define CLONE_NEWNET       0x40000000        /* New network namespace */
#define CLONE_IO           0x80000000        /* Clone io context */
```

### `_do_fork`
- tls: Thread Local Storage

```c
long _do_fork(unsigned long clone_flags,
        unsigned long stack_start,
        unsigned long stack_size,
        int `__user` *parent_tidptr,
        int `__user` *child_tidptr,
        unsigned long tls)
{
    // ...
    p = copy_process(clone_flags, stack_start, stack_size,
            child_tidptr, NULL, trace, tls, NUMA_NO_NODE);
    // ...
}

// 复制一个新的`task_struct`
static __latent_entropy struct task_struct *copy_process(
                    unsigned long clone_flags,
                    unsigned long stack_start,
                    unsigned long stack_size,
                    int `__user` *child_tidptr,
                    struct pid *pid,
                    int trace,
                    unsigned long tls,
                    int node)
{
    struct task_struct* p;
    // ...
    p = dup_task_struct(current, node); // 拷贝taskt_struct结构体
    // ...
    // 清除一些东西
    // ...
    // 根据clone_flags选择性的复制一些东西
    retval = copy_fs(clone_flags, p);
    if (retval)
        goto bad_fork_cleanup_semundo;
    retval = copy_mm(clone_flags, p);
    if (retval)
        goto bad_fork_cleanup_signal;
    retval = copy_thread_tls(clone_flags, stack_start, stack_size, p, tls);
    if (retval)
        goto bad_fork_cleanup_io;
    // ...
}
```

- dup_task_struct
	-- alloc_thread_stack_node 分配一片连续的内存, 做为栈空间. 至于从哪里分配出来的, 这就属于内存管理的部分了. 通过vmalloc, 和task_struct的内存分配一样.
		--- task_struct.stack_vm_area
		--- task_struct.stack

### 从fork/clone来谈谈进程与线程的区别: [Fork vs Clone on 2.6 Kernel Linux](https://unix.stackexchange.com/questions/199686/fork-vs-clone-on-2-6-kernel-linux)
clone可以根据需要复制进程的特定信息. 当新进程与原来的进程`共用一个虚拟地址空间`时创建出来的就称为`子线程`, 当然也可创建出某些介于`子进程`和`子线程`之间的东西, 或是完全不同的东西, 完全取决于`flags`参数.

子线程的栈是从进程的地址空间中map出来的一块内存区域, 不能动态增长 (对应代码中的哪里??, 由clone的调用者申请???)

所谓“新开一条线程”，实质上就是另外申请了一块内存，然后把这块内存当作堆栈，维护另外一条调用链。

linux内核中的多线程栈空间模型: <https://www.zhihu.com/question/323415592>

而fork仅能用于创建`子进程`, 不共用地址空间 ??

#### 内核线程
> linux内核可以看做是一个`服务进程`. 内核需要多个执行流并行, 为了防止可能的阻塞, 多线程化是必要的. 内核线程就是内核的分身, 一个分身处理一件特定事情.
> *linux内核使用内核线程来将内核分成多个功能模块*.
> 内核线程的调度由内核负责, 一个内核线程处于阻塞状态不影响其他内核线程.
> 所有的内核线程共用一个内核栈空间.
([linux内核线程之深入浅出]( https://blog.csdn.net/yiyeguzhou100/article/details/53126626 ))

### 程序的执行 execve
- linux中的程序通过命令解释器(`sehll`)执行. 用户输入命令后, shell会在搜索路径中查找和输入名利匹配的镜像, 如果找到就fork当前进程, 然后用找到的镜像覆盖子进程正在执行的shell二进制镜像.
```c
int main(int argc, char* argv[], char* envp[]){
    // envp 用来传递环境变量, VAR_NAME=something
}
```

```c
// fs/exec.c
SYSCALL_DEFINE3(execve,
		const char `__user` *, filename,
		const char `__user` *const `__user` *, argv,
		const char `__user` *const `__user` *, envp)
{
	return do_execve(getname(filename), argv, envp);
}

static int __do_execve_file(int fd, struct filename *filename,
			struct user_arg_ptr argv,
			struct user_arg_ptr envp,
			int flags, struct file *file)
{
	struct linux_binprm* bprm; // 用`linux_binprm`来保存加载二进制所需的环境

	bprm = kzalloc(sizeof(*bprm), GFP_KERNEL);

	retval = prepare_binprm(bprm); // 读取文件的头128字节

	retval = exec_binprm(bprm);	// 执行二进制文件
	if (retval < 0)
		goto out;
}

static int exec_binprm(struct linux_binprm *bprm)
{
	// ...
	ret = search_binary_handler(bprm);
	// ...
}

int search_binary_handler(struct linux_binprm *bprm)
{
	list_for_each_entry(fmt, &formats, lh) { // formats是一个`linux_binfmt`链表, 记录着系统中所有的可执行文件格式
		retval = fmt->load_binary(bprm);

		if (retval < 0 && !bprm->mm) {
			// we got to flush_old_exec() and failed after it
			read_unlock(&binfmt_lock);
			force_sigsegv(SIGSEGV, current);
			return retval;
		}
		if (retval != -ENOEXEC || !bprm->file) {
			read_unlock(&binfmt_lock);
			return retval;
		}
	}
}

// formats链表通过`__register_binfmt`和`unregister_binfmt`来维护
static LIST_HEAD(formats);
static DEFINE_RWLOCK(binfmt_lock);

void __register_binfmt(struct linux_binfmt* fmt, int insert)
{
	BUG_ON(!fmt);
	if (WARN_ON(!fmt->load_binary))
		return;
	write_lock(&binfmt_lock);
	insert ? list_add(&fmt->lh, &formats) :
		 list_add_tail(&fmt->lh, &formats);
	write_unlock(&binfmt_lock);
}
EXPORT_SYMBOL(`__register_binfmt`);

void unregister_binfmt(struct linux_binfmt* fmt)
{
	write_lock(&binfmt_lock);
	list_del(&fmt->lh);
	write_unlock(&binfmt_lock);
}
EXPORT_SYMBOL(unregister_binfmt);

//用`linux_binfmt`数据结构来描述所支持的各种可执行文件.
// include/linux/binfmts.h
struct linux_binfmt {
	struct list_head lh;
	struct module* module;
	int (*load_binary)(struct linux_binprm*);
	int (*load_shlib)(struct file*);
	int (*core_dump)(struct coredump_params* cprm);
	unsigned long min_coredump;	// minimal dump size
} `__randomize_layout`;
```
