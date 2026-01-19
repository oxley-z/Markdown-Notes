# 1、VMWare安装出现`Failed to install the hcmon driver`

![image-20230710084706295](image/Windows%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98%E8%A7%A3%E5%86%B3%E6%96%B9%E6%B3%95/image-20230710084706295.png)

解决方法：

新建文本文件输入以下内容：

```bash
Windows Registry Editor Version 5.00
 
[HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion]
"ProgramFilesDir"="C:\\Program Files"
"CommonFilesDir"="C:\\Program Files\\Common Files"
"ProgramFilesDir (x86)"="C:\\Program Files (x86)"
"CommonFilesDir (x86)"="C:\\Program Files (x86)\\Common Files"
"CommonW6432Dir"="C:\\Program Files\\Common Files"
"DevicePath"=hex(2):25,00,53,00,79,00,73,00,74,00,65,00,6d,00,52,00,6f,00,6f,\
  00,74,00,25,00,5c,00,69,00,6e,00,66,00,00,00
"MediaPathUnexpanded"=hex(2):25,00,53,00,79,00,73,00,74,00,65,00,6d,00,52,00,\
  6f,00,6f,00,74,00,25,00,5c,00,4d,00,65,00,64,00,69,00,61,00,00,00
"ProgramFilesPath"=hex(2):25,00,50,00,72,00,6f,00,67,00,72,00,61,00,6d,00,46,\
  00,69,00,6c,00,65,00,73,00,25,00,00,00
"ProgramW6432Dir"="C:\\Program Files"
```

将文本文件后缀改为`.reg`后执行该文件。

将vmware安装位置配置为默认的C盘。

# 2、Windows Terminal

[消除WSL中ls Windows文件夹时背光配色的方法_wsl字体颜色_天色微凉的博客-CSDN博客](https://blog.csdn.net/qq_33882435/article/details/116264702)

[去掉ubuntu下windows文件夹绿色背景_有硬币就是土豪的博客-CSDN博客](https://blog.csdn.net/gengli2017/article/details/82917827)

# 3、禁用驱动程序强制签名

打开设置（win+i）->更新和安全->恢复->高级启动->立即重新启动->疑难解答->高级选项->启动设置->重启->根据提示输入`7`禁用驱动程序强制签名

# 4、添加VSCode至鼠标右键

新建文件键入如下内容

```cmd
Windows Registry Editor Version 5.00

[HKEY_CLASSES_ROOT\*\shell\VSCode]
@="Open with Code"
"Icon"="D:\\VScode\\Microsoft VS Code\\Code.exe"

[HKEY_CLASSES_ROOT\*\shell\VSCode\command]
@="\"D:\\VScode\\Microsoft VS Code\\Code.exe\" \"%1\""

Windows Registry Editor Version 5.00

[HKEY_CLASSES_ROOT\Directory\shell\VSCode]
@="Open with Code"
"Icon"="D:\\VScode\\Microsoft VS Code\\Code.exe"

[HKEY_CLASSES_ROOT\Directory\shell\VSCode\command]
@="\"D:\\VScode\\Microsoft VS Code\\Code.exe\" \"%V\""

Windows Registry Editor Version 5.00

[HKEY_CLASSES_ROOT\Directory\Background\shell\VSCode]
@="Open with Code"
"Icon"="D:\\VScode\\Microsoft VS Code\\Code.exe"

[HKEY_CLASSES_ROOT\Directory\Background\shell\VSCode\command]
@="\"D:\\VScode\\Microsoft VS Code\\Code.exe\" \"%V\""
```

将上述文件内容的目录替换为本机VSCode安装目录，之后将文本文件后缀改为`.reg`后执行该文件。

# 5、win10无显示器调整分辨率

## 打开注册表

win + r 输入 regedit

## 查找键

计算机\HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\GraphicsDrivers\Configuration\SIMULATED_10DE_1288_00000003_00000000_3300^A3958568FC3B6353745639A9050D7B69\00

如果找不到`SIMULATED_10DE_1288_00000003_00000000_3300^A3958568FC3B6353745639A9050D7B69\00` 桌面鼠标右键查看当前设置的分辨率是多少，然后在`Configuration\xxxxxxxxxxxx\00`下找到对应的以下Key值。

## 修改Key

宽 PrimSurfSize.cx

高 PrimSurfSize.cy

# 6、Windows共享Wifi给其他通过网线连接的电脑

## 步骤1

将本地的被共享的以太网口IP设置为自动获取

## 步骤2

网络更改适配器设置中

![image-20250613174733171](image/Windows%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98%E8%A7%A3%E5%86%B3%E6%96%B9%E6%B3%95/image-20250613174733171.png)

允许其他网络用户通过此计算机的Internet连接来连接。

此时需将被共享网络的电脑的网络配置为自动获取IP地址，此时即可共享Wifi至其他网线链接的电脑。

## 常见问题

### [目前已为连接’wireless_32768’启用Internet 连接共享。lnternet连接共享访问被启用时，出现了一个错误](https://blog.csdn.net/Y_an_Y/article/details/117266448)

先将共享设置给其他网卡，之后再设置为需要共享的网卡

### [win10无法启用internet连接共享，为lan连接配置的IP地址需要使用自动ip寻址](https://blog.csdn.net/Caoyang_He/article/details/91125678)

此问题为`192.168.137.1`地址被占用，将占用的网路的地址强制改为其他的IPv4地址即可

# 8、esd转iso

使用Dism++软件可转
