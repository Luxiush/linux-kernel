## 汇编

```
section.data

section.bss

section.text
    global _start

_start:

```

- 汇编程序代码分为三节: `data`/`bss`和`text`,
    o `data`: 用于存放常量, 运行时数据不会改变.
    o `bss`: 用于定义变量.
    o `text`: 存放程序代码, 必须以`global _start`开始, 用于告诉系统程序的入口.

- 程序运行时的内存模型:
    o 在上面的基础上多了一个`栈`区

- Reference: [Assembly - Basic Syntax]( https://www.tutorialspoint.com/assembly_programming/assembly_basic_syntax.htm )


### 基本汇编语句:
```
movl $123 %ebx
movl (%eax) %ecx
```

- 指令的最后一个字符表示操作数的长度: b(byte), w(word), l(long)
- `源`操作数在前, `目标`作数在后: op-code src dest
- 寄存器名称以`%`开头: `%eax`, `%ebx`
- 立即数以`$`开头: `$123`, `$0x80`
- 访问寄存器所指向的内存: `disp(base, index, scale)`(base + index * scale),
  如: `mov -4(%esi) (%eax, %ebx, 5)`, move 4bytes at address `esi+(-4)` into `eax+ebx*5`


### 8个通用寄存器
```
        32                16       8        0
---------------------------------------------
EAX     |                 |       AX        |
        |                 |  AH    |   AL   |
---------------------------------------------
EBX     |                 |       BX        |
        |                 |  BH    |   BL   |
---------------------------------------------
ECX     |                 |       CX        |
        |                 |  CH    |   CL   |
---------------------------------------------
EDX     |                 |       DX        |
        |                 |  DH    |   DL   |
---------------------------------------------
ESI     |                 |        SI       | Source Index
---------------------------------------------
EDI     |                 |        DI       | Destination Index
---------------------------------------------
ESP                                         | Stack Pointer
---------------------------------------------
EBP                                         | Base Pointer
---------------------------------------------
```

- EAX的低16位称为AX; AX的高8位称为AL, 低8位称为AL; 这三个在物理上是同一个寄存器,
更新AX的值, EAX/AH/AL的值都会跟着改变.

- ESP通常用来放栈顶指针.
- EBP通常指向栈的底部.

- Reference: [x86 Assembly Guide]( http://flint.cs.yale.edu/cs421/papers/x86-asm/asm.html )



## GCC内联汇编
```
asm (assembler template
   : output operands                  /* optional */
   : input operands                   /* optional */
   : list of clobbered registers      /* optional */
   );
```

### 例:
```
int a=10, b;
asm ("movl %1, %%eax;\n\t"
     "movl %%eax, %0;"
     :"=r"(b)        /* output */
     :"r"(a)         /* input */
     :"%eax"         /* clobbered register */
     );
```

#### output/input:
- `%0`, `%1` 分别对应`第0个操作数`和`第1个操作数`, 以此类推, `%n`表示第n个操作数(0 <= n <= 9).
- 为了区分寄存器和操作数占位符, 寄存器名称前多加了一个`%`.

- `"=r"(b)`: b为第0个操作数; `=r`为操作数的约束符, 其中:
    `r`表示用`任意一个`通用寄存器来存放操作数,
    `=`表示该变量是`只写`的通常用来修饰汇编语句的输出操作数.

#### clobberd list:
- 告诉GCC哪些寄存器的值会被修改.

#### 常见的操作数约束符:
| .  | .  |
|:---|:---|
| r | 任意一个通用寄存器 |
| a | %eax, %ax, %ah, %al |
| b | %ebx, %bx, %bh, %bl |
| c | %ecx, %cx, %ch, %cl |
| d | %edx, %dx, %dh, %dl |
| S | %esi, %si |
| D | %edi, %di |
| . | . |
| 0 | 和第0个操作数的约束符保持一致 |
| . | . |
| m | 直接操作内存, 不经过寄存器, 其他的寄存器约束符会先加载到寄存器操作完之后再写回内存 |

- Reference: [GCC-Inline-Assembly-HOWTO]( http://www.ibiblio.org/gferg/ldp/GCC-Inline-Assembly-HOWTO.html?spm=a2c4e.10696291.0.0.679619a420poev )


### linux内核代码中的汇编
- 通过内联汇编调系统调用
```
// ./build/source/tools/testing/selftests/rcutorture/bin/nolibc.h
#define _syscall0(type, name)          \
    type name(void)                        \
    {                                      \
        long __res;                        \
        __asm__ volatile ( "int $0x80"    \
          : "=a" (__res)                  \
          : "0" (__NR_##name));            \
        __syscall_return(type, __res);    \
    }

static __attribute__((unused))
int sys_chdir(const char *path)
{
    return my_syscall1(__NR_chdir, path);
}
```
