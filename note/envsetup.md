# Environment Setup
- linux内核开发环境搭建

## part 1. 内核编译
### 一些依赖
> Please read the `Documentation/process/changes.rst` file, as it contains the
requirements for building and running the kernel, and information about
the problems which may result by upgrading your kernel.

###  编译配置
```
$ make O=./build menuconfig
```

- kgdb的一些编译配置 <https://www.kernel.org/doc/html/latest/dev-tools/kgdb.html>

- 用于调试的一些编译配置
```
RANDOMIZE_BASE / RANDOMIZE_MEMORY - Randomize the address of the kernel image, 禁掉,不然断不下来

CC_OPTIMIZE_FOR_PERFORMANCE - Optimize for performance, 去优化

CONFIG_PRINTK_TIME - add time stamps to dmesg
CONFIG_DEBUG_KERNEL - turn on kernel debugging
CONFIG_DETECT_HUNG_TASK - good for figuring out what's causing a kernel freeze
CONFIG_DEBUG_INFO - ensures you can decode kernel oops symbols
CONFIG_EARLY_PRINTK
CONFIG_LOG_BUF_SHIFT=21 - sets the kernel buffer log size to the biggest buffer

CONFIG_READABLE_ASM
```

### 编译
```
make O=./build # 指定输出路径

```

---
## part 2. busybox 根文件系统
- <https://consen.github.io/2018/01/17/debug-linux-kernel-with-qemu-and-gdb/>
- <https://blog.0x972.info/?d=2014/11/27/18/45/48-linux-kernel-system-debugging-part-1-system-setup>

- [关于initramfs]( https://wiki.gentoo.org/wiki/Custom_Initramfs )
- 关于`initrd`: `Documentation/admin-guide/init*`


### busybox编译安装
```
$ cd busybox.xxxxx
$ make menuconfig # Settings ---> Build static binary
$ make -j 8
$ make install # 安装在_install目录
```

### 构建initramfs
- 只构建initranfs, 没有硬盘
- 参见busybox的`example/bootfloppy/`目录
```
$ mkdir initramfs
$ cd initramfs

# create standard file devices
$ sudo cp -va /dev/{null,console,tty} dev/

# import busybox filesystem
$ cp -r ../busybox-$BVERSION/_install/* .

# 启动内核时没有指定init参数时会默认尝试加载'/init'文件:
$ cat > /init <<INITEND
#!/bin/sh
/bin/mount -t proc none /proc
exec /sbin/init
INITEND

# 打包initramfs
$ find . -print0 | cpio --null -ov --format=newc > ../initramfs.cpio
```

---
## part 3. qemu虚拟机

### 源码编译安装
```
$ git clone git://git.qemu.org/qemu.git # 下载源码
$ cd qemu-x.y.z
$ mkdir build
$ cd build
$ ../configure
$ make
$ make install # install to /usr/local/share
```

#### 一些依赖
1. pixman
```
$ apt-cache search pixman
$ sudo apt-get install libpixman-1-dev
```

2. glib-2.40 gthread-2.0 is required to compile QEMU
```
$ apt-cache search glib2
$ sudo apt install libglib2.0-dev
```

### 创建一个虚拟硬盘
```
# qemu-img create [-q] [-f fmt] [-o options] filename [size]
$ $qemu_install_path/qemu-img create disk.img 512M

# 格式化硬盘为ext2格式(还是ext4格式?)
$ mkfs.ext2 -F disk.img
$

# 查看虚拟硬盘信息
$ qemu-img info disk.img
```

### 启动新编译的内核
```
# 无硬盘启动, 只加载initrd
$ $qemu_install_path/qemu-system-x86_64 -nographic -kernel $linux_build_path/arch/x86_64/boot/bzImage -initrd initramfs.cpio -append "console=ttyS0" -s

$ $qemu_install_path/qemu-system-x86_64 -nographic -hda disk.img -kernel $linux_build_path/arch/x86_64/boot/bzImage -initrd initramfs.cpio -append "console=ttyS0" -s
```

#### 参数说明
- `-s`  Shorthand for -gdb tcp::1234, i.e. open a gdbserver on TCP port 1234.

- `-S`  Do not start CPU at startup (you must type 'c' in the monitor).

- `-hda [file]`,`-hdb [file]`,`-hdc [file]`,`hdd [file]`    Use `[file]` as hard disk 0, 1, 2 or 3 image.

- `-drive file=/home/lxs-hadoop1/qemu.disk,index=0,media=disk,format=raw` 相当于`-hda`, 定义一块硬盘.

- `-kernel` 指定内核镜像.

- `initrd` 指定制作的initramfs

- `-append [cmdline]` 指定内核的启动参数.

- `-nographic`  with ths option, you can totally disable graphical output so that QEMU is a simple command line application. Use `Ctrl-a h` for more information

- use command `man qemu` for more details.

- 常用命令 (带-nographic参数时)
| . | . |
|:---|:---|
| C-a ? | help |
| C-a x | exit emulator |
| C-a c | switch between console and monitor |

---
## part 4. 启动gdb, 附加到进程调试
```
$ cd path/to/linux-build
$ gdb vmlinux
(gdb) target remote localhost:1234
(gdb) b start_kernel
```

---
## part 5. Error

### 1. gdb加载vmlinux的时候, NameError: name 'MS_RDONLY' is not defined
- scripts/gdb/vmlinux-gdb.py加载出问题
- 好吧, 现在的方法是在整个目录下grep一下, 找到该变量的定义是在`usr/include/linux/mount.h` `include/uapi/linux/mount.h`, 然后手动把那些变量的的值挪过来即可.


### 2. 设置了断点, 但是断不下来
- 禁掉编译选项:`RANDOMIZE_BASE`和`RANDOMIZE_MEMORY`


### 3. non-retpoline compiler
> You are building kernel with non-retpoline compiler.
> Please update your compiler.

#### 分析:
报错位置`arch/x86/Makefile`:
```
296 ifdef CONFIG_RETPOLINE
297 ifeq ($(RETPOLINE_CFLAGS),)
298 	@echo "You are building kernel with non-retpoline compiler." >&2
299 	@echo "Please update your compiler." >&2
300 	@false
301 endif
302 endif
```

再看看`RETPOLINE_CFLAGS`的定义在`./Makefile`:
```
512 RETPOLINE_CFLAGS := $(call cc-option,$(RETPOLINE_CFLAGS_GCC),$(call cc-option,$(RETPOLINE_CFLAGS_CLANG)))
513 RETPOLINE_VDSO_CFLAGS := $(call cc-option,$(RETPOLINE_VDSO_CFLAGS_GCC),$(call cc-option,$(RETPOLINE_VDSO_CFLAGS_CLANG)))
514 export RETPOLINE_CFLAGS
515 export RETPOLINE_VDSO_CFLAGS
```

根据`scripts/Kconfig.include`中`cc-option`函数的定义将`RETPOLINE_CFLAGS`展开后对应的命令如下:
```
$ gcc -Werror -mindirect-branch=thunk-extern -mindirect-branch-register -E -x c /dev/null -o /dev/null

$ gcc -Werror -mretpoline-external-thunk -E -x c /dev/null -o /dev/null
```

在命令行中执行以上命:
```
$ gcc -Werror -mindirect-branch=thunk-extern -mindirect-branch-register -E -x c /dev/null -o /dev/null
gcc: error: unrecognized command line option ‘-mindirect-branch=thunk-extern’
gcc: error: unrecognized command line option ‘-mindirect-branch-register’

$ gcc -Werror -mretpoline-external-thunk -E -x c /dev/null -o /dev/null
gcc: error: unrecognized command line option ‘-mretpoline-external-thunk’
```
可以看到, gcc并不支持这些选项.

这么看来, 关掉配置项`CONFIG_RETPOLINE`("Processor type and features" -> "Avoid speculative indirect branches in kernel")就可以了 <https://lkml.org/lkml/2018/12/8/92>
