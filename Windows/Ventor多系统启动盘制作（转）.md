Title: 亲历打造Ventoy多功能多系统启动U盘，解决各种Windows 系统安装维护难题！(亲测保姆)

URL Source: https://zhuanlan.zhihu.com/p/17040276952

Markdown Content:

# 亲历打造Ventoy多功能多系统启动U盘，解决各种Windows 系统安装维护难题！(亲测保姆)

缘起，在之前的教程中，尝试整合不同版本的Windows系统，以简化装机流程。但许多朋友的老旧机型使用仅兼容BIOS启动的Windows XP，在合并后使用EFI引导时遇到了诸多问题，调试过程非常复杂，着实秃头…。

为了解决这些问题，决定另辟蹊径，为大家介绍 Ventoy 这一简单快捷的系统多功能启动U盘制作。希望这个方法能简化装机流程，解决兼容性问题，尤其对经常装机或者碰到不同新旧机器较多的朋友。

[![点击可播放视频](./image/Ventor多系统启动盘制作（转）/v2-600a137e22a5d6711c6ad0081877165d.jpeg)亲历打造Ventoy多功能多系统启动U盘，解决Windows系统安装维护难题！206 播放 · 0 赞同 视频](https://www.zhihu.com/zvideo/1860251961267478528)

![](./image/Ventor多系统启动盘制作（转）/v2-2f5ed18e491a3fd657a4899888210495.jpeg)

04:13

亲历打造Ventoy多功能多系统启动

**问题概述：**

如何使用Ventoy制作全能多系统启动U盘，集系统安装ISO、系统维护PE、[WinToGo](https://zhida.zhihu.com/search?content_id=252429915&content_type=Article&match_order=1&q=WinToGo&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NjU1ODkzMjIsInEiOiJXaW5Ub0dvIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6MjUyNDI5OTE1LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.i5PetnFA5_7W_XqAro2B3jLQwThNZ9ovajF_tiJzuOQ&zhida_source=entity)（口袋）随身系统。

## Ventoy多功能系统启动U盘优势：

1、操作简单；

2、支持多种格式镜像文件（ISO/WIM）；

3、塞入多种PE工具箱维护系统；

4、将WinToGo(Win系统)直接塞入Ventoy中；

5、应对不同新老旧设备不同系统不同需求；

6、支持多种操作系统（win/Linux/Unix/Ubuntu）等等。

## **初级应用**

**操作方法：**

### **第一步、准备工作**

1、下载安装Ventoy 多系统工具（文末下载）。

官网

```text
https://gitee.com/longpanda/Ventoy.git
```

![](./image/Ventor多系统启动盘制作（转）/v2-aee68104d8f6526da6cee9f0c0c1e458.jpeg)

2、准备一个至少16G的3.0 串口U盘。

![](./image/Ventor多系统启动盘制作（转）/v2-bc61725e3b697930e81a10c8e607730d.png)

3、准备PE工具（微**PE**&**[HotPE](https://zhida.zhihu.com/search?content_id=252429915&content_type=Article&match_order=1&q=HotPE&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NjU1ODkzMjIsInEiOiJIb3RQRSIsInpoaWRhX3NvdXJjZSI6ImVudGl0eSIsImNvbnRlbnRfaWQiOjI1MjQyOTkxNSwiY29udGVudF90eXBlIjoiQXJ0aWNsZSIsIm1hdGNoX29yZGVyIjoxLCJ6ZF90b2tlbiI6bnVsbH0.0edmyVZEaVHljSh-m20sAKkPRWuRf4tS0op-uKw7f8Q&zhida_source=entity)**）版本根据自己的需求选择（文末下载）最近准备自己DIY一个纯净PE给各位粉丝老铁们，给自己先埋个雷 。

![](./image/Ventor多系统启动盘制作（转）/v2-469707fb55e9961de6a2e71f5707a491.jpeg)

4、准备不同版本的Win系统&ubuntu系统的ISO镜像文件。

![](./image/Ventor多系统启动盘制作（转）/v2-6c8ea06a3883f5953c6f81c2aad31fe9.jpeg)

### **第二步、安装Ventoy** **到U盘**

### **操作方法：**

1、插上准备好的U盘。

2、在下载的Ventoy中找到Ventoy2Disk 应用程序，启动安装，在安装界面选择插入的 U 盘，点击安装（操作很简单）。

**注意：配置选项检查是否把** **“支持安全启动”勾选。**

![](./image/Ventor多系统启动盘制作（转）/v2-fb3375ec7b0e01a63b0a517f070d3990.jpeg)

3、安装之前会格式化 U 盘 ,注意备份数据（格式化将分为两个区，存放区1 NTFS和引导区2 GPT）。

![](./image/Ventor多系统启动盘制作（转）/v2-b05d8a7207cfa50dccd9da41a42a076a.jpeg)

4、打开等待30秒 即可安装完成。

![](./image/Ventor多系统启动盘制作（转）/v2-af2f0afc683af9cfc73cebd538f80fe3.png)

5、查看U盘 名称 变为“**Ventoy**” 说明安装完成。

![](./image/Ventor多系统启动盘制作（转）/v2-01f30348fea2d814ce782f5a965c906b.jpeg)

6、初始化U 盘 Ventoy 分区

**警告：Ventoy 分区** **配置文件系统为NTFS ，否则后续操作容易出错。**

![](./image/Ventor多系统启动盘制作（转）/v2-7f218287cfd1d975154178a6296987f6.jpeg)

**注意：如果只是单纯需要安装系统，只需将系统的镜像ISO文件拷贝到“Ventoy” 下根目录中或任意子目录就可以直接装系统啦！**

**7、重启电脑将U盘设置为第一启动盘即可进入Ventoy** **。**

![](./image/Ventor多系统启动盘制作（转）/v2-bb72b59fb7f6a7ee0e4fb0d572e2f142.jpeg)

而爱折腾的摄影大叔想要分享的是打造一个多功能系统启动盘，它即可安装系统还能进行日常系统维护那么接着往下看!

### **第三步、集成多版PE工具到Ventoy** **中**

将前面准备的 **[微PE](https://zhida.zhihu.com/search?content_id=252429915&content_type=Article&match_order=1&q=%E5%BE%AEPE&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NjU1ODkzMjIsInEiOiLlvq5QRSIsInpoaWRhX3NvdXJjZSI6ImVudGl0eSIsImNvbnRlbnRfaWQiOjI1MjQyOTkxNSwiY29udGVudF90eXBlIjoiQXJ0aWNsZSIsIm1hdGNoX29yZGVyIjoxLCJ6ZF90b2tlbiI6bnVsbH0.LT4r0rNnQR3zrlKVcm25jR4xoRwFga9_oCqNmSxzzQI&zhida_source=entity)**&**HotPE** 安装到 Ventoy 中，便于后期管理不同电脑硬件，也可放置其它Pe，这里PE侧重不同设备维护。

### **操作方法：**

1、将微PE 以ISO的方式 安装到**Ventoy**中。

1.1、启动微PE 工具箱（WePE_V2.3 **64**位 或**32**位 ），在微PE 界面右下角点击光盘图标，生成可启动ISO 文件。

![](./image/Ventor多系统启动盘制作（转）/v2-0674aa25118c2928c05818d18293013b.jpeg)

1.2、ISO输入位置 U 盘的Ventoy盘中，建议如下图勾选即可，设置完成后点击“理解生成ISO”。

![](./image/Ventor多系统启动盘制作（转）/v2-58dc38d8bad681f5500c0ff5310bb75f.jpeg)

1.3、等待生成完成即可。

![](./image/Ventor多系统启动盘制作（转）/v2-9eb25bf58eb8a1b08bd660b410fdd47c.jpeg)

2、将HotPE 以ISO 的方式安装到 “**Ventoy**” U盘。

2.1、启动HotPE 中的 “HotPE Client” 安装文件，在首页下载必要文件。

![](./image/Ventor多系统启动盘制作（转）/v2-c0922c2a008686438820f4d36b0bdffa.jpeg)

2.2、下载完成后，选择 安装 选项下 “生成ISO 镜像” 点击 开始生成，选择插入的U 盘“**Ventoy**”，点击保存等待生成完成。

![](./image/Ventor多系统启动盘制作（转）/v2-694ee25743d0d9611806097ead402d40.jpeg)

3、检查 U 盘中 微 PE &HotPe 文件。

![](./image/Ventor多系统启动盘制作（转）/v2-22301c87881c78a73c926049ab8125aa.jpeg)

### **第四步、添加ISO 文件到Ventoy** **中**

**警告：Ventoy** **不直接支持Win XP 系统的ISO 镜像文件，使用Ventoy中安装的PE工具启动后再进行安装Win XP 系统。**

### **操作方法：**

1、官方给出的测试支持1221个镜像文件，涵盖了大部分市面上可见的系统ISO 文件，如Windows、Linux、Ubuntu等操作系统。

https://ventoy.net/cn/isolist.html

![](./image/Ventor多系统启动盘制作（转）/v2-ce32779cd47b5af68870becb7cf5ca8f.jpeg)

2、而爱折腾的摄影大叔在 Ventoy 根目录下新建了子目录ISO 文件夹，在ISO 文件夹中添加了win 系统的各版本本系统。

![](./image/Ventor多系统启动盘制作（转）/v2-efdaff51ab67e34066a32b3e6c7ae47a.jpeg)

## **进阶应用**

### **进阶一：添加支持WIM安装插件**

Ventoy 还支持直接使用\*\*\*.wim 文件安装系统(Legacy BIOS + UEFI)，wim 文件需要含有引导文件，使用Wim安装的优势是节省内存及磁盘空间。

### **操作方法：**

1、下载ventoy_wimboot.img 插件文件(文末下载)。

2、在Ventoy 中U盘中手动新创建一个名为“ventoy”的文件夹（注意是小写，默认没有这个文件夹目录）。路径为“/ventoy/ventoy_wimboot.img”、

![](./image/Ventor多系统启动盘制作（转）/v2-d17cbcb41275bd5ec37954267e7df1be.jpeg)

3、将下载“ventoy_wimboot.img”文件拷贝到U盘新建的“ventoy”文件夹中。

![](./image/Ventor多系统启动盘制作（转）/v2-cac91266a989877ba0d90ef6a149b320.jpeg)

4、将自己封装的wim格式系统文件拷贝到新建的“ventoy”文件夹中即可。

![](./image/Ventor多系统启动盘制作（转）/v2-a55a73d3104419d7f0701db659a3012a.jpeg)

**关于自己如何封装包含驱动及常用软件的Windows系统WIM** **文件后面有机会再介绍。**

**安装演示：**

重启电脑，进入U盘 Ventoy界面，重启电脑进U盘即可进入Ventoy。

![](./image/Ventor多系统启动盘制作（转）/v2-793362790d0bf2b2dac2b62bfbb677a9.jpeg)

### **进阶二：ventoy主题美化**

如何使用ventoy主题美化及配置ventoy主题方法！

### **操作方法：**

### 第一步：下载支持 Ventoy 的 GRUB 主题，挑选自己心仪的主题下载。

官网

https://www.gnome-look.org/browse/

1、进入后选择[GRUB Themes](https://zhida.zhihu.com/search?content_id=252429915&content_type=Article&match_order=1&q=GRUB+Themes&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NjU1ODkzMjIsInEiOiJHUlVCIFRoZW1lcyIsInpoaWRhX3NvdXJjZSI6ImVudGl0eSIsImNvbnRlbnRfaWQiOjI1MjQyOTkxNSwiY29udGVudF90eXBlIjoiQXJ0aWNsZSIsIm1hdGNoX29yZGVyIjoxLCJ6ZF90b2tlbiI6bnVsbH0.EnL6aVwWr7AJpEN5K20eRoEyxIUBsZQDM_krTV0eX2o&zhida_source=entity) 。

![](./image/Ventor多系统启动盘制作（转）/v2-3d7def7f1c415da21339a74628c8db47.jpeg)

2、选择自己喜欢的主题。

![](./image/Ventor多系统启动盘制作（转）/v2-561b089d3c5e4906c50cfe4718d1ac99.jpeg)

3、下载喜欢的主题。

![](./image/Ventor多系统启动盘制作（转）/v2-de981230be98a85bc78f7abfe66c532b.jpeg)

### 第二步：添加 主题到Ventoy。

1、插上 U 盘，在 Ventoy 分区新建小写的 ventoy 文件夹。

![](./image/Ventor多系统启动盘制作（转）/v2-7fb412dce3190cbc1cca8fbf1b24da5e.jpeg)

2、于 ventoy 文件夹下添加 themes 文件。

![](./image/Ventor多系统启动盘制作（转）/v2-bed0914ccbe401c5c71772f329ccb678.jpeg)

3、再将下载的主题拷贝进去。

![](./image/Ventor多系统启动盘制作（转）/v2-b42661cb5043a688836d22b05e4500f8.jpeg)

4、打开主题中的 theme.txt 文档，添加 title。

![](./image/Ventor多系统启动盘制作（转）/v2-135d0f02aea60ea517497792317bbb15.jpeg)

5、将下载的“**Ventoy热键显示脚本.txt**”中的内容复制到“theme.txt” 底部即可（文末下载）。

![](./image/Ventor多系统启动盘制作（转）/v2-d0fdd5f53bc7dd78b9bb2cdf0bf2f2b0.jpeg)

第三步、配置ventoy 主题必要项。

1、启动 “**VentoyPlugson.exe**” 工具。

![](./image/Ventor多系统启动盘制作（转）/v2-b2763a0dd784019689a79f0eda97ee3d.jpeg)

2、点击 “主题插件”，添加 thems 主题文档的绝对路径。

![](./image/Ventor多系统启动盘制作（转）/v2-0e425ba0f364930ea4ed1d897e3c5232.jpeg)

3、依据自己显示器及主题分辨率情况设置分辨率。

![](./image/Ventor多系统启动盘制作（转）/v2-5b44123cb7f736dac29619e352670084.jpeg)

4、把主题包中的字体 PF2 文件路径全部加载进来。

![](./image/Ventor多系统启动盘制作（转）/v2-27dd27062dcd572adf9805b94cc0ebf9.jpeg)

5、选择 “菜单类型插件”，新增添加图标关键词，（windows系统和winPE ）（区分大小写）完成后查看（根据U盘中文件名称设置，可以自定义图标）。

![](./image/Ventor多系统启动盘制作（转）/v2-e3f687be0417d9957e9247aac1c09bc9.jpeg)

6、Ventoy 就比原来美观许多（当然还可以多主体，自己研究玩吧）。

### **个性化一：**

![](./image/Ventor多系统启动盘制作（转）/v2-e8121019bd9651521ffc49f12fa9c6c4.jpeg)

### **个性化二：**

![](./image/Ventor多系统启动盘制作（转）/v2-12c3785794ec5741b85446b38e427821.jpeg)

### **扩展：ventoy多主题**

1、多分辨率多主题：根据自己显示器设置一个或多个分辨率主题，将不同分辨率主题添加进行themes 文件夹，然后重新配置即可。

![](./image/Ventor多系统启动盘制作（转）/v2-49e60d702f048b471d4e1d399d32cd3b.jpeg)

![](./image/Ventor多系统启动盘制作（转）/v2-c5454ee6cfbec54de1dabb5b0dfd81bb.jpeg)

2、多模式主题：可以分别配置 theme_legacy theme_uefi theme_ia32 theme_aa64 theme_mips，分别针对 x86 Legacy BIOS、x86_64 UEFI、IA32 UEFI、ARM64 UEFI 和 MIPS64EL UEFI 模式生效。

![](./image/Ventor多系统启动盘制作（转）/v2-a8fbc02596cc30131db1328ebb0b7765.jpeg)

## **高级应用**

### **操作方法：**

### 第六步、添加Windows系统到Ventoy 中

注意事项：不是高速SSD且空间大于128GB 的U盘不要尝试！

注意事项：不是高速SSD且空间大于128GB 的U盘不要尝试！

注意事项：不是高速SSD且空间大于128GB 的U盘不要尝试！

将Windows系统（仅Win7以上）安装到Ventoy 中，集成一个强大携带方便的移动电脑系统。需安装支持Windows VHD文件的启动插件。支持 Legacy BIOS 和 UEFI 模式。支持固定大小以及动态扩展类型的 VHD/VHDX 格式。

![](./image/Ventor多系统启动盘制作（转）/v2-dd4b805e498f32badacd69a2bc0341d2.jpeg)

1、下载支持Windows VHD 文件启动插件“**ventoy_vhdboot.img**”文件（文末）。

开源下载:

```text
https://github.com/ventoy/vhdiso/releases
```

2、在Ventoy 中U盘中手动新创建一个名为“ventoy”的文件夹（注意是小写，默认没有需手动创建），路径为“/ventoy/ventoy_vhdboot.img”。。

![](./image/Ventor多系统启动盘制作（转）/v2-d7bedceb166daea95edab55504132eda.jpeg)

3、使用[VirtualBox](https://zhida.zhihu.com/search?content_id=252429915&content_type=Article&match_order=1&q=VirtualBox&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NjU1ODkzMjIsInEiOiJWaXJ0dWFsQm94IiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6MjUyNDI5OTE1LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.Zo4f3SQyy_wI52j9E6D_heNYEAItNcG2WeFCAICshMM&zhida_source=entity) 虚拟机创建Windows 10的VHD 文件。

具体关于VirtualBox 如何使用查看下文：

### 创建Virtualbox虚拟机

3.1、打开VirtualBox 虚拟机 创建虚拟电脑，设置名称: win1022H2_ventoy，路径在本地磁盘。

![](./image/Ventor多系统启动盘制作（转）/v2-e7f5a54f15b7d4bd102a1fd6d6af7df6.jpeg)

3.2、注意勾选 启用EFI,使既能支持Legacy BIOS 模式启动，也能支持 UEFI 模式启动。

![](./image/Ventor多系统启动盘制作（转）/v2-339703f2f6b3ab3fa9c784cadb2392f5.jpeg)

3.3、并创建“**(VHD)虚拟硬盘**”，空间大小在50Gb左右（不小于40GB）,根据需求调整，位置路径设置在本地磁盘中，制作完成后需要拷贝win1022H2_ventoy.vhd 文件到U盘中。

![](./image/Ventor多系统启动盘制作（转）/v2-7391137793d6dfbd5d742ef36bbdeff4.jpeg)

4、使用VirtualBox 虚拟机将Windows10 系统安且装激活完成后，设置及安装必备使用软件后，正常关闭虚拟机电脑。再将“win1022H2_ventoy.vhd” 拷贝到U 盘“ventoy” 目录下。

![](./image/Ventor多系统启动盘制作（转）/v2-f043fad7ada71a3761473943ab550dcc.jpeg)

5、测试，重启电脑选择U盘启动，选择VHD 系统文件启动。

![](./image/Ventor多系统启动盘制作（转）/v2-6840de770205c6268e5ae1db3c7d4b71.jpeg)

6、注意：VHD虚拟磁盘如果不是NTFS一般很难启动!

![](./image/Ventor多系统启动盘制作（转）/v2-fdb6178376cf7f7a9d81d9d43acbe2c9.jpeg)

好了，本文终于完成，断断续续制作测试好几天！

关于更多ventoy使用

```text
https://ventoy.net/cn/doc_news.html
```

发布于 2025-01-08 10:32・河南

[系统盘装机](//www.zhihu.com/topic/25944492)

[U盘启动](//www.zhihu.com/topic/19819313)

[ventoy](//www.zhihu.com/topic/26141708)

赞同 288 条评论

分享

喜欢收藏申请转载