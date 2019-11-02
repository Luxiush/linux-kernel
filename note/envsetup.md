# Environment Setup
- linux内核开发环境搭建

## part 1. 内核编译
###  编译配置
```
$ make menuconfig
```

- kgdb的一些编译配置 <https://www.kernel.org/doc/html/latest/dev-tools/kgdb.html>

- 用于调试的一些编译配置
```
CONFIG_PRINTK_TIME - add time stamps to dmesg
CONFIG_DEBUG_KERNEL - turn on kernel debugging
CONFIG_DETECT_HUNG_TASK - good for figuring out what's causing a kernel freeze
CONFIG_DEBUG_INFO - ensures you can decode kernel oops symbols
CONFIG_EARLY_PRINTK
CONFIG_LOG_BUF_SHIFT=21 - sets the kernel buffer log size to the biggest buffer
```

### 编译
```
make O=./build # 指定输出路径
make M
```

---
## part 2. busybox 根文件系统
- <https://consen.github.io/2018/01/17/debug-linux-kernel-with-qemu-and-gdb/>
- <https://blog.0x972.info/?d=2014/11/27/18/45/48-linux-kernel-system-debugging-part-1-system-setup>

- [关于initramfs]( https://wiki.gentoo.org/wiki/Custom_Initramfs )


### busybox编译安装
```
$ cd busybox.xxxxx
$ make menuconfig # Settings ---> Build static binary
$ make -j 8
$ make install # 安装在_install目录
```

### 构建initramfs
```
$ mkdir initramfs
$ cd initramfs

# create standard filesystem directories
$ mkdir -pv bin lib dev etc mnt/root proc root sbin sys

# create standard file devices
$ sudo cp -va /dev/{null,console,tty} dev/

$ sudo mknod dev/sda b 8 0

# import busybox filesystem
$ cp ../busybox-$BVERSION/_install/* . -rv

# prepare an "init" scripts
$ cat > init << EOF
#!/bin/sh

/bin/mount -t proc none /proc
/bin/mount -t sysfs sysfs /sys
/bin/mount -t ext2 /dev/sda /mnt/root

exec /bin/sh
EOF

$ chmod 755 init
```

### 打包initramfs
```
$ cd initramfs
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
$ make install # install to /usr/local
```

#### 一些依赖
1. pixman
```
$ apt-cache search pixman
$ sudo apt-get install libpixman-1-dev
```

### 创建一个虚拟硬盘
```
# qemu-img create [-q] [-f fmt] [-o options] filename [size]
$ $qemu_install_path/qemu-img create disk.img 512M

# 格式化硬盘为ext2格式
$ mkfs.ext2 -F disk.img
```

### 启动新编译的内核
```
$ $qemu_install_path/qemu-system-x86_64 -nographic -hda disk.img -kernel $linux_build_path/arch/x86_64/boot/bzImage -initrd initramfs.cpio -append "console=ttyS0" -s
```

#### 参数说明
- `-s`  Shorthand for -gdb tcp::1234, i.e. open a gdbserver on TCP port 1234.

- `-S`  Do not start CPU at startup (you must type 'c' in the monitor).

- `-hda [file]`,`-hdb [file]`,`-hdc [file]`,`hdd [file]`    Use `[file]` as hard disk 0, 1, 2 or 3 image.

- `-drive file=/home/lxs-hadoop1/qemu.disk,index=0,media=disk` 相当于`-hda`, 定义一块硬盘.

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
```

### Error
#### 加载vmlinux的时候, NameError: name 'MS_RDONLY' is not defined
- scripts/gdb/vmlinux-gdb.py加载出问题
- 好吧, 现在的方法是在整个目录下grep一下, 找到该变量的定义是在`usr/include/linux/mount.h`, 然后手动把那些变量的的值挪过来即可.
