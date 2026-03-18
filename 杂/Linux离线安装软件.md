# Linux离线下载安装软件包及依赖项

## 1.下载软件包且不进行安装

```bash
# 例如使用 apt 下载 wireshark 安装包
sudo apt download wireshark
# 下载多条的时候直接使用 空格 分割即可
sudo apt download vim sshpass
```

## 2.下载所有的依赖项

### 先查询当前包所需的依赖项有哪些

```bash
# 查询包的直接依赖
$ sudo apt-cache depends vim
```

### 查询所有依赖项

```bash
# 命令可以递归地列出软件包及其所有依赖项。这对于了解软件包的完整依赖关系非常有用。
$ sudo apt-cache depends --recurse
# 查看wireshark 的所有依赖
$ sudo apt-cache depends --recurse wireshark
```

使用上述的命令会查询大量的依赖包，也包含建议安装的包，和增强的包。继续为命令添加上更多的参数，进行精准的查询;

如果你想在使用 apt-cache 命令时忽略建议、建议的依赖项、冲突、中断、替代、增强和预先依赖项，可以通过添加对应的选项来实现。

以下是对应选项的说明：

- `--no-recommends`：忽略建议的依赖项。
- `--no-suggests`：忽略建议的软件包。
- `--no-conflicts`：忽略冲突。
- `--no-breaks`：忽略中断。
- `--no-replaces`：忽略替代。
- `--no-enhances`：忽略增强。
- `--no-pre-depends`：忽略预先依赖项。

常用组合如下：

```bash
# 查找依赖包, 并且忽略冲突等信息
$ sudo apt-cache depends --recurse --no-recommends --no-suggests --no-conflicts  --no-breaks --no-replaces --no-enhances --no-pre-depends wireshark
...
libedit2
  依赖: libbsd0
  依赖: libc6
  依赖: libtinfo6
libtinfo6
  依赖: libc6
libsensors-config
<debconf-2.0>
<qtbase-abi-5-12-8>
<fonts-freefont>
```

剔除无效字符

```bash
$ apt-cache depends --recurse --no-recommends --no-suggests --no-conflicts  --no-breaks --no-replaces --no-enhances --no-pre-depends wireshark |grep  -v "^ " | grep -v '^<'
wireshark
wireshark-qt
libc6
...
libedit2
libtinfo6
libsensors-config
```

将[下载命令](#1.下载软件包且不进行安装)与[查询命令](#查询所有依赖项)结合即可下载所有依赖包：

```bash
$ sudo apt-get download $(apt-cache depends --recurse --no-recommends --no-suggests --no-conflicts  --no-breaks --no-replaces --no-enhances --no-pre-depends 包名 |grep  -v "^ " | grep -v '^<')
```

## 3. 数据包安装

```bash
# 切换到下载安装包的目录
dpkg -i *.deb
# 权限不足的时候, 加上 sudo
sudo dpkg -i *.deb
```

## 4. 一键下载脚本

将上述命令整合为shell脚本即可一次性下载某个软件包及其依赖项：

新建download.sh文件，键入以下内容：

```bash
#!/bin/bash

#$1     pkg
get_all_depends()
{
        apt-cache depends --no-pre-depends --no-suggests --no-recommends \
                --no-conflicts --no-breaks --no-enhances\
                --no-replaces --recurse $1 | awk '{print $2}'| tr -d '<>' | sort --unique
}
 
 
 
for pkg in $*
do
        all_depends=$(get_all_depends $pkg)
        echo -e "所有依赖共计"$(echo $all_depends | wc -w)"个"
        echo $all_depends
        i=0
        for depend in $all_depends
        do
                i=$((i+1))
                echo -e "\033[1;32m正在下载第$i个依赖："$depend "\033[0m"
                apt-get download $depend
        done
done
```

脚本运行

```bash
# 添加权限
$ chmod +x 
# 运行脚本下载软件包及依赖项
$ ./download.sh wireshark
```

# 参考

[Ubuntu 离线安装的常见操作](https://www.cnblogs.com/Blogwj123/p/17593096.html)