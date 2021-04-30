---
layout: post
title:  "深入理解GUI (一) GUI的基本理论（未完）"
date:   2019-10-05 15:52:00 +0800
categories: [Computers, OS]
tags: [Computer Graphics]
---

# 第一章 概述

## 1 一点历史

> 下述历史内容基本都是从Wikipedia得来，仅供参考

**人机交互 ( Human–Computer Interaction )** 领域研究如何在人与计算机之间建造高效沟通的桥梁，这个桥梁就是我们现在经常听到的**用户界面 ( User Interface, UI )** 。UI 无处不在，可以说，只要涉及到人机交互的设备：游戏机、手机、MP3播放器、嵌入式设备等，都需要 UI 。我们熟悉的两种 UI 分别是**命令行界面 ( Command Line Interface )** 和 **图形用户界面 ( Graphic User Interface )** 。

命令行界面从早期的**电传打字机 ( Teleprinter, TTY )** 里的一种**对话框 ( Dialog )** 演化而来，这也是**终端 ( Terminal )** 这个词得以由来。我们现在看到的终端其实只是一个模拟器，早期的终端实际上就是一台机器。

1973年，**原施乐帕罗奥图研发中心**生产的  **Xerox Alto** 成为了世界上第一台搭载了完整的图形用户界面的计算机（2）。这台计算机具有一个完整的图形窗口系统，包括窗口、菜单、单选按钮和复选框，并且当时就有了指针设备和键盘。并在1981年正式商业化，并搭载了 Xerox Star 图形化操作系统。



![Xerox Alto 上的图形用户界面]({{site.url}}/assets/img/GUI/Xerox-Alto.png){:class="img-responsive"}

<center>图 1-1 Xerox Alto 上的图形用户界面</center>



Xerox Star 虽然没有大卖，但由于 Xerox Star 带来的影响，促进了一批图形用户界面的操作系统的诞生，其中就包括 Apple Macintosh 和 Microsoft Windows 。Macintosh 取自于一种苹果的名字 ( mcintosh, 麦金托什苹果 )，最早的型号是 Macintosh 128K，于1984年1月发布。借着当时超级碗广告的造势(3)，到同年三月的时候，就取得了惊人的7万台的销售业绩【6】，这使得 Macintosh 128K 成为历史上第一次被大规模销售的带有图形用户界面的个人计算机【8】。然而，精美的Macintosh 128K由于糟糕的性能使其销量后劲不足【7】。

1985年的计算机经销商博览会 ( Comdex ) 微软正式发布了为 IBM 个人电脑打造的图形化操作系统 Windows 1.0，然而，Windows 1.0 仅仅只是一个 DOS 的图形化接口，在视觉体验表现并不如 Macintosh 那么惊艳，因此也没有掀起太大波浪。尽管如此，Apple 和 微软并没有放弃对 GUI 的追求，在1987年，微软发布了 Windows 2.0 和 Windows 2.1 操作系统，1988年，离开了苹果公司的乔布斯也发布了 NeXT 电脑。但 NeXT 并没有引起市场轰动，失去了乔布斯的苹果公司也一直萎靡不振，另一方面，微软用了多年时间打造的 Windows 3.0 也于1990年推出，此后借由1995年发布的 Windows 95 操作系统成为了史上最成功的个人计算机操作系统。微软也从此开始了桌面操作系统的市场绝对占有优势。

苹果公司经历了1990年到1997年的低谷期，并迎来了乔布斯的回归后，终于在1998年开始复苏，并于2001年推出了基于 UNIX 内核的 OS X 操作系统 ( 以前的苹果操作系统称为 Classic Mac OS，即经典的 Mac OS，一共有9代，均**不是** UNIX 内核的操作系统 )，也就是我们今天看到的 MacOS，并于2005年正式开始使用 Intel 的处理器，直到现在 (5)。



## 2 实现

2.1 终端

早期人们实现计算机图形的方式非常艰辛，用尽了各种智慧才只能实现像 Macintosh 128k 那样的简单图形产品 ( 尽管在当年，Macintosh 已经非常不简单了 )。现代计算机技术发展迅速，图形实现方式已趋于标准化，大部分都是通过**光栅扫描系统**显示图形【10】，操作系统通过专门的模块渲染图形，并实现了一套窗口管理器，通过与**输入输出设备**的交互实现图形用户界面【1】。

![简单的光栅图形系统架构]({{site.url}}/assets/img/GUI/os-simple-architecture.jpg){:class="img-responsive"}

<center>图 1-2 简单的光栅图形系统架构</center>

前面我们提到过，现在的终端都是图形模拟出来的，早期的终端都是独立的设备。在Linux中，设备分为字符设备和块设备。块设备的一个例子就是可以随机存取的磁盘。字符设备通过串口的方式与操作系统进行通信。终端是典型的字符设备，通过字符设备驱动来输入和输出字符。后来的计算机引入了视频卡，支持更丰富的显示，因此引入了视频卡驱动，我们需要让显示器显示的内容可以写入到**帧缓冲**，最终会以光栅化的方式显示出来。下面这片文章很好的介绍了终端，推荐一读。

[命令行界面 (CLI)、终端 (Terminal)、Shell、TTY，傻傻分不清楚？](https://segmentfault.com/a/1190000016129862)

### 2.2 窗口管理器和X System

**窗口构成了桌面环境**，GUI 中的人机交互更多是人与窗口的互动，因此窗口管理是GUI最重要的组件。早期，窗口直接通过帧缓冲绘制，整个桌面的信息都存储在一个帧缓冲里。后来人们有了更高的渲染需求，印日了X System和窗口管理器。X System只是一个协议，但他提供了一个沿用至今的C/S架构模型，让Unix/Linux上得以运行各种桌面系统。（待补充）

### 2.3 Wayland

X System尽管有悠久的历史，但这个架构天然携带的性能缺陷至今仍被诟病，因此社区开始考虑更好的架构用于GUI系统。Wayland是其中最引人注目的一支。（待补充）

### 2.3 带有 Darwin 核心的 Mac OS

Mac OS 的系统核心是一个完整的 UNIX 实现，代号叫做 Darwin 【2】。但 Mac 系统并没有走其他 UNIX 系统采用 X11的老路 ，而是采用了一个全新的图形系统 —— **Quartz** 。Quartz 首次于1999年五月的 WWDC 披露，Quartz 能够保存矢量信息，可以对已经显示出来的图形进行无损变换。尽管到现代矢量显示器并没有普及，但保存矢量信息的优势仍然存在，例如 macOS 可以轻松的输出 PDF，并且默认就支持矢量到栅格的处理，对于 Adobe Illustrator 这样的图形处理软件就有了非常大的帮助，他们可以不需要实现自己的矢量到栅格中间的处理层了。

Quartz 由两部分组成： **Core Graphic Service  ( 核心图形服务 )** 与 **Core Graphic Rendering ( 核心图形渲染 )** 。Core Graphic Service 负责处理事件以及组合需要显示到屏幕上的元素，呈现出 **与实现细节无关的绘图模型 ( drawing model agnostic )** ，然后通过 Core Graphic Rendering 把这些元素的矢量描述，转化成像素，呈现到光栅显示器里。

值得注意的是，Core Graphic Rendering 使用 Adobe 的 PDF 来描述它的内部矢量图形（ 这里 PDF 不是指文件格式，而是 Adobe 设计的一整套矢量描述语言 ），桌面应用程序的元素都会经过 PDF 引擎渲染到桌面上，但 3D 游戏这类直接利用 OpenGL 或者 Metal 渲染的图形则不经过 Core Graphic Rendering。



> 注意，Apple 官方文档以 API 的角度描述Quartz，在官方文档中， Quartz 主要由 Quartz 2D API 和基于这套 API 实现的 Core Graphic Framework组成，并可以与 Core Image, Core Video, OpenGL 和 QuickTime 协同工作。



### 2.4 把 GUI 做在内核中的 Windows

无论是 X Window 还是 mac OS，它们负责图形桌面的组件都位于操作系统的**用户空间 ( User Space )** 中，而微软的 Windows 操作系统则另辟蹊径，把图形桌面系统做在了**内核空间 ( Kernel Space )** 中，大大提高了桌面渲染速度，他们称这套系统为 **窗口和图形系统 ( The Windowing and Graphics System ) 【5】**。我们在桌面看到的窗口环境以及开发一些基础图形功能用到的 GDI 函数都位于这个层。尽管把图形系统做在内核空间大幅度提升了桌面环境的性能，但也会给内核带来更大的不稳定性。



## 3 注释

（1）是的，你可能会联想到现在家喻户晓的 SSH 客户端 PuTTY。从概念上来看，PuTTY 确实是一个模拟的终端，但 PuTTY 的名字来源至今尚不确定。 

（2）帕罗奥多研究中心是许多现代计算机技术的诞生地，他们的创造性的研发成果包括：个人电脑、激光打印机、鼠标、以太网；图形用户界面、Smalltalk、页面描述语言Interpress（PostScript的先驱）、图标和下拉菜单、所见即所得文本编辑器、语音压缩技术等。 

（3）超级碗（Super Bowl）是NFL职业橄榄球大联盟的年度冠军赛，胜者被称为“世界冠军”。超级碗多年来都是全美收视率最高的电视节目，并逐渐成为美国一个非官方的全国性节日，因此超级碗广告带来的影响非同小可，苹果的这个广告当年花费了37万美元，在我们这个年代相当于百万美元的广告成本了。 

（4）早期使用 TCP 实现图形界面因为往返通信延迟，极容易影响图形程序性能，因此后来为了改善这个问题，改用了UNIX Domain或者共享内存作为通信方式。

（5）苹果于2020年6月WWDC宣布下一代Mac产品将使用基于ARM架构的自研片上系统 ( SoC )

## 4 参考文献

【1】 《操作系统概念》Ch.13.2 

【2】 《Mac OS X and iOS Internals To the Apples Core by Jonathan Levin 》 Ch.2 

【3】《Hands-On GUI Application Development in Go by Andrew Williams (z-lib.org)》 Ch.1 

【4】《iOS and macOS Performance Tuning Cocoa, Cocoa Touch, Objective-C, and Swift by Marcel Weiher (z-lib.org)》 Ch. 14 

【5】《Windows Internals, Part 1 System architecture, processes, threads, memory management, and more by Pavel Yosifovich, Alex Ionescu, Mark E. Russinovich, David A. Solomon (z-lib.org)》 Ch.2 

【6】https://en.wikipedia.org/wiki/Macintosh_128K 

【7】https://en.wikipedia.org/wiki/Steve_Jobs#Pre-Apple 

【8】https://en.wikipedia.org/wiki/Macintosh 

【9】https://en.wikipedia.org/wiki/IBM_Common_User_Access

【10】《计算机图形学》Ch.2