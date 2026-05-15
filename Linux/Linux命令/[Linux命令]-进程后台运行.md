---
github.io: true
csdn: false
---
# 进程后台运行

要在后台运行 Linux 命令，只需要在命令的末尾添加一个 & 符号，如下所示：

```bash
iperf -s
iperf -c 127.0.0.1 -t 10 > log.txt &
```

当命令在后台完成时，你应该能在终端上看到该命令的信息：

```bash
[1]  + 5728 done       iperf -c 127.0.0.1 -t 10 > log.txt
```

## 将正在运行的进程发送到后台

按 Ctrl + Z 键暂停当前进程，然后使用 bg（background 的缩写）向后台发送进程，该被挂起的进程就会转到后台继续运行。

```bash
sleep 30 
^z
bg
```

## 查看后台运行的所有进程

```bash
jobs
```

## 将后台运行的进程转至前台

要恢复后台进程，可使用 fg（foreground的简写） 命令：

```bash
fg
```

单单使用 fg 命令而不带任何参数，会把后台作业列表中的最后一个进程带到前台。

如果我们要将某个指定的进程带到前台，还需要指定作业的 id。这个 id 就是我们前面提到的，使用 jobs 命令的输出中，每一行最前面的数字。

```bash
fg n
```

### 后台运行某个命令

```bash
echo iperf -c 127.0.0.1 -t 10 -l 256 >> iperf_Test.log | iperf -c 127.0.0.1 -t 10 -l 256 >> iperf_test.log &
```

