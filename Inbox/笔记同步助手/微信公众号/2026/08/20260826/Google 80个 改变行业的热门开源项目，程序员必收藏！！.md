---
author: Debug 蟹老板
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247492104&idx=1&sn=257d3b0c7f0c1c5fbc064b29273c4d97&chksm=c3e7ddb2418d903e1770baef2b48a8f5fef99a670d69280f8ad494972cff4519a9601c9f8fdf&mpshare=1&scene=1&srcid=0826qwFnDlqTDuvMVHA25w1R&sharer_shareinfo=13136bbe4878e81e443e29ff78656247&sharer_shareinfo_first=13136bbe4878e81e443e29ff78656247#rd
saved: 2026-08-26 20:50:35
tags:
  - 笔记同步助手
id: e2842180-cfac-4a29-b0fd-5a63891e0d87
---

公众号名称：Linux教程

作者名称：Debug 蟹老板

发布时间：2026-08-26 20:18

# 大家好，我是蟹老板～

# Google 的开源生态非常庞大，其官方 GitHub 组织下拥有超过 2800 个公开仓库。

从人工智能、大模型，到云计算、数据库、移动操作系统，再到安全工具，Google 的开源项目几乎覆盖现代软件工程的核心技术方向。

很多今天已经成为行业标准的技术，都来自 Google 开源体系：

-   • TensorFlow 推动深度学习工程化；
    
-   • Kubernetes 改变云原生基础设施；
    
-   • Chromium 构建现代浏览器生态；
    
-   • Android 成为全球移动操作系统基础；
    
-   • gRPC 成为微服务通信的重要方案。
    

本文整理 **Google 热门、具有影响力的 80 个开源项目**，覆盖了AI 与机器学习、前端与UI、云计算与基础设施、数据库与存储、开发工具、移动操作系统和安全隐私等多个领域。

## 一、机器学习与人工智能（14个）

### 1\. TensorFlow

### —— Google 开源的深度学习计算框架

TensorFlow 是 Google Brain 团队推出的开源机器学习框架，是全球最具影响力的深度学习平台之一。

它提供完整的 AI 开发生态，从模型构建、训练优化，到生产部署均有支持。

![](https://relay-1.bijitongbu.site/p/cb31bdd0155da5eecef4ff3a5dfc13bc.png)

**项目地址：**

```
https://github.com/tensorflow/tensorflow
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 197.6K

**它提供什么：**

-   • 深度学习模型构建
    
-   • 自动微分计算
    
-   • GPU/TPU 加速
    
-   • 分布式训练
    
-   • 模型部署工具
    
-   • 移动端推理（TensorFlow Lite）
    

**典型应用：**

-   • 图像识别
    
-   • 语音识别
    
-   • 推荐系统
    
-   • 自动驾驶
    
-   • 医疗 AI
    
-   • 自然语言处理
    

### 2\. JAX

### —— Google 下一代高性能机器学习计算框架

JAX 是 Google Research 开发的高性能数值计算框架，将 NumPy 风格接口、自动微分和 XLA 编译技术结合，为现代 AI 研究提供高效计算能力。

![](https://relay-1.bijitongbu.site/p/f62ea73377249a8639615bf38acd0826.png)

**项目地址：**

```
https://github.com/jax-ml/jax
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 36.2K

**它提供什么：**

-   • NumPy 兼容计算
    
-   • 自动梯度计算
    
-   • XLA 编译优化
    
-   • GPU/TPU 加速
    
-   • 科学计算能力
    

**典型应用：**

-   • 大模型训练研究
    
-   • 深度学习实验
    
-   • 强化学习
    
-   • 科学计算
    
-   • AI 算法优化
    

### 3\. Gemma

### —— Google 开源大语言模型系列

Gemma 是 Google 推出的开放式大语言模型，基于 Gemini 技术体系，为开发者提供轻量化、高性能的生成式 AI 能力。

![](https://relay-1.bijitongbu.site/p/f00dc9b3db02853cbd6fb74de2f32c06.png)

**项目地址：**

```
https://github.com/google-deepmind/gemma
```

> **开源协议：** Gemma License

> **GitHub Star：** 5.7K

**它提供什么：**

-   • 开放模型权重
    
-   • 文本生成能力
    
-   • 模型微调支持
    
-   • AI 应用开发能力
    

**典型应用：**

-   • AI 助手
    
-   • 企业知识库
    
-   • 智能客服
    
-   • Agent 应用
    
-   • 文档总结
    

### 4\. MediaPipe

### —— Google 端侧 AI 应用开发框架

MediaPipe 是 Google 开源的跨平台机器学习框架，主要用于在手机、浏览器和边缘设备中实现实时 AI 感知。

![](https://relay-1.bijitongbu.site/p/bf433fb81842d607dcda26374188c845.png)

**项目地址：**

```
https://github.com/google-ai-edge/mediapipe
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 36.7K

**它提供什么：**

-   • 人脸检测
    
-   • 手势识别
    
-   • 姿态估计
    
-   • 目标追踪
    
-   • 实时视觉处理
    

**典型应用：**

-   • AR 应用
    
-   • 手机相机特效
    
-   • 虚拟试妆
    
-   • 健身动作分析
    
-   • 人机交互
    

### 5\. seq2seq

### —— Google 经典神经序列学习框架

seq2seq 是 Google 在自然语言处理领域的重要开源项目，用于解决序列到序列转换问题。

![](https://relay-1.bijitongbu.site/p/03fe051f2af7f428cc832b5b25011a0f.png)

**项目地址：**

```
https://github.com/google/seq2seq
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 5.6K

**它提供什么：**

-   • 编码器-解码器模型
    
-   • 神经机器翻译框架
    
-   • 文本生成能力
    
-   • 序列学习工具
    

**典型应用：**

-   • 机器翻译
    
-   • 文本摘要
    
-   • 对话系统
    
-   • NLP 研究
    

### 6\. Magenta

### —— Google 开源的 AI 音乐与艺术创作平台

Magenta 是 Google Brain 团队推出的人工智能艺术创作项目，主要探索机器学习在音乐、绘画和创意领域的应用。

![](https://relay-1.bijitongbu.site/p/26acf993599ee8f98ef5f244a7c3d90b.png)

它希望研究一个问题：

> 人工智能是否能够参与创造性工作？

**项目地址：**

```
https://github.com/magenta/magenta
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 19.8K

**它提供什么：**

-   • AI 音乐生成模型
    
-   • MIDI 数据处理工具
    
-   • 音频生成算法
    
-   • 艺术创作实验框架
    
-   • 深度学习创作模型
    

**典型应用：**

-   • AI 作曲
    
-   • 音乐辅助创作
    
-   • 自动旋律生成
    
-   • 艺术实验项目
    
-   • 音频智能应用
    

### 7\. Sonnet

### —— DeepMind 开源神经网络构建库

Sonnet 是 DeepMind 开发的高级神经网络库，建立在 TensorFlow 基础之上。

它主要用于研究人员快速构建复杂深度学习模型。

![](https://relay-1.bijitongbu.site/p/3405a11a39f6e04d8a6c7bdfe63c3538.png)

**项目地址：**

```
https://github.com/google-deepmind/sonnet
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 10K

**它提供什么：**

-   • 神经网络模块化设计
    
-   • 高级模型组件
    
-   • 参数管理机制
    
-   • 深度学习实验工具
    

**典型应用：**

-   • 深度学习研究
    
-   • 强化学习
    
-   • 复杂神经网络实验
    
-   • AI 算法验证
    

### 8\. Edward

### —— Google 开源概率机器学习框架

Edward 是 Google 开源的概率编程和统计机器学习框架。

它将深度学习与贝叶斯统计方法结合，用于处理复杂的不确定性问题。

![](https://relay-1.bijitongbu.site/p/81c02f635ab52621526a6f3e3cf5de66.png)

**项目地址：**

```
https://github.com/blei-lab/edward
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 4.8K

**它提供什么：**

-   • 概率模型构建
    
-   • 贝叶斯推理
    
-   • 深度概率学习
    
-   • 不确定性分析
    

**典型应用：**

-   • 数据科学研究
    
-   • 风险预测
    
-   • 概率模型分析
    
-   • 科研计算
    

### 9\. Cartographer

### —— Google 开源实时 SLAM 建图系统

Cartographer 是 Google 开发的实时定位与地图构建系统（SLAM）。

![](https://relay-1.bijitongbu.site/p/1d582c8ebf2e838e3a76500ba653c9e6.png)

它主要解决机器人领域中的核心问题：

> 机器人如何知道自己在哪里，并构建周围环境地图。

**项目地址：**

```
https://github.com/cartographer-project/cartographer
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 7.9K

**它提供什么：**

-   • 二维 SLAM
    
-   • 三维 SLAM
    
-   • 激光雷达数据处理
    
-   • 传感器融合
    
-   • 实时地图构建
    

**典型应用：**

-   • 自动驾驶
    
-   • 机器人导航
    
-   • 无人设备
    
-   • 智能仓储
    

### 10\. JAX-NeRF

### —— 基于 JAX 的神经辐射场研究项目

JAX-NeRF 是 Google 开源的 NeRF（Neural Radiance Fields）实现项目。

NeRF 是近年来计算机视觉领域的重要技术，可以通过二维图片重建三维场景。

![](https://relay-1.bijitongbu.site/p/ee415435c7cd08d45c0e699d1aa2a39c.png)

**项目地址：**

```
https://github.com/google-research/jaxnerf
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 6K

**它提供什么：**

-   • NeRF 模型实现
    
-   • 三维场景重建
    
-   • 高性能训练代码
    
-   • JAX 加速支持
    

**典型应用：**

-   • 三维建模
    
-   • 虚拟现实
    
-   • 数字孪生
    
-   • 自动驾驶视觉系统
    

### 11\. ADK-Go

### —— Google Agent 开发工具

ADK-Go 是 Google 面向 AI Agent 开发推出的工具框架，用于帮助开发者构建能够调用工具、执行任务的智能代理。

![](https://relay-1.bijitongbu.site/p/72c27cab19a02d5e6b15064ce1e5209e.png)

**项目地址：**

```
https://github.com/google/adk-go
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 8.7K

**它提供什么：**

-   • Agent 开发框架
    
-   • 工具调用机制
    
-   • 工作流管理
    
-   • 多 Agent 协作能力
    

**典型应用：**

-   • AI 助手
    
-   • 企业 Agent
    
-   • 自动化工作流
    
-   • 智能应用开发
    

### 12\. Gemini CLI

### —— Google Gemini AI 命令行工具

Gemini CLI 是 Google 推出的终端 AI 工具，让开发者可以直接在命令行环境中使用 Gemini 模型能力。

![](https://relay-1.bijitongbu.site/p/26a7152cc17eac70c8b03835f134b352.png)

**项目地址：**

```
https://github.com/google-gemini/gemini-cli
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 106.7K

**它提供什么：**

-   • AI 命令行交互
    
-   • 代码分析
    
-   • 文件处理
    
-   • 开发辅助
    
-   • 自动化任务执行
    

**典型应用：**

-   • 程序代码分析
    
-   • Linux 开发辅助
    
-   • 自动化脚本生成
    
-   • AI 编程助手
    

### 13\. langextract

### —— AI 驱动的信息抽取工具

langextract 是 Google 开源的文本结构化提取工具。

它利用大语言模型能力，将非结构化文本转换为结构化数据。

![](https://relay-1.bijitongbu.site/p/fefa55bf6bc390814a208ed3d1285bba.png)

**项目地址：**

```
https://github.com/google/langextract
```

> \*\*开源协议：\*\*Apache License 2.0

> **GitHub Star：** 35.8K

**它提供什么：**

-   • 文档信息提取
    
-   • 文本结构化
    
-   • LLM 提示工程支持
    
-   • 数据转换能力
    

**典型应用：**

-   • 企业文档分析
    
-   • 知识库构建
    
-   • 数据标注
    
-   • 信息检索
    

### 14\. Temporian

### —— Google 时间序列机器学习框架

Temporian 是 Google 推出的时间序列数据处理框架。

它专注于处理工业、传感器和业务系统中的连续变化数据。

![](https://relay-1.bijitongbu.site/p/990a8810600cbbe48874642991e372cf.png)

**项目地址：**

```
https://github.com/google/temporian
```

> **开源协议：**

Apache License 2.0

> **GitHub Star：** 713

**它提供什么：**

-   • 时间序列处理
    
-   • 特征工程
    
-   • 数据分析流水线
    
-   • 机器学习数据准备
    

**典型应用：**

-   • 工业预测维护
    
-   • IoT 数据分析
    
-   • 金融时间序列
    
-   • 设备监控
    

## 二、前端与 UI 开发（11）

Google 在前端领域长期推动现代 Web 技术发展。

从大型 Web 应用框架，到跨平台 UI 技术，再到设计体系，Google 构建了一套完整前端生态。

### 15\. Angular

### —— Google 主导维护的企业级前端框架

Angular 是 Google 开发维护的大型 Web 前端框架。

它采用 TypeScript 技术体系，适合构建复杂企业级应用。

![](https://relay-1.bijitongbu.site/p/ad3ad88f4ba4297c409e9c8a8e1f6683.png)

**项目地址：**

```
https://github.com/angular/angular
```

> **开源协议：** MIT License

> **GitHub Star：** 101K

**它提供什么：**

-   • 组件化开发
    
-   • TypeScript 支持
    
-   • 路由系统
    
-   • 状态管理方案
    
-   • 企业级工程能力
    

**典型应用：**

-   • 企业后台系统
    
-   • SaaS 平台
    
-   • 大型 Web 应用
    
-   • 管理系统
    

### 16\. Flutter

### —— Google 跨平台 UI 开发框架

Flutter 是 Google 推出的跨平台应用开发框架。

它允许开发者使用一套代码构建多个平台应用。

![](https://relay-1.bijitongbu.site/p/7721a6f47bc80b62615d4bd4614488e5.png)

**项目地址：**

```
https://github.com/flutter/flutter
```

> **开源协议：** BSD 3-Clause License

> **GitHub Star：** 178.6K

**它提供什么：**

-   • 跨平台 UI 框架
    
-   • Dart 语言支持
    
-   • 高性能渲染引擎
    
-   • 丰富组件库
    
-   • 移动端开发工具链
    

**典型应用：**

-   • Android 应用
    
-   • iOS 应用
    
-   • Web 应用
    
-   • 桌面软件
    
-   • 企业移动应用
    

### 17\. Material Design Icons

### —— Google Material Design 图标库

Material Design Icons 是 Google Material Design 体系中的图标资源集合。

![](https://relay-1.bijitongbu.site/p/6887f79f45c91f1bf7cd38448a195e9b.png)

**项目地址：**

```
https://github.com/google/material-design-icons
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 53.8K

**它提供什么：**

-   • UI 图标资源
    
-   • 多尺寸图标
    
-   • Web/移动端使用规范
    

**典型应用：**

-   • Android 应用
    
-   • Web UI
    
-   • 移动设计系统
    

### 18\. Material Design Lite（MDL）

### —— 轻量级 Material Design Web 框架

MDL 是 Google 推出的轻量 Web UI 框架，用于帮助开发者快速构建符合 Material Design 风格的网站。

![](https://relay-1.bijitongbu.site/p/f7463cab4568c23b5d0451c7cc468679.png)

**项目地址：**

```
https://github.com/google/material-design-lite
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 32.3K

**它提供什么：**

-   • CSS 组件
    
-   • UI 控件
    
-   • 响应式布局
    
-   • Material 风格设计
    

**典型应用：**

-   • 企业网站
    
-   • Web 应用界面
    
-   • 快速原型开发
    

### 19\. Closure Library

### —— Google 开源的大型 JavaScript 工具库

Closure Library 是 Google 开发的一套成熟 JavaScript 库，最初用于支撑 Google 内部大型 Web 应用开发。

它提供了大量经过长期实践验证的前端基础组件。

![](https://relay-1.bijitongbu.site/p/7aaf283532cace0f6def5ac368983dc6.png)

**项目地址：**

```
https://github.com/google/closure-library
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 4.9K

**它提供什么：**

-   • JavaScript 基础工具库
    
-   • DOM 操作封装
    
-   • 事件处理机制
    
-   • UI 组件
    
-   • 模块化开发支持
    
-   • 编译优化工具
    

**典型应用：**

-   • 大型 Web 应用
    
-   • 企业级前端系统
    
-   • Google 内部 Web 项目
    
-   • 复杂 JavaScript 工程
    

### 20\. Blockly

### —— Google 开源的可视化编程框架

Blockly 是 Google 推出的图形化编程工具，通过拖拽代码块的方式，让用户无需编写传统代码即可创建程序逻辑。

![](https://relay-1.bijitongbu.site/p/74ca169d5382573a87d968dff413a729.png)

**项目地址：**

```
https://github.com/google/blockly
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 13.5K

**它提供什么：**

-   • 图形化代码编辑器
    
-   • 自定义代码块
    
-   • 多语言代码生成
    
-   • Web 嵌入能力
    
-   • 编程教学工具
    

**典型应用：**

-   • 少儿编程教育
    
-   • STEM 教学平台
    
-   • 机器人编程
    
-   • 可视化开发工具
    

### 21\. Tracing Framework

### —— Google Web 性能分析框架

Tracing Framework 是 Google 开源的 Web 性能分析工具，用于帮助开发者观察复杂应用运行过程。

![](https://relay-1.bijitongbu.site/p/547de45668943994e585c200a110454b.png)

**项目地址：**

```
https://github.com/google/tracing-framework
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 2.6K

**它提供什么：**

-   • 性能事件追踪
    
-   • 时间线分析
    
-   • 数据可视化
    
-   • 浏览器性能调试
    

**典型应用：**

-   • Web 性能优化
    
-   • 浏览器调试
    
-   • 游戏性能分析
    
-   • 大型 Web 应用优化
    

### 22\. Dart

### —— Flutter 生态核心编程语言

Dart 是 Google 开发的现代编程语言，也是 Flutter 官方开发语言。

它结合了：

-   • 面向对象设计；
    
-   • 高性能运行环境；
    
-   • 跨平台开发能力。
    
    ![](https://relay-1.bijitongbu.site/p/3eabacbb5d0877f0a980a4f434b54b69.png)
    

**项目地址：**

```
https://github.com/dart-lang/sdk
```

> **开源协议：** BSD 3-Clause License

> **GitHub Star：** 11.3K

**它提供什么：**

-   • 编译器
    
-   • 虚拟机
    
-   • 标准库
    
-   • 异步编程模型
    
-   • Flutter 开发支持
    

**典型应用：**

-   • Flutter 移动应用
    
-   • Web 应用
    
-   • 桌面应用
    
-   • 跨平台软件开发
    

### 23\. GXUI

### —— Google X 前端 UI 组件项目

GXUI 是 Google 相关开源 UI 组件项目，用于探索现代 Web 界面设计和交互方式。

![](https://relay-1.bijitongbu.site/p/73364e73cc127fbedc3fce38e5153c9f.png)

**项目地址：**

```
https://github.com/google/gxui
```

> **开源协议：** MIT License

> **GitHub Star：** 4.4K

**它提供什么：**

-   • UI 控件
    
-   • 图形界面组件
    
-   • 交互设计工具
    
-   • 应用界面开发基础
    

**典型应用：**

-   • 桌面 UI
    
-   • 工具软件界面
    
-   • 图形化应用开发
    

### 24\. Google Fonts

### —— 全球最大的开源字体服务平台之一

Google Fonts 是 Google 提供的免费字体服务，为开发者提供大量开源字体资源。

![](https://relay-1.bijitongbu.site/p/08d10c5546a05910e95a89c4b80833fb.png)

**项目地址：**

```
https://github.com/google/fonts
```

> **开源协议：** Apache License / OFL 等多种协议

> **GitHub Star：** 20.4K

**它提供什么：**

-   • 开源字体资源
    
-   • 字体文件
    
-   • 字体测试工具
    
-   • Web 字体服务
    

**典型应用：**

-   • 网站设计
    
-   • 移动应用 UI
    
-   • 品牌视觉设计
    
-   • 开源项目字体支持
    

### 25\. AnyPixel.js

### —— Google 创意交互显示系统

AnyPixel.js 是 Google Creative Lab 开源的实验性项目，用于创建大型可交互显示屏。

它允许开发者将：

-   • LED；
    
-   • 按钮；
    
-   • 传感器；
    

组合成大型互动像素屏幕。

![](https://relay-1.bijitongbu.site/p/3196d5c1c524cec612c2beeaa89c65ee.png)

**项目地址：**

```
https://github.com/googlecreativelab/anypixel
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 6.4K

**它提供什么：**

-   • 交互式显示框架
    
-   • 硬件控制接口
    
-   • Web 控制系统
    
-   • 创意互动方案
    

**典型应用：**

-   • 艺术装置
    
-   • 展览互动屏
    
-   • 创意广告
    
-   • 数字艺术项目
    

## 三、云计算与基础设施（16）

Google 在云计算领域的影响力极其深远。

许多今天云原生时代的核心技术，都来源于 Google 的工程实践。

从容器管理，到服务通信，再到微服务治理，Google 开源项目已经成为现代互联网基础设施的重要组成部分。

### 26\. Kubernetes

### —— 云原生时代的容器编排标准

Kubernetes 是 Google 开源的容器编排平台，也是目前全球最主流的云原生基础设施。

它最初来源于 Google 内部 Borg 系统的设计思想。

![](https://relay-1.bijitongbu.site/p/b1daa64687a30192fcee14ee397d89cf.png)

**项目地址：**

```
https://github.com/kubernetes/kubernetes
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 125.2K

**它提供什么：**

-   • 容器自动部署
    
-   • 服务发现
    
-   • 自动扩缩容
    
-   • 负载均衡
    
-   • 故障恢复
    
-   • 集群资源管理
    

**典型应用：**

-   • 云计算平台
    
-   • 微服务架构
    
-   • DevOps 平台
    
-   • 企业级应用部署
    
-   • 大规模在线服务
    

### 27\. Protocol Buffers（protobuf）

### —— Google 高性能数据序列化方案

Protocol Buffers 是 Google 开发的数据交换格式，被广泛用于分布式系统通信。

相比 JSON：

protobuf 具有：

-   • 更小的数据体积；
    
-   • 更快解析速度；
    
-   • 更强类型约束。
    

![](https://relay-1.bijitongbu.site/p/6fbac935620e25648f047f599a43a022.png)

**项目地址：**

```
https://github.com/protocolbuffers/protobuf
```

> **开源协议：** BSD 3-Clause License

> **GitHub Star：** 71.8K

**它提供什么：**

-   • 数据序列化
    
-   • 多语言代码生成
    
-   • 高性能通信格式
    
-   • Schema 管理
    

**典型应用：**

-   • 微服务通信
    
-   • RPC 框架
    
-   • 分布式系统
    
-   • 数据存储
    

### 28\. gRPC

### —— Google 开源高性能 RPC 框架

gRPC 是 Google 推出的现代远程过程调用框架。

它基于：

-   • HTTP/2
    
-   • Protocol Buffers
    

实现高性能服务通信。

![](https://relay-1.bijitongbu.site/p/0611984a9d1f36df27f64624053b8d70.png)

**项目地址：**

```
https://github.com/grpc/grpc
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 45.3K

**它提供什么：**

-   • 高性能 RPC 调用
    
-   • HTTP/2 通信
    
-   • 多语言支持
    
-   • 双向流通信
    
-   • 服务定义工具
    

**典型应用：**

-   • 微服务架构
    
-   • 云原生系统
    
-   • 分布式后端
    
-   • 服务间通信
    

### 29\. Bazel

### —— Google 大规模构建系统

Bazel 是 Google 开源的自动化构建工具，用于管理大型软件项目。

它解决的问题：

> 当代码规模达到百万甚至千万级时，如何快速可靠地完成编译。

![](https://relay-1.bijitongbu.site/p/af1475195dd47fc145a8eb06069e30cb.png)

**项目地址：**

```
https://github.com/bazelbuild/bazel
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 25.8K

**它提供什么：**

-   • 增量编译
    
-   • 多语言构建
    
-   • 分布式缓存
    
-   • 可复现构建
    

**典型应用：**

-   • 大型 C++ 项目
    
-   • Android 构建
    
-   • 云服务系统
    
-   • 企业级软件工程
    

### 30\. Istio

### —— 云原生服务网格平台

Istio 是 Google、IBM、Lyft 等共同推动的服务网格项目。

它用于解决微服务环境中的：

-   • 服务通信；
    
-   • 流量控制；
    
-   • 安全认证；
    
-   • 可观测性。
    

![](https://relay-1.bijitongbu.site/p/0256ef7227e1a824f658edc3b13bef84.png)

**项目地址：**

```
https://github.com/istio/istio
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 38.4K

**它提供什么：**

-   • 服务发现
    
-   • 流量管理
    
-   • 熔断机制
    
-   • 服务安全
    
-   • 监控指标
    

**典型应用：**

-   • 微服务平台
    
-   • Kubernetes 集群
    
-   • 企业云架构
    
-   • 大规模服务治理
    

### 31\. Knative

### —— Google 推动的 Kubernetes Serverless 平台

Knative 是 Google 主导开发的云原生 Serverless 平台，构建于 Kubernetes 之上。

它的目标是：

> 让开发者无需关注服务器管理，只需要关注代码和业务逻辑。

![](https://relay-1.bijitongbu.site/p/30c3a16d1a7369bb01b4638c48e8d3d2.png)

**项目地址：**

```
https://github.com/knative/serving
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 6.1K

**它提供什么：**

-   • Serverless 服务部署
    
-   • 自动扩缩容
    
-   • 请求驱动实例管理
    
-   • 容器生命周期管理
    
-   • Kubernetes 集成能力
    

**典型应用：**

-   • Serverless 应用平台
    
-   • 云函数服务
    
-   • 微服务快速部署
    
-   • AI 推理服务部署
    

### 32\. cAdvisor

### —— Google 开源的容器资源监控工具

cAdvisor（Container Advisor）是 Google 开发的容器监控工具，用于分析运行中容器的资源使用情况。

它能够自动收集：

-   • CPU 使用率；
    
-   • 内存占用；
    
-   • 网络流量；
    
-   • 文件系统数据。
    

![](https://relay-1.bijitongbu.site/p/9cdfb2e2bc90ee90d2ac314e8576c7ac.png)

**项目地址：**

```
https://github.com/google/cadvisor
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 19.4K

**它提供什么：**

-   • 容器性能监控
    
-   • 资源统计
    
-   • Docker 容器分析
    
-   • Prometheus 指标输出
    

**典型应用：**

-   • Kubernetes 监控
    
-   • 云平台运维
    
-   • 容器性能分析
    
-   • DevOps 平台
    

### 33\. gVisor

### —— Google 容器安全隔离运行时

gVisor 是 Google 开发的容器安全运行环境。

![](https://relay-1.bijitongbu.site/p/e0bacc70f126a05535b677de5a5acf3d.png)

传统容器：

```
应用程序

↓

Docker

↓

Linux Kernel
```

容器直接共享宿主机内核。

gVisor 增加了一层：

```
应用程序

↓

gVisor Sandbox

↓

Linux Kernel
```

通过用户态内核实现更强隔离。

**项目地址：**

```
https://github.com/google/gvisor
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 19.2K

**它提供什么：**

-   • 容器隔离
    
-   • 用户态内核
    
-   • 系统调用拦截
    
-   • 沙箱运行环境
    

**典型应用：**

-   • 云计算安全
    
-   • 多租户平台
    
-   • 不可信代码运行
    
-   • Serverless 平台
    

### 34\. Skaffold

### —— Kubernetes 应用开发工具

Skaffold 是 Google 开发的 Kubernetes 开发工作流工具。

![](https://relay-1.bijitongbu.site/p/edf68a6179d4ab78a8f0a991d05b6e39.png)

它主要解决：

> 开发者如何快速构建、部署和调试 Kubernetes 应用。

**项目地址：**

```
https://github.com/GoogleContainerTools/skaffold
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 15.9K

**它提供什么：**

-   • 自动构建镜像
    
-   • 自动部署 Kubernetes
    
-   • 开发环境热更新
    
-   • CI/CD 集成
    

**典型应用：**

-   • Kubernetes 开发
    
-   • 云原生项目调试
    
-   • DevOps 流程自动化
    

### 35\. Kaniko

### —— Google 开源的容器镜像构建工具

Kaniko 用于在没有 Docker Daemon 的环境中构建容器镜像。

![](https://relay-1.bijitongbu.site/p/64ddea0dcb2e5a2a20267cdcd58c861b.png)

传统方式：

```
Docker CLI

↓

Docker Daemon

↓

镜像
```

Kaniko：

```
Dockerfile

↓

Kaniko Executor

↓

镜像
```

更加适合云环境。

**项目地址：**

```
https://github.com/GoogleContainerTools/kaniko
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 15.8K

**它提供什么：**

-   • 无 Docker 环境镜像构建
    
-   • Kubernetes 集成
    
-   • CI/CD 镜像生成
    
-   • 安全构建流程
    

**典型应用：**

-   • 云原生流水线
    
-   • Kubernetes 构建系统
    
-   • 持续集成平台
    

### 36\. Seesaw

### —— Google 开源负载均衡平台

Seesaw 是 Google 开源的高性能负载均衡解决方案。

![](https://relay-1.bijitongbu.site/p/c7ee669b54bdfa053b5300744b915027.png)

它主要用于：

-   • 数据中心流量管理；
    
-   • 服务入口控制；
    
-   • 高可用网络架构。
    

**项目地址：**

```
https://github.com/google/seesaw
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 5.7K

**它提供什么：**

-   • 四层负载均衡
    
-   • 高可用服务入口
    
-   • 网络流量分配
    
-   • 集群管理
    

**典型应用：**

-   • 数据中心网络
    
-   • 企业内部服务
    
-   • 高可用系统
    

### 37\. Agones

### —— Google 开源游戏服务器管理平台

Agones 是 Google 与 Ubisoft 合作推出的 Kubernetes 游戏服务器管理系统。

![](https://relay-1.bijitongbu.site/p/22cd64b9c68194197b73644164b517a3.png)

它解决：

> 大规模多人在线游戏服务器如何自动部署和管理。

**项目地址：**

```
https://github.com/googleforgames/agones
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 7K

**它提供什么：**

-   • 游戏服务器生命周期管理
    
-   • 自动扩缩容
    
-   • Kubernetes 集成
    
-   • 游戏实例调度
    

**典型应用：**

-   • MMO 游戏
    
-   • 云游戏
    
-   • 在线竞技游戏
    
-   • 游戏后端基础设施
    

### 38\. Microservices Demo

### —— Google 云原生微服务示例项目

Microservices Demo 是 Google Cloud 提供的开源微服务示例应用。

![](https://relay-1.bijitongbu.site/p/5e40fa89375fd63773480a7f67f2aaa4.png)

它展示如何：

-   • 构建微服务；
    
-   • 部署 Kubernetes；
    
-   • 使用云原生工具。
    

**项目地址：**

```
https://github.com/GoogleCloudPlatform/microservices-demo
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 20.9K

**它提供什么：**

-   • 微服务架构示例
    
-   • Kubernetes 部署配置
    
-   • 服务通信案例
    
-   • 云平台实践方案
    

**典型应用：**

-   • 云原生学习
    
-   • Kubernetes 入门
    
-   • 微服务架构参考
    

### 39\. Open Health Stack

### —— Google 医疗健康开源技术体系

Open Health Stack 是 Google 推动的医疗数据和数字健康开源项目集合。

![](https://relay-1.bijitongbu.site/p/929510366d089aa1dec9485826b72e63.png)

帮助开发者构建符合标准的医疗应用。

**项目地址：**

```
https://developers.google.com/
```

**开源协议：**

Apache License 2.0

**它提供什么：**

-   • 医疗数据标准支持
    
-   • FHIR 数据处理
    
-   • 健康应用开发工具
    
-   • 医疗 AI 基础能力
    

**典型应用：**

-   • 数字医疗平台
    
-   • 医疗数据管理
    
-   • 健康应用开发
    

### 40\. Elafros

### —— Google 云原生基础设施项目

Elafros 是 Google 开源的基础设施相关项目，主要用于探索现代云环境中的服务管理和自动化能力。

![](https://relay-1.bijitongbu.site/p/91fe3f61e61d2f5dbe585bac50b90886.png)

**项目地址：**

```
https://github.com/google/elafros
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 6.1K

**它提供什么：**

-   • 云基础设施实验能力
    
-   • 服务管理组件
    
-   • 自动化运维工具
    

**典型应用：**

-   • 云平台研究
    
-   • 基础设施实验
    
-   • 分布式系统探索
    

### 41\. SLO Generator

### —— Google 开源服务可靠性工具

SLO Generator 是 Google 开源的服务可靠性工程（SRE）工具。

![](https://relay-1.bijitongbu.site/p/da4c6619209f3f73199fc3476e650313.png)

它帮助团队自动生成和管理：

-   • SLO；
    
-   • SLA；
    
-   • 服务可靠性指标。
    

**项目地址：**

```
https://github.com/google/slo-generator
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 564

**它提供什么：**

-   • SLO 配置生成
    
-   • 可靠性指标管理
    
-   • 云监控集成
    
-   • SRE 工作流支持
    

**典型应用：**

-   • 云服务运维
    
-   • 大规模系统可靠性管理
    
-   • DevOps 平台
    

## 四、数据库与存储（8）

数据库是软件系统的核心基础设施。

Google 在数据存储领域贡献了大量优秀项目，从嵌入式数据库，到高性能数据结构，再到分布式数据库方案。

### 42\. LevelDB

### —— Google 开源高性能键值数据库

LevelDB 是 Google 开发的轻量级嵌入式 Key-Value 数据库。

它采用 LSM Tree 存储结构，在写入性能方面表现优秀。

![](https://relay-1.bijitongbu.site/p/1a70c28c1833be134a5957bab77cb169.png)

**项目地址：**

```
https://github.com/google/leveldb
```

**开源协议：** BSD 3-Clause License

**GitHub Star：** 39.4K

**它提供什么：**

-   • Key-Value 存储
    
-   • LSM Tree 架构
    
-   • 数据压缩
    
-   • 高性能写入
    
-   • 本地持久化
    

**典型应用：**

-   • 区块链数据库
    
-   • 浏览器存储
    
-   • 嵌入式系统
    
-   • 本地缓存
    

### 43\. Guava

### —— Google 开源的 Java 核心工具库

Guava 是 Google 开发的一套 Java 通用工具库。

它补充了 Java 标准库中缺少的很多高级功能，被大量企业级 Java 项目广泛使用。

![](https://relay-1.bijitongbu.site/p/113a6a2c82ddb9a8c6eacb86445925d3.png)

**项目地址：**

```
https://github.com/google/guava
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 51.9K

**它提供什么：**

-   • 集合增强工具
    
-   • 缓存机制
    
-   • 字符串处理
    
-   • 并发工具
    
-   • 函数式编程支持
    
-   • I/O 工具
    

**典型应用：**

-   • Java 后端开发
    
-   • 企业级应用
    
-   • 分布式系统
    
-   • 大规模服务端项目
    

### 44\. Gson

### —— Google 开源 Java JSON 序列化库

Gson 是 Google 开发的 Java JSON 处理框架。

它可以帮助开发者快速完成：

对象 ⇄ JSON

之间的数据转换。

![](https://relay-1.bijitongbu.site/p/7e50edcc5b841ab9607e7fa51f85bef9.png)

**项目地址：**

```
https://github.com/google/gson
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 24.5K

**它提供什么：**

-   • JSON 序列化
    
-   • JSON 反序列化
    
-   • 对象映射
    
-   • 自定义转换器
    
-   • 泛型支持
    

**典型应用：**

-   • Android 开发
    
-   • Java Web 服务
    
-   • API 数据交换
    
-   • 配置文件解析
    

### 45\. FlatBuffers

### —— Google 高性能数据序列化框架

FlatBuffers 是 Google 开源的数据序列化工具。

它主要用于解决：

> 如何在性能敏感场景下快速访问结构化数据。

相比传统 JSON：

FlatBuffers 不需要完整解析即可访问数据。

![](https://relay-1.bijitongbu.site/p/9872e3b9c70e1e7e7a24fdbba71c756a.png)

**项目地址：**

```
https://github.com/google/flatbuffers
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 26.4K

**它提供什么：**

-   • 零拷贝数据访问
    
-   • 高性能序列化
    
-   • 多语言支持
    
-   • Schema 定义
    

**典型应用：**

-   • 游戏开发
    
-   • 移动应用
    
-   • 高性能通信
    
-   • 嵌入式系统
    

### 46\. Brotli

### —— Google 开源压缩算法

Brotli 是 Google 开发的一种现代数据压缩算法。

它主要用于替代传统 gzip，在 Web 场景中提供更高压缩率。

![](https://relay-1.bijitongbu.site/p/ccea68be7892bdcb9a868a359c4553bc.png)

**项目地址：**

```
https://github.com/google/brotli
```

> **开源协议：** MIT License

> **GitHub Star：** 14.9K

**它提供什么：**

-   • 高压缩率算法
    
-   • 快速解压
    
-   • 多语言支持
    
-   • Web 内容压缩
    

**典型应用：**

-   • 浏览器网络传输
    
-   • CDN 数据压缩
    
-   • Web 静态资源优化
    
-   • 软件分发
    

### 47\. Guetzli

### —— Google 图像压缩优化工具

Guetzli 是 Google 开源的 JPEG 图像压缩算法。

它目标是在保持视觉质量的同时降低图片大小。

![](https://relay-1.bijitongbu.site/p/f5fea1f2d57419537fe7dae0eebef818.png)

**项目地址：**

```
https://github.com/google/guetzli
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 12.9K

**它提供什么：**

-   • JPEG 压缩优化
    
-   • 感知质量优化
    
-   • 图片体积降低
    

**典型应用：**

-   • 网站图片优化
    
-   • 图片存储系统
    
-   • 内容分发平台
    
-   • 移动应用资源优化
    

### 48\. Lovefield

### —— Google 开源浏览器端关系数据库

Lovefield 是 Google 开发的 Web 端关系型数据库框架。

它让浏览器应用能够使用类似 SQL 数据库的方式管理本地数据。

![](https://relay-1.bijitongbu.site/p/c4fc12e44029372b2c6810ded1ce1d6b.png)

**项目地址：**

```
https://github.com/google/lovefield
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 6.8K

**它提供什么：**

-   • 浏览器端数据库
    
-   • SQL 风格查询
    
-   • 数据事务
    
-   • 索引管理
    

**典型应用：**

-   • Web 离线应用
    
-   • 浏览器本地存储
    
-   • 前端复杂数据管理
    

### 49\. Vitess

### —— Google 开源的大规模 MySQL 分布式数据库系统

Vitess 最初来源于 YouTube 数据库架构。

它解决 MySQL 在大规模互联网业务中的：

-   • 扩展；
    
-   • 分片；
    
-   • 高可用；
    

问题。

![](https://relay-1.bijitongbu.site/p/9f7a7546e7650c0eed100620e5bf56ec.png)

**项目地址：**

```
https://github.com/vitessio/vitess
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 21.2K

**它提供什么：**

-   • MySQL 分片
    
-   • 数据库代理
    
-   • 自动扩展
    
-   • 集群管理
    
-   • 高可用能力
    

**典型应用：**

-   • 大型互联网服务
    
-   • 云数据库
    
-   • 高并发业务
    
-   • 分布式存储系统
    

## 五、开发工具与测试（12）

优秀的软件工程离不开完善的工具链。

Google 在开发工具领域开源了大量项目，包括：

-   • 编程语言；
    
-   • 测试框架；
    
-   • 代码分析；
    
-   • 安全检测；
    
-   • 构建工具。
    

### 50\. Go

### —— Google 开源的现代系统编程语言

Go（Golang）是 Google 开发的一门编程语言。

![](https://relay-1.bijitongbu.site/p/539b0a54c3bb2cb91483df481de58f07.png)

它主要用于解决：

> 大规模服务器软件开发中的效率、性能和并发问题。

**项目地址：**

```
https://github.com/golang/go
```

> **开源协议：** BSD License

> **GitHub Star：** 136.4K

**它提供什么：**

-   • 简洁语法
    
-   • 高性能编译
    
-   • Goroutine 并发模型
    
-   • 网络编程支持
    
-   • 垃圾回收机制
    

**典型应用：**

-   • 云计算平台
    
-   • 微服务
    
-   • 容器系统
    
-   • 网络服务
    

代表项目：

-   • Kubernetes
    
-   • Docker
    
-   • Prometheus
    

### 51\. GoogleTest

### —— Google 开源 C++ 单元测试框架

GoogleTest 是目前 C++ 生态中最流行的测试框架之一。

它帮助开发者：

快速编写、

运行、

维护测试代码。

![](https://relay-1.bijitongbu.site/p/0ca6c7cb08fdbb0d1078f1eccbc3f5a5.png)

**项目地址：**

```
https://github.com/google/googletest
```

> **开源协议：** BSD 3-Clause License

> **GitHub Star：** 39K

**它提供什么：**

-   • 单元测试框架
    
-   • 测试断言
    
-   • Mock 支持
    
-   • 测试自动化
    

**典型应用：**

-   • C++ 项目测试
    
-   • Linux 系统开发
    
-   • 游戏引擎
    
-   • 工业软件
    

### 52\. Python Fire

### —— Google Python 命令行生成工具

Python Fire 是 Google 开源的 Python 工具。

它可以自动将 Python 对象转换为命令行接口。

![](https://relay-1.bijitongbu.site/p/7704dd965c1e2041703280100d6875b2.png)

**项目地址：**

```
https://github.com/google/python-fire
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 28.2K

**它提供什么：**

-   • 自动 CLI 生成
    
-   • Python 对象调用
    
-   • 快速工具开发
    

**典型应用：**

-   • 数据分析工具
    
-   • AI 实验脚本
    
-   • 自动化工具
    

### 53\. YAPF

### —— Google Python 代码格式化工具

YAPF 是 Google 开源的 Python 自动格式化工具。

它根据 Python 风格规范自动调整代码格式。

![](https://relay-1.bijitongbu.site/p/5b065b781fe08dd119de9d0c2ce768d3.png)

**项目地址：**

```
https://github.com/google/yapf
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 14K

**它提供什么：**

-   • Python 自动格式化
    
-   • 风格统一
    
-   • IDE 集成
    

**典型应用：**

-   • Python 项目规范化
    
-   • 团队代码管理
    
-   • CI 自动检查
    

### 54\. Error-Prone

### —— Google Java 静态代码检查工具

Error-Prone 是 Google 开源的 Java 编译期错误检测工具。

它可以发现：

很多普通编译器无法发现的问题。

![](https://relay-1.bijitongbu.site/p/39b90b23b709bc8bfb68083a739b1202.png)

**项目地址：**

```
https://github.com/google/error-prone
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 7.2K

**它提供什么：**

-   • Java 静态分析
    
-   • 编译检查
    
-   • Bug 模式检测
    
-   • 代码质量提升
    

**典型应用：**

-   • 大型 Java 项目
    
-   • 企业代码审查
    
-   • 自动化质量控制
    

### 55\. Auto

### —— Google Java 自动代码生成工具

Auto 是 Google 开源的 Java 自动代码生成框架。

它主要用于减少 Java 项目中的重复代码，让开发者通过简单注解生成标准代码。

![](https://relay-1.bijitongbu.site/p/ac887b3a5562468c2ed1be00b000c7a5.png)

**项目地址：**

```
https://github.com/google/auto
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 10.6K

**它提供什么：**

-   • Java 注解处理器
    
-   • 自动代码生成
    
-   • 编译期代码检查
    
-   • 类型安全代码生成
    

**典型应用：**

-   • Java 企业项目
    
-   • Android 开发
    
-   • 框架代码生成
    
-   • 库开发
    

### 56\. ZXing

### —— Google 开源二维码与条码识别库

ZXing（"Zebra Crossing"）是 Google 开源的二维码和条码处理库。

它是移动端二维码识别领域非常经典的开源项目。

![](https://relay-1.bijitongbu.site/p/a2226034d3916714f5bdb0815756958f.png)

**项目地址：**

```
https://github.com/zxing/zxing
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 34.1K

**它提供什么：**

-   • 二维码生成
    
-   • 二维码扫描
    
-   • 条码识别
    
-   • 多种编码格式支持
    
-   • 图像解析能力
    

支持：

-   • QR Code
    
-   • Data Matrix
    
-   • UPC
    
-   • EAN
    
-   • Code 128 等。
    

**典型应用：**

-   • 手机扫码功能
    
-   • 支付二维码
    
-   • 商品条码系统
    
-   • 身份认证
    

### 57\. Compile Testing

### —— Google Java 编译测试框架

Compile Testing 是 Google 开源的 Java 编译测试工具。

![](https://relay-1.bijitongbu.site/p/2c8ffef655ae089b4be93b8c4e49c0c3.png)

它主要用于测试：

> 注解处理器和代码生成工具是否正确工作。

**项目地址：**

```
https://github.com/google/compile-testing
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 722

**它提供什么：**

-   • Java 编译测试
    
-   • 注解处理器测试
    
-   • 编译错误验证
    
-   • 自动化测试工具
    

**典型应用：**

-   • Java 框架开发
    
-   • Android 工具开发
    
-   • 编译插件测试
    

### 58\. GRR

### —— Google 开源远程取证响应框架

GRR（Google Rapid Response）是 Google 开发的远程安全取证平台。

它主要用于：

大规模终端设备调查和安全事件响应。

![](https://relay-1.bijitongbu.site/p/1a286855c39e9979ec71f41a353ec329.png)

**项目地址：**

```
https://github.com/google/grr
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 5.1K

**它提供什么：**

-   • 远程终端分析
    
-   • 数字取证
    
-   • 文件搜索
    
-   • 系统状态采集
    
-   • 安全事件调查
    

**典型应用：**

-   • 企业安全运营
    
-   • 网络攻击调查
    
-   • 主机取证
    
-   • 安全响应中心
    

### 59\. Tsunami

### —— Google 网络安全漏洞扫描框架

Tsunami 是 Google 开源的大规模网络安全扫描系统。

它用于自动发现企业网络中的安全风险。

![](https://relay-1.bijitongbu.site/p/cab575968d311127b2785753b4e40ffe.png)

**项目地址：**

```
https://github.com/google/tsunami-security-scanner
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 8.6K

**它提供什么：**

-   • 网络漏洞扫描
    
-   • 插件化检测机制
    
-   • 安全风险识别
    
-   • 自动化安全测试
    

**典型应用：**

-   • 企业安全检测
    
-   • 云环境安全扫描
    
-   • 渗透测试辅助
    

### 60\. Sanitizers

### —— Google 开源程序运行时检测工具集

Sanitizers 是 Google 开发的一系列程序错误检测工具。

它们主要用于发现：

-   • 内存错误；
    
-   • 未定义行为；
    
-   • 线程问题。
    

![](https://relay-1.bijitongbu.site/p/ca2aee564df98cedc1417af3bba5ec24.png)

**项目地址：**

```
https://github.com/google/sanitizers
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 12.5K

**它提供什么：**

主要包括：

-   • AddressSanitizer（ASan）
    
-   • MemorySanitizer（MSan）
    
-   • ThreadSanitizer（TSan）
    
-   • UndefinedBehaviorSanitizer（UBSan）
    

**典型应用：**

-   • C/C++ 开发
    
-   • Linux 系统开发
    
-   • 浏览器开发
    
-   • 高可靠软件测试
    

### 61\. OSS-Fuzz

### —— Google 开源持续模糊测试平台

OSS-Fuzz 是 Google 推动的开源软件安全测试平台。

它通过持续运行模糊测试（Fuzzing）发现开源项目中的隐藏漏洞。

![](https://relay-1.bijitongbu.site/p/8a799ec7daa297d3c09e179720258239.png)

**项目地址：**

```
https://github.com/google/oss-fuzz
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 12.6K

**它提供什么：**

-   • 自动化 Fuzz 测试
    
-   • 崩溃检测
    
-   • 漏洞发现
    
-   • CI 集成
    

**典型应用：**

-   • 开源软件安全测试
    
-   • 编译器测试
    
-   • 浏览器安全
    
-   • 网络协议安全
    

## 六、移动与操作系统（7）

移动操作系统是 Google 开源生态的重要组成部分。

从 Android，到 Chromium，再到 Fuchsia，Google 构建了覆盖手机、浏览器、IoT 的完整系统生态。

### 62\. Android

### —— 全球最大的移动操作系统生态之一

Android 是 Google 主导开发的开源移动操作系统。

它基于 Linux Kernel 构建，形成了完整的软件平台。

![](https://relay-1.bijitongbu.site/p/93b4c56164dec318d91dbc5336f8cbc3.png)

**项目地址：**

```
https://android.googlesource.com/
```

> **开源协议：** Apache License 2.0  
> GPL License（Linux Kernel 部分）

**它提供什么：**

-   • 移动操作系统框架
    
-   • Linux 内核支持
    
-   • 应用运行环境
    
-   • 系统服务
    
-   • 硬件抽象层 HAL
    

**典型应用：**

-   • 智能手机
    
-   • 平板设备
    
-   • 智能电视
    
-   • 车载系统
    
-   • IoT 设备
    

### 63\. Chromium

### —— Google Chrome 浏览器开源核心

Chromium 是 Google Chrome 浏览器的开源基础。

大量现代浏览器都基于 Chromium 构建。

![](https://relay-1.bijitongbu.site/p/117857f53c7a9ed51daba49e84e69086.png)

**项目地址：**

```
https://chromium.googlesource.com/chromium/src/
```

> **开源协议：** BSD License

**它提供什么：**

-   • 浏览器渲染引擎
    
-   • JavaScript 引擎
    
-   • 网络协议支持
    
-   • 多进程架构
    
-   • 浏览器安全机制
    

**典型应用：**

-   • Google Chrome
    
-   • Microsoft Edge
    
-   • Opera
    
-   • 企业浏览器
    

### 64\. Fuchsia

### —— Google 新一代操作系统项目

Fuchsia 是 Google 开发的新型操作系统。

不同于 Android 基于 Linux Kernel，Fuchsia 使用自研 Zircon 微内核。

![](https://relay-1.bijitongbu.site/p/481b41ee908fc69b965ec06c1fa59ada.png)

**项目地址：**

```
https://fuchsia.googlesource.com/
```

> **开源协议：** BSD License / Apache License 2.0

**它提供什么：**

-   • 微内核架构
    
-   • 跨设备系统能力
    
-   • 安全隔离机制
    
-   • 模块化系统设计
    

**典型应用：**

-   • 智能设备
    
-   • IoT 系统
    
-   • 新型计算平台
    

### 65\. ExoPlayer

### —— Google 开源 Android 媒体播放器

ExoPlayer 是 Google 开发的 Android 高性能播放器框架。

它相比系统 MediaPlayer，提供更灵活的视频播放能力。

![](https://relay-1.bijitongbu.site/p/dcc058d43a6249b2709dc7f13198fd11.png)

**项目地址：**

```
https://github.com/google/ExoPlayer
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 21.9K

**它提供什么：**

-   • 视频播放
    
-   • 音频播放
    
-   • 自适应码率
    
-   • 流媒体支持
    
-   • DRM 支持
    

**典型应用：**

-   • 视频 APP
    
-   • 在线直播
    
-   • OTT 平台
    
-   • Android 媒体应用
    

### 66\. iOSched

### —— Google 移动调度研究项目

iOSched 是 Google 针对移动系统任务调度进行研究的开源项目。

![](https://relay-1.bijitongbu.site/p/75e52d8100b603f4253ffb703b0bcce1.png)

**项目地址：**

```
https://github.com/google/iosched
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 21.6K

**它提供什么：**

-   • Android 应用示例
    
-   • 移动开发最佳实践
    
-   • 架构设计参考
    

**典型应用：**

-   • Android 学习
    
-   • 移动应用架构研究
    
-   • 开发模板参考
    

### 67\. Filament

### —— Google 开源实时渲染引擎

Filament 是 Google 开发的跨平台实时 3D 渲染引擎。

它用于在移动设备和桌面环境中实现高质量图形渲染。

![](https://relay-1.bijitongbu.site/p/294c148285deb9d266b7bdade870cc16.png)

**项目地址：**

```
https://github.com/google/filament
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 20.4K

**它提供什么：**

-   • PBR 渲染
    
-   • 3D 图形处理
    
-   • GPU 加速
    
-   • 跨平台渲染
    

**典型应用：**

-   • Android 3D 应用
    
-   • 游戏开发
    
-   • AR/VR
    
-   • 数字孪生
    

### 68\. Hover

### —— Google 移动交互实验项目

Hover 是 Google 相关移动交互实验项目，用于探索新的 UI 交互方式。

![](https://relay-1.bijitongbu.site/p/1b43b20ad41b228b67e21e0d1063749d.png)

**项目地址：**

```
https://github.com/google/hover
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 2.6K

**它提供什么：**

-   • 移动 UI 实验
    
-   • 交互设计方案
    
-   • Android 开发示例
    

**典型应用：**

-   • 移动交互研究
    
-   • UI 原型设计
    
-   • Android 学习
    

## 七、安全与隐私（6）

随着软件系统规模不断扩大，安全已经成为现代软件工程的重要组成部分。

Google 在安全领域长期投入，并开源了大量工具，用于解决：

-   • 数据加密；
    
-   • 隐私保护；
    
-   • 漏洞发现；
    
-   • 软件供应链安全；
    

等问题。

### 69\. Tink

### —— Google 开源现代密码学工具库

Tink 是 Google 开发的一套跨语言密码学库。

它的目标是：

> 让开发者能够更安全、更简单地使用加密技术。

很多应用中的：

-   • 数据加密；
    
-   • 密钥管理；
    
-   • 数字签名；
    

都需要依赖可靠的密码学实现。

![](https://relay-1.bijitongbu.site/p/34e18ed3004150c4bf98d1c2ff718c74.png)

**项目地址：**

```
https://github.com/google/tink
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 13.5K

**它提供什么：**

-   • 对称加密
    
-   • 非对称加密
    
-   • 数字签名
    
-   • MAC 校验
    
-   • 密钥管理
    
-   • 多语言 API
    

支持：

-   • Java
    
-   • C++
    
-   • Go
    
-   • Python
    
-   • Objective-C
    

**典型应用：**

-   • 移动应用数据保护
    
-   • 云服务安全
    
-   • 用户隐私保护
    
-   • 企业级加密系统
    

### 70\. Differential Privacy

### —— Google 差分隐私技术框架

Differential Privacy（差分隐私）是 Google 推动的重要隐私保护技术。

![](https://relay-1.bijitongbu.site/p/c5a822cc5befc60863f6ac29165dd64b.png)

它解决的问题：

> 如何在利用数据价值的同时，保护个人隐私。

例如：

企业希望分析用户行为：

```
用户数据

↓

统计分析

↓

业务优化
```

但不能泄露单个用户信息。

差分隐私通过加入数学噪声，实现：

数据可用：

-   •
    

个人不可识别。

**项目地址：**

```
https://github.com/google/differential-privacy
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 3.3K

**它提供什么：**

-   • 差分隐私算法
    
-   • 数据分析工具
    
-   • 隐私预算管理
    
-   • 统计查询保护
    

**典型应用：**

-   • 用户数据分析
    
-   • 医疗数据研究
    
-   • 政府统计
    
-   • 大规模数据平台
    

### 71\. Open Location Code

### —— Google 开源地理位置编码系统

Open Location Code 又称 Plus Codes，是 Google 开发的位置编码方案。

它可以将：

经纬度坐标

转换为：

简短可读的位置编码。

例如：

```
经度/纬度

↓

7FG49Q00+
```

即使没有传统地址，也可以准确表示位置。

![](https://relay-1.bijitongbu.site/p/5fc4176dcfb44dc0a3d15abae0e0e2a6.png)

**项目地址：**

```
https://github.com/google/open-location-code
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 4.3K

**它提供什么：**

-   • 经纬度编码
    
-   • 地址替代方案
    
-   • 全球位置表示
    
-   • 多语言支持
    

**典型应用：**

-   • 地图服务
    
-   • 无地址地区定位
    
-   • 应急救援
    
-   • 物流配送
    

### 72\. OSV-Scanner

### —— Google 开源软件漏洞扫描工具

OSV-Scanner 是 Google 开发的软件供应链安全工具。

它基于 OSV（Open Source Vulnerabilities）漏洞数据库，对项目依赖进行安全检查。

现代软件大量依赖：

-   • 开源库；
    
-   • 第三方组件；
    
-   • 软件包。
    

因此依赖安全成为关键问题。

![](https://relay-1.bijitongbu.site/p/d5a33e7545b4c5e1674b01fd2d3b6e82.png)

**项目地址：**

```
https://github.com/google/osv-scanner
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 10.9K

**它提供什么：**

-   • 开源依赖漏洞扫描
    
-   • 软件包检测
    
-   • SBOM 分析
    
-   • CI/CD 集成
    

**典型应用：**

-   • 企业软件安全
    
-   • 开源项目审计
    
-   • DevSecOps 流程
    
-   • 软件供应链保护
    

### 73\. Scorecards

### —— Google 开源软件供应链安全检查工具

Scorecards 是 Google Open Source Security Team 推出的安全评估工具。

它用于评价一个开源项目的安全成熟度。

例如：

-   • 是否开启代码审查；
    
-   • 是否启用安全扫描；
    
-   • 是否保护发布流程。
    

![](https://relay-1.bijitongbu.site/p/8d8a8d68df5606086e44185a2e61eaca.png)

**项目地址：**

```
https://github.com/ossf/scorecard
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 5.7K

**它提供什么：**

-   • 开源项目安全评分
    
-   • 自动化安全检查
    
-   • GitHub Actions 集成
    
-   • 软件供应链评估
    

**典型应用：**

-   • 开源项目管理
    
-   • 企业依赖评估
    
-   • 软件供应链安全
    

### 74\. Wycheproof

### —— Google 密码学算法测试项目

Wycheproof 是 Google 开源的密码学测试工具。

它主要用于发现密码算法实现中的安全漏洞。

密码算法本身可能正确：

但是：

实现错误

↓

可能导致安全问题。

![](https://relay-1.bijitongbu.site/p/f1fcca37ef009827d3ca92989b81315d.png)

**项目地址：**

```
https://github.com/google/wycheproof
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 3.1K

**它提供什么：**

-   • 加密算法测试案例
    
-   • 安全漏洞检测
    
-   • 密码库验证
    

支持：

-   • RSA
    
-   • AES
    
-   • ECDSA
    
-   • DH 等。
    

**典型应用：**

-   • 密码库开发
    
-   • 安全软件测试
    
-   • 金融系统安全验证
    

## 八、其他重要项目（6）

除了核心技术领域，Google 还开源了大量影响深远的基础项目。

这些项目覆盖：

-   • 代码审查；
    
-   • 大数据处理；
    
-   • 移动测试；
    
-   • Web 性能分析。
    

### 75\. Gerrit

### —— Google 开源代码评审系统

Gerrit 是 Google 开发的 Git 代码审查平台。

它改变了传统 Git 协作模式，将：

代码提交

↓

代码审核

↓

合并发布

形成完整流程。

![](https://relay-1.bijitongbu.site/p/54ad3dc61f90a1824c0b741720eb1860.png)

**项目地址：**

```
https://github.com/GerritCodeReview/gerrit
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 1.2K

**它提供什么：**

-   • Git 代码审核
    
-   • 权限管理
    
-   • Patch Review
    
-   • CI 集成
    

**典型应用：**

-   • 大型软件团队
    
-   • Linux 开发流程
    
-   • 企业代码管理
    
-   • 开源项目协作
    

### 76\. Apache Beam

### —— Google 大规模数据处理框架

Apache Beam 最初由 Google 开发，后来成为 Apache 基金会项目。

它提供统一的数据处理模型。

开发者可以使用同一套代码运行：

-   • 批处理；
    
-   • 流处理。
    

![](https://relay-1.bijitongbu.site/p/0ca33f5cf09fe093dfee8b8f25e6f9f3.png)

**项目地址：**

```
https://github.com/apache/beam
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 8.6K

**它提供什么：**

-   • 批处理计算
    
-   • 流式计算
    
-   • 数据流水线
    
-   • 多运行引擎支持
    

支持：

-   • Google Cloud Dataflow
    
-   • Spark
    
-   • Flink
    

**典型应用：**

-   • 大数据分析
    
-   • 实时计算
    
-   • ETL 数据处理
    
-   • 数据工程平台
    

### 77\. Firebase SDK

### —— Google 移动应用开发平台

Firebase 是 Google 提供的一套移动和 Web 应用开发平台。

Firebase SDK 提供：

应用连接云服务的能力。

![](https://relay-1.bijitongbu.site/p/44e1c3c458b66d56dbf650733ae68e8d.png)

**项目地址：**

```
https://github.com/firebase
```

> **开源协议：** Apache License 2.0（部分组件）

> **GitHub Star：** 多个仓库累计数万级

**它提供什么：**

-   • 用户认证
    
-   • 云数据库
    
-   • 消息推送
    
-   • 崩溃分析
    
-   • 数据统计
    
-   • 云函数支持
    

**典型应用：**

-   • 移动 App
    
-   • Web 应用
    
-   • 快速原型开发
    
-   • 创业项目
    

### 78\. go-github

### —— GitHub API 的 Go 语言客户端

go-github 是 Google 工程师维护的 Go 语言 GitHub API 客户端。

它帮助开发者通过 Go 程序访问 GitHub 服务。

![](https://relay-1.bijitongbu.site/p/742fceca25d7a8814e2e82af4d603c80.png)

**项目地址：**

```
https://github.com/google/go-github
```

> **开源协议：** BSD 3-Clause License

> **GitHub Star：** 11.3K

**它提供什么：**

-   • GitHub API 封装
    
-   • 仓库管理
    
-   • Issue 操作
    
-   • Pull Request 管理
    
-   • 用户信息查询
    

**典型应用：**

-   • DevOps 自动化
    
-   • CI/CD 工具
    
-   • GitHub 管理平台
    
-   • 开发者工具
    

### 79\. EarlGrey

### —— Google iOS 自动化测试框架

EarlGrey 是 Google 开源的 iOS UI 自动化测试框架。

它帮助开发者测试复杂移动应用交互。

![](https://relay-1.bijitongbu.site/p/aab17606fdfe8d3eb705c65da8353629.png)

**项目地址：**

```
https://github.com/google/EarlGrey
```

> **开源协议：** Apache License 2.0

> **GitHub Star：** 5.7K

**它提供什么：**

-   • UI 自动化测试
    
-   • 页面交互验证
    
-   • 异步测试支持
    
-   • iOS 测试工具
    

**典型应用：**

-   • iOS 应用测试
    
-   • 移动质量保障
    
-   • 自动化测试平台
    

### 80\. Lighthouse

### —— Google Web 性能分析工具

Lighthouse 是 Google 开发的自动化 Web 质量检测工具。

它可以分析网站：

-   • 性能；
    
-   • 可访问性；
    
-   • SEO；
    

![](https://relay-1.bijitongbu.site/p/dd1f67a9d06e11ab65f06dec8f805e1a.png)

**项目地址：**

```
https://github.com/GoogleChrome/lighthouse
```

**开源协议：** Apache License 2.0

**GitHub Star：** 30.7K

**它提供什么：**

-   • Web 性能评分
    
-   • 页面优化建议
    
-   • SEO 检测
    
-   • PWA 检查
    
-   • 可访问性分析
    

**典型应用：**

-   • 网站性能优化
    
-   • 前端工程质量检测
    
-   • Web 开发流程
    
-   • CI 自动检测
    

## 总结：Google 开源生态到底有多强？

Google 的 80 个开源项目，实际上构成了一张现代软件工程地图：

```
人工智能
   ↓
TensorFlow / JAX / Gemma

前端生态
   ↓
Angular / Flutter

云原生
   ↓
Kubernetes / gRPC / Istio

数据库
   ↓
LevelDB / Vitess

开发工具
   ↓
Go / GoogleTest

移动系统
   ↓
Android / Chromium

安全体系
   ↓
Tink / OSV
```

这些项目最大的价值，并不是代码本身。

更重要的是，它们代表了世界顶级工程团队解决复杂问题的方法。

如果你想提升技术能力：

-   • 学 AI，可以研究 TensorFlow、JAX；
    
-   • 学云原生，可以研究 Kubernetes、gRPC；
    
-   • 学系统，可以研究 Android、Chromium；
    
-   • 学安全，可以研究 Tink、OSS-Fuzz；
    
-   • 学工程能力，可以研究 Go、Bazel、Gerrit。
    

真正优秀的开发者，不只是会调用框架。

更重要的是：

> 知道优秀的软件系统，是如何被设计、构建和维护的。

往期推荐

> [图解 TCP/IP协议，看图秒懂](https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247487183&idx=1&sn=aefcdcb4b5e1699e63599be595a7a704&scene=21#wechat_redirect)
> 
> [Linux 值得学习的10个开源项目](https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247490558&idx=1&sn=6a1ce49560e9d52d89f518d70df7e301&scene=21#wechat_redirect)
> 
> [嵌入式从入门到精通：全靠这60个软硬件开源项目！](https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247491780&idx=1&sn=d6c4eddb1f5393b1234bfb1205571d81&scene=21#wechat_redirect)

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/949d8aa7_1787748633320?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkzNDk2NTUwOQ%3D%3D%26mid%3D2247492104%26idx%3D1%26sn%3D257d3b0c7f0c1c5fbc064b29273c4d97%26chksm%3Dc3e7ddb2418d903e1770baef2b48a8f5fef99a670d69280f8ad494972cff4519a9601c9f8fdf%26mpshare%3D1%26scene%3D1%26srcid%3D0826qwFnDlqTDuvMVHA25w1R%26sharer_shareinfo%3D13136bbe4878e81e443e29ff78656247%26sharer_shareinfo_first%3D13136bbe4878e81e443e29ff78656247%23rd&s=obsidian)