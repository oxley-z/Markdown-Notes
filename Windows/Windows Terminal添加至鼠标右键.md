---
github.io: true
csdn: false
---
# Windows Terminal添加至鼠标右键

## 安装

在`Microsoft Store`中即可下载。

![image-20230619110413661](image/Windows%20Terminal%E6%B7%BB%E5%8A%A0%E8%87%B3%E9%BC%A0%E6%A0%87%E5%8F%B3%E9%94%AE/image-20230619110413661.png)

## 配置

### 在鼠标右键打开

#### 下载Terminal图标

图标地址:https://raw.githubusercontent.com/microsoft/terminal/master/res/terminal.ico

下载后保存在某个文件夹`path`

#### 添加到鼠标右键

##### 批处理修改注册表

```bash
@echo off
reg.exe add "HKEY_CLASSES_ROOT\Directory\Background\shell\wt" /f /ve /d "Windows Terminal here"
reg.exe add "HKEY_CLASSES_ROOT\Directory\Background\shell\wt" /f /v "Icon" /t REG_EXPAND_SZ /d "path\terminal.ico"
reg.exe add "HKEY_CLASSES_ROOT\Directory\Background\shell\wt\command" /f /ve /t REG_EXPAND_SZ /d "\"%%LOCALAPPDATA%%\Microsoft\WindowsApps\wt.exe\""
pause
```

##### 手动修改注册表

按下Win+R快捷键打开运行,然后输入`regedit`打开**注册表编辑器**,然后在地址栏中输入:

```text
计算机\HKEY_CLASSES_ROOT\directory\background\shell
```

![image-20230619111056009](image/Windows%20Terminal%E6%B7%BB%E5%8A%A0%E8%87%B3%E9%BC%A0%E6%A0%87%E5%8F%B3%E9%94%AE/image-20230619111056009.png)

新建**项**:`wt`

**点击**wt，**双击**右边的**默认**，在里面输入鼠标右键选项的名称：`Windows Terminal here`

右键wt，选择新建字符串，名为为`Icon`，在里面输入下载好的图标的地址：`path\terminal.ico`

右键wt，选择新建项,输入command

然后点击command,双击右边的默认,输入Terminal的地址`"%%LOCALAPPDATA%%\Microsoft\WindowsApps\wt.exe\"`即可。

![image-20230619111734537](image/Windows%20Terminal%E6%B7%BB%E5%8A%A0%E8%87%B3%E9%BC%A0%E6%A0%87%E5%8F%B3%E9%94%AE/image-20230619111734537.png)

#### 在当前目录打开

##### 打开配置文件

点击Windows Terminal菜单栏上的**下拉箭头**,然后选择Settings:

![image-20230619110508615](image/Windows%20Terminal%E6%B7%BB%E5%8A%A0%E8%87%B3%E9%BC%A0%E6%A0%87%E5%8F%B3%E9%94%AE/image-20230619110508615.png)

![image-20230619112038125](image/Windows%20Terminal%E6%B7%BB%E5%8A%A0%E8%87%B3%E9%BC%A0%E6%A0%87%E5%8F%B3%E9%94%AE/image-20230619112038125.png)

在默认打开的终端配置中添加`"startingDirectory" : ".",`这个配置即可:

![image-20230619112140525](image/Windows%20Terminal%E6%B7%BB%E5%8A%A0%E8%87%B3%E9%BC%A0%E6%A0%87%E5%8F%B3%E9%94%AE/image-20230619112140525.png)