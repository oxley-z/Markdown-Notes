 
# C内联汇编
---

C语言内联汇编基本语法如下：

```c
asm asm-qualifiers ( AssemblerTemplate 
                 : OutputOperands 
                 [ : InputOperands
                 [ : Clobbers ] ])

asm asm-qualifiers ( AssemblerTemplate 
                      : OutputOperands
                      : InputOperands
                      : Clobbers
                      : GotoLabels)
asm 修饰词(
			指令部分
			: 输出部分
			: 输入部分
			: 损坏部分)
```
其中：
* **asm**：声明C程序中插入汇编代码，也可写作```__asm__```。

* **asm-qualifiers**：限定符，其可选值有：
```volatile、inline、goto```，其中```volatile```是指示编译器对此汇编代码不进行优化，```inline```用于指示编译器尽可能小的假设汇编代码指令的大小，```goto```用于汇编代码可以跳转到一个或多个C标签。

* **AssemblerTemplate**:汇编指令，在使用中需要用双引号包含起来，若存在多条指令，每条指令后需加入```\n\t```分开。

* **OutputOperands**：输出操作数，内联汇编代码执行时，返回给C程序保存，当存在多个输出参数时，使用逗号分开。格式为：```[ [asmSymbolicName] ] constraint (cvariablename)```
    * **asmSymbolicName**：符号名，任何允许的C变量均是被允许的，同一语句中两个操作数不能使用相同的符号名。
    * **constraint**：表示约束，可选值参见表1。
    * **cvariablename**：C语言变量名。
* **InputOperands**：输入操作数，从哪里传入操作数据。格式为：```[ [asmSymbolicName] ] constraint (cexpression)```。
    * **asmSymbolicName**：是符号名，任何有效的C变量名均可被接受，同一语句中两个操作数不能使用相同的符号名。当不使用时，可以不写。
    * **constraint**：表示约束。
    * **cexpression**：C语言表达式。
* **Clobbers**：在内联汇编中，对于输出操作数所涉及的寄存器、内存，做出了修改，```Clobbers``` 用于表示修改的内容是什么，可选值见表3。

不同平台的内联汇编会有不同，称为汇编方言，具体个平台的差异可见[GNU C扩展汇编](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#Clobbers-and-Scratch-Registers)。

---

<center>表1 constraint可选值 </center>

| constraint | 描述 |
| -- | -- |
| m | memory operand 表示要传入有效的地址，只要CPU能支持该地址，就可传入。 |
| r | register operand 寄存器操作数，使用寄存器来保存这些操作数。 |
| i | immediate integer operand 表示可以传入一个立即数。 |

---

<center>表2 constrain修饰符 </center>

| constraint Modifier Characters | 描述                                                                                                                     |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| =                              | 只写操作数，表示内联汇编会修改这个操作数。                                                                                                  |
| &                              | 意味着这个操作数为一个早期的改动操作数，其在该指令完成前通过使用输入操作数被修改了。因此，这个操作数不可以位于一个被用作输出操作数或任何内存地址部分的寄存器。如果在旧值被写入之前它仅用作输入而已，一个输入操作数可以为一个早期改动操作数。 |
| +                              | 可读可写操作数，只能用来修饰输出操作数。                                                                                                   |
| 无                              | 默认操作是只读。                                                                                                               |

更多修饰符见 [http://hehezhou.cn/gccint/Modifiers.html#Modifiers](http://hehezhou.cn/gccint/Modifiers.html#Modifiers)。

---

<center>表3 Clobbers可选值 </center>

| Clobbers | 描述                 |
| -------- | ------------------ |
| cc       | 表示汇编代码会修改条件（标志）寄存器 |
| memory   | 表示汇编代码中，会修改内存。     |

参考例子
```c
#define __ASM_STR(x)  #x

#define csr_write(csr,val)                              \
({                                                      \
    unsigned long __val = (unsigned long)(val);         \
    __asm__ __volatile__("csrw "__ASM_STR(csr) ", %0"   \
                        :                               \
                        : "rK"(__val)                   \
                        : "memory");                    \
})

#define csr_read_set(csr,val)                               \
({                                                          \
    register unsigned long __v = val;                       \
    __asm__ __volatile__("csrrs %0,"__ASM_STR(csr) ", %1"   \
                        : "=r"(__v)                         \
                        : "rK"(__v)                         \
                        : "memory");                        \
    __v;
})

```

# 参考

[GNU C扩展汇编](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#Clobbers-and-Scratch-Registers)

[ARM GCC 内嵌（inline）汇编手册](http://blog.chinaunix.net/uid-20543672-id-3194385.html)

[GCC 内联汇编](https://www.cnblogs.com/kongj/p/14031770.html)

---

🧇

🍙

🍜

