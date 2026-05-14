# GDB调试

## 带参数执行

```bash
gdb --args ./A V1 V2 V3

gdb ./A，进入gdb后  run V1 V2 V3

gdb ./A，进入gdb后  设置参数set args V1 V2 V3 再直接 run。
```

## 常用命令

### 执行程序 run

缩写r运行程序，若存在断点，执行到断点处暂停，若无断点，则程序执行完成

### 查看 info

查看

```bash
查看断点
info break
查看自动显示列表
info display
查看寄存器信息
info registers
```

### 显示源码 list

缩写l查看当前文件内容，默认显示10行，可通过 `set listsize 行数` 或 `set list 行数`进行设置

```bash
显示当前位置后10行
list
显示第5行上下10行
list 5
显示函数上下文
list fun
```

### 设置断点 break

缩写b

### 删除 delete 

删除 缩写 d 

```bash
删除断点num
delete num
删除第1-5断点
delete num1-5
删除第2 4 6个断点
delete 2 4 6
```

### 打印变量 print

```bash
以fmt形式显示变量
print/fmt 变量名
```

```bash
格式化字符串(/fmt)说明
/x 十六进制显示
/d 有符号十进制显示
/u 无符号十进制显示
/o 八进制显示
/t 二进制显示
/f 浮点显示
/c 字符显示
```

### 打印变量类型 pypte、whatis

查看变量类型

```bash
pypte 变量名
whatis 变量名
```

### 自动显示 display

自动打印信息，每当程序暂停执行，都会打印一次相应变量

```bash
display/fmt 变量名
取消自动显示
undisplay num1 num2 ...
undisplay num1-numN
或
delete display num1 num2 ...
delete display num1-numN
禁用自动显示的变量
disable display num1 num2 ...
disable display num1-numN
启用自动显示的变量
enable display num1 num2 ...
enable display num1-numN
```

### 单步跳入step

单步跳入，缩写为s

```bash
单步执行
step
单步汇编
stepi
```



### 跳出函数体 finish

跳出当前函数体

### 单步跳过 next

```bash
单步汇编
nexti
```



### until

跳出某个循环体，需满足：循环体内不存在断点、须在循环体开始/结束行执行该命令。

```bash
执行程序至test.c第23行
until test.c:23
```

### 启用 enable

```bash
开启第n个断点
enable n
```

### 禁用

```bash
禁用断点n
disable n
```

### 调用函数 call

```bash
调用函数fun并传参25
call fun(25)
```

### 查看函数参数 frame

```bash
frame N
```

### 设置变量值 set

```bash
set var 变量名=值
```

### continue

从当前停止位置开始执行，缩写c

### 查看程序调用栈信息 backtrace

查看程序调用栈信息

### 格式化显示结构体

```bash
set print pretty on
```

### 更改GDB调试源码路径

```bash
set substitute-path <当前搜索路径> <指的搜索路径>
set substitute-path /usr/src/kernel /opt/linux-xlnx-xilinx-v2021.1
```











### 退出调试 quit

缩写q

