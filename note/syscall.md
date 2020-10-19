## 系统调用

### 内核与shell
![](./img/linux_sys_arch.jpg)
- 内核对外提供了一套系统调用接口(System Call Interface)
> SCI 实际上是一个非常有用的函数调用多路复用和多路分解服务。<https://www.ibm.com/developerworks/cn/linux/l-linux-kernel/>

- 参考: Documentation/process/adding-syscalls.rst
- 用户空间和内核提供的服务之间的一个接口. 由于服务在内核中提供, 用户程序无法直接调用, 因此需要一个进程来`跨越用户空间与内核空间的界限`.

- 用户程序通过中断向内核发请求.

### 用户应用程序调用系统调用的方式:

1. 通过glibc库封装的函数(`/usr/include/unistd.h`)
2. `syscall()`和相应的系统调用号(`/usr/include/sys/syscall.h`)
3. `int`汇编指令

### `int`指令
- `interrupt`, 软件中断
- `int n`:
    共2字节，第一字节是int的机器码，第二字节是8位立即数，表示中断号。
    CPU在执行到INT指令时，通过中断描述符表找到中断号对应的中断服务子程序的地址（本质上就是一个异常处理程序的软件调用）.

- linux下, `int 0x8`的处理函数为`系统调用`的总入口.

### 参数传递 & 返回值
`system_call` 所有系统调用的总入口

系统调用中断`int 0x80`, 中断处理函数`system_call`.
通过`eax`寄存器传系统调用号, 通过寄存器传递参数
在`sys_call_table`查找相应的处理函数
执行对应的处理函数
`ret_from_sys_call`

- 系统调用的声明: `include/linux/syscalls.h`
- 系统调用号的定义: `include/uapi/asm-generic/unistd.h` (通用的)

### 中断向量表的初始化
```c
start_kernel /* init/main.c */
    -> trap_init   /* arch/x86/kernel/traps.c */
        -> idt_setup_traps /* arch/x86/kernel/idt.c */
            -> idt_setup_from_table
```

```c
/* arch/x86/include/asm/irq_vectors.h */
#define IA32_SYSCALL_VECTOR     0x80

/* arch/x86/kernel/idt.c */
static const __initconst struct idt_data def_idts[] = {
    // ...
#if defined(CONFIG_IA32_EMULATION)
    SYSG(IA32_SYSCALL_VECTOR,       entry_INT80_compat),
#elif defined(CONFIG_X86_32)
    SYSG(IA32_SYSCALL_VECTOR,       entry_INT80_32),
#endif
};

void __init idt_setup_traps(void)
{
    idt_setup_from_table(idt_table, def_idts, ARRAY_SIZE(def_idts), true); // 初始化中断向量表
}
```

### 系统调用的调用过程
```c
entry_INT80_32
    -> do_int80_syscall_32
        -> do_syscall_32_irqs_on
            -> regs->ax = ia32_sys_call_table[nr](regs)
```

#### 相关代码
```c
/* arch/x86/entry/entry_32.S */
/* `系统调用`中断的处理函数 */
ENTRY(entry_INT80_32)
	ASM_CLAC
	pushl	%eax			/* pt_regs->orig_ax */

	SAVE_ALL pt_regs_ax=$-ENOSYS switch_stacks=1	/* save rest */

	/*
	 * User mode is traced as though IRQs are on, and the interrupt gate
	 * turned them off.
	 */
	TRACE_IRQS_OFF

	movl	%esp, %eax
	call	do_int80_syscall_32  /*  */
.Lsyscall_32_done:

	STACKLEAK_ERASE

restore_all:
	TRACE_IRQS_IRET

    /* ...... */

ENDPROC(entry_INT80_32)
```

```c
/* arch/x86/entry/common.c */
/* Handles int $0x80 */
__visible void do_int80_syscall_32(struct pt_regs *regs)
{
	enter_from_user_mode();
	local_irq_enable();
	do_syscall_32_irqs_on(regs);
}

/*
 * Does a 32-bit syscall.  Called with IRQs on in CONTEXT_KERNEL.  Does
 * all entry and exit work and returns with IRQs off.  This function is
 * extremely hot in workloads that use it, and it's usually called from
 * do_fast_syscall_32, so forcibly inline it to improve performance.
 */
static __always_inline void do_syscall_32_irqs_on(struct pt_regs* regs)
{
	struct thread_info* ti = current_thread_info();
	unsigned int nr = (unsigned int)regs->orig_ax;

#ifdef CONFIG_IA32_EMULATION
	ti->status |= TS_COMPAT;
#endif

	if (READ_ONCE(ti->flags) & \_TIF_WORK_SYSCALL_ENTRY) {
		// Subtlety here: if ptrace pokes something larger than
		// 2^32-1 into orig_ax, this truncates it.  This may or
		// may not be necessary, but it matches the old asm
		// behavior.
		nr = syscall_trace_enter(regs);
	}

	if (likely(nr < IA32_NR_syscalls)) {
		nr = array_index_nospec(nr, IA32_NR_syscalls);

#ifdef CONFIG_IA32_EMULATION
		regs->ax = ia32_sys_call_table[nr](regs);
#else

		 // It's possible that a 32-bit syscall implementation
		 // takes a 64-bit parameter but nonetheless assumes that
		 // the high bits are zero.  Make sure we zero-extend all
		 // of the args.
		regs->ax = ia32_sys_call_table[nr](
			(unsigned int)regs->bx, (unsigned int)regs->cx,
			(unsigned int)regs->dx, (unsigned int)regs->si,
			(unsigned int)regs->di, (unsigned int)regs->bp);
#endif // CONFIG_IA32_EMULATION
	}

	syscall_return_slowpath(regs);
}
```

### 系统调用表的定义和初始化
#### `ia32_sys_call_table`的定义
- `arch/x86/entry/syscall_32.c`
```c
// SPDX-License-Identifier: GPL-2.0
/* System call table for i386. */

#include <linux/linkage.h>
#include <linux/sys.h>
#include <linux/cache.h>
#include <asm/asm-offsets.h>
#include <asm/syscall.h>

#ifdef CONFIG_IA32_EMULATION
// On X86_64, we use struct pt_regs * to pass parameters to syscalls
#define __SYSCALL_I386(nr, sym, qual) extern asmlinkage long sym(const struct pt_regs *); // atom_bug__

// this is a lie, but it does not hurt as sys_ni_syscall just returns -EINVAL
extern asmlinkage long sys_ni_syscall(const struct pt_regs \*);

#else // CONFIG_IA32_EMULATION
#define __SYSCALL_I386(nr, sym, qual) extern asmlinkage long sym(unsigned long, unsigned long, unsigned long, unsigned long, unsigned long, unsigned long); // atom_bug__
extern asmlinkage long sys_ni_syscall(unsigned long, unsigned long, unsigned long, unsigned long, unsigned long, unsigned long);
#endif // CONFIG_IA32_EMULATION

#include <asm/syscalls_32.h> // 编译的时候生成
#undef __SYSCALL_I386

#define __SYSCALL_I386(nr, sym, qual) [nr] = sym,

__visible const sys_call_ptr_t ia32_sys_call_table[__NR_syscall_compat_max+1] = {
	// Smells like a compiler bug -- it doesn't work
	// when the & below is removed.
	[0 ... __NR_syscall_compat_max] = &sys_ni_syscall,
#include <asm/syscalls_32.h> // 编译的时候生成
};
```

- `sys_call_ptr_t`
```c
// arch/x86/include/asm/syscall.h
#ifdef CONFIG_X86_64
typedef asmlinkage long (\*sys_call_ptr_t)(const struct pt_regs \*);
#else
typedef asmlinkage long (\*sys_call_ptr_t)(unsigned long, unsigned long,
					  unsigned long, unsigned long,
					  unsigned long, unsigned long);
#endif // CONFIG_X86_64
```

#### 头文件的生成
- 代码中系统调用号的最终定义: `./arch/x86/entry/syscalls/syscall_32.tbl`
- 生成脚本: `./arch/x86/entry/syscalls/*.sh`
- 输出
```c
// xxxxxx/arch/x86/include/generated/asm/syscalls_32.h
#ifdef CONFIG_X86_32
__SYSCALL_I386(0, sys_restart_syscall, ) // atom_bug__
#else
__SYSCALL_I386(0, __ia32_sys_restart_syscall, ) // atom_bug__
#endif
#ifdef CONFIG_X86_32
__SYSCALL_I386(1, sys_exit, ) // atom_bug__
#else
__SYSCALL_I386(1, __ia32_sys_exit, ) // atom_bug__
#endif
// 略......
```

```c
// xxxxxx/arch/x86/include/generated/asm/syscalls_32.h
#ifndef _ASM_X86_UNISTD_32_H
#define _ASM_X86_UNISTD_32_H 1

#define __NR_restart_syscall 0
#define __NR_exit 1
#define __NR_fork 2
#define __NR_read 3
#define __NR_write 4
#define __NR_open 5
#define __NR_close 6
#define __NR_waitpid 7 // atom_bug__
// 略......

#endif /* _ASM_X86_UNISTD_32_H */
```
