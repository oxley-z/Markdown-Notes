
# 玄铁RISC-V处理器软件生态


## 开发环境

### CDS
![请添加图片描述](https://img-blog.csdnimg.cn/c40b37e4b43a48aa93874654cdb4ae43.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBARWRkeV9s,size_1,color_FFFFFF,t_70,g_se,x_16#pic_center)
<center>CDS 开发环境</center>

&emsp;&emsp;剑池CDS是面向平头哥全系列CPU的一站式开发工具，主要基于Eclipse框架，Eclipse插件开发的方式实现。在产品使用体验上，更符合Eclipse风格的开发者偏好，CDS包含了T-Head的全部系列的CPU，支持从裸板程序到嵌入式Linux应用程序的开发，支持图形化的Trace/Profiling，支持RTOS的图形化的配置。通过简单易用的图形化配置系统，让芯片开发变得简单、高效。
### CDK
![在这里插入图片描述](https://img-blog.csdnimg.cn/7b4b7eb674d14f1aa72d52cb93d07a47.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBARWRkeV9s,size_1,color_FFFFFF,t_70,g_se,x_16#pic_center)
<center>CDK 开发环境</center>

&emsp;&emsp;剑池CDK以极简开发为理念，是专业为 IoT 应用开发打造的集成开发环境。适用于MCU类型的开发者使用，它风格简洁，与市面主流的MCU类开发工具的操作习惯贴合，因此非常适合MCU、IOT设备应用开发。
### QEMU模拟器
![请添加图片描述](https://img-blog.csdnimg.cn/41f5d54f0f1a4efba05559eaf94223e1.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBARWRkeV9s,size_1,color_FFFFFF,t_70,g_se,x_16#pic_center)


&emsp;&emsp;平头哥完善了CDS开发环境，内置了 [QEMU模拟器](https://github.com/T-head-Semi/qemu?spm=a2cl5.25269445.0.0.5da51f9cK2MHqC)，可以在线仿真，便于学习及仿真。

---

## 操作系统

### Linux
![在这里插入图片描述](https://img-blog.csdnimg.cn/07ca9bcd7d724005b19d2652ec59c551.png#pic_center)
<center>Linux</center>

&emsp;&emsp;平头哥提供了用于构建 Linux 系统的开源工具 [Buildroot](https://github.com/T-head-Semi/buildroot?spm=a2cl5.25269445.0.0.5da51f9cmnwpzo) 以及 [Yocto](https://github.com/T-head-Semi/linux?spm=a2cl5.25269445.0.0.5da51f9cmnwpzo)。

&emsp;&emsp;2021 年 12 月 29 日，在 [优麒麟](https://www.ubuntukylin.com/index-cn.html) 社区和海河实验室研发团队的共同努力下，首个支持 RISC-V 架构的 Ubuntu Kylin 20.04 Pro 发布。

### Android
![在这里插入图片描述](https://img-blog.csdnimg.cn/fe5d130a5d5b4a7a86289ef3dcccd914.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBARWRkeV9s,size_1,color_FFFFFF,t_70,g_se,x_16#pic_center)
<center>Android</center>

&emsp;&emsp;平头哥提供用于构建 Android 操作系统的 [AOSP on RISC-V](https://github.com/T-head-Semi/riscv-aosp?spm=a2cl5.25269445.0.0.5da51f9cmnwpzo) 开源项目。
### RTOS
![在这里插入图片描述](https://img-blog.csdnimg.cn/a273f896e280468c871a0a4b223e5757.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBARWRkeV9s,size_1,color_FFFFFF,t_70,g_se,x_16#pic_center)
<center>FreeRTOS</center>

#### FreeRTOS
&emsp;&emsp;C910 Smart 平台提供了移植完成的 FreeRTOS 10.3.1 源码。


#### RT-Thread

![请添加图片描述](https://img-blog.csdnimg.cn/5cdb504f3f304e4aa964fdf189e50bc2.png#pic_center)

<center>RISC-V & RT-Thread</center>

&emsp;&emsp;2021年10月 RT-Thread 宣布加入 RISC-V 基金会。同时在中国科学院软件研究所支持下完成 RV64 异构多核处理器下实现RT-Thread和Linux 同时运行，并开源了相关代码 [仓库](https://github.com/RT-Thread/rtthread-pomegranate)

#### AliOS
![在这里插入图片描述](https://img-blog.csdnimg.cn/507685c8d8304f0ab2263865a4652d0e.png#pic_center)
<center>AliOS</center>

&emsp;&emsp;提供基于 [AliOS](https://github.com/T-head-Semi/yoc-open?spm=a2cl5.25269445.0.0.5da51f9cZyPNEL) 的RISC-V RTOS开源项目

## 工具

#### RISC-V GCC工具链
* [GCC编译器](https://github.com/T-head-Semi/gcc?spm=a2cl5.25269445.0.0.5da51f9cmnwpzo)
基于GCC并为玄铁处理器优化的编译套件。

* [Binutils/GDB](https://github.com/T-head-Semi/binutils-gdb?spm=a2cl5.25269445.0.0.5da51f9cmnwpzo)
汇编器、链接器、调试器等二进制工具集。

* [Newlib](https://github.com/T-head-Semi/newlib?spm=a2cl5.25269445.0.0.5da51f9cmnwpzo)
用于嵌入式系统的轻量级C库。

* [Glibc](https://github.com/T-head-Semi/glibc?spm=a2cl5.25269445.0.0.5da51f9cmnwpzo)
通用标准C库，一般用于Linux系统。

#### U-Boot
&emsp;&emsp;平头哥开源了支持玄铁处理器的 [U-Boot 开源项目](https://github.com/T-head-Semi/u-boot?spm=a2cl5.25269445.0.0.5da51f9cmnwpzo)。

#### RISC-V AI工具 - TVM
* [TVM](https://github.com/T-head-Semi/tvm?spm=a2cl5.25269445.0.0.5da51f9cK2MHqC)
AI编译器，用于自动化针对芯片生成高效推理引擎的工具集。

* [CSI-NN2](https://gitee.com/hhb-tools/csi-nn2?spm=a2cl5.25269445.0.0.5da51f9cK2MHqC)
神经网络算子库，包含了主流神经网络算子的优化实现。

