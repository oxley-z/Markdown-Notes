

# Sub-process /usr/bin/dpkg returned an error code (1)

```bash
dpkg: error processing package libc-bin (--configure):
 installed libc-bin package post-installation script subprocess returned error exit status 134
Errors were encountered while processing:
 libc-bin
E: Sub-process /usr/bin/dpkg returned an error code (1)
```

解决方法

```bash
sudo mv /var/lib/dpkg/info/libc-bin.* /tmp/
sudo dpkg --remove --force-remove-reinstreq libc-bin
sudo dpkg --purge libc-bin
sudo apt install libc-bin
sudo mv /tmp/libc-bin.* /var/lib/dpkg/info/
```

参考[libc-bin update error · Issue #4760 · microsoft/WSL --- libc-bin 更新错误 · 问题 #4760 · microsoft/WSL (github.com)](https://github.com/microsoft/WSL/issues/4760)

# /bin/bash: warning: setlocale: LC_ALL: cannot change locale (en_US.UTF-8)

```bash
sudo localedef -i en_US -f UTF-8 en_US.UTF-8
```

# 发生错误 - Microsoft-Windows-GroupPolicy-ClientExtensions-Package 错误: 0x8000ffff

解决方法

[gpedit.msc安装失败 - Microsoft Community](https://answers.microsoft.com/zh-hans/windows/forum/all/gpeditmsc安装失败/ca9f884f-2759-47ba-88fe-749d8a800951)

管理员打开cmd，执行`DISM /Online /Cleanup-Image /RestoreHealth`，修复系统文件，之后执行`sfc /scannow`扫描系统损坏的文件。

# Windows10专业版 找不到文件'gpedit.msc'。请确定文件名是否正确后，再试一次。

解决方法

[windows找不到gpedit.msc请确定文件名-CSDN博客](https://blog.csdn.net/qq_42676511/article/details/122715711)

新建文件`gpedit.txt`，键入以下内容。

```txt
@echo off

pushd "%~dp0"

dir /b C:\Windows\servicing\Packages\Microsoft-Windows-GroupPolicy-ClientExtensions-Package~3*.mum >List.txt

dir /b C:\Windows\servicing\Packages\Microsoft-Windows-GroupPolicy-ClientTools-Package~3*.mum >>List.txt

for /f %%i in ('findstr /i . List.txt 2^>nul') do dism /online /norestart /add-package:"C:\Windows\servicing\Packages\%%i"

pause

```

修改后缀为`.bat`。管理员运行CMD，执行`start gpedit.bat`，等待完成，完成之后即可打开`gpedit.msc`。

# WSL迁移

[# Ubuntu迁移记录](https://blog.csdn.net/u013097500/article/details/163096983?spm=1001.2100.3001.7377&utm_medium=distribute.pc_feed_blog.none-task-blog-personrec_tag-1-163096983-null-null.nonecase&depth_1-utm_source=distribute.pc_feed_blog.none-task-blog-personrec_tag-1-163096983-null-null.nonecase)
