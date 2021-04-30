---
layout: post
title:  "在M1上尝试编译一些工具"
date:   2021-01-09 15:52:00 +0800
categories: [Machine Learning, Mathematics]
tags: [Theory of Machine Learning]
---


> 本文写于2021年1月9日且内容极具时效性，如果您在阅读的时候发现有信息已经过时，可以去对应项目的官网查看最新说明。

## 1 编译器

目前，GCC仍不支持M1的编译。尽管使用Rosetta2可以，但编译出的结果仍然是x86平台的目标代码。因此唯一的选择是MacOS自带的Clang。

## 2 Homebrew

光有编译器还不行，大型C++项目通常有各种各样的依赖。而Homebrew是管理这些依赖最方便的工具之一，幸运的是，现在Homebrew已可以在M1下运行，

![M1下运行的Homebrew]({{site.url}}/assets/img/M1/homebrew.jpeg){:class="img-responsive"}

尽管官方的更新日志仍然推荐使用Rosetta2版的Homebrew，但只要安装路径和传统的/usr/local区隔开（例如官方推荐的/opt/homebrew），那么大部分库和工具都可以顺利安装。这里推荐一个新旧两个版本的Homebrew共存的方法，可以参考这篇博客：

[Setup Homebrew on Apple Siliconnoahpeeters.de](https://noahpeeters.de/posts/apple-silicon/homebrew-setup/)

在使用brew install命令安装某个依赖的时候，会优先寻找是否已有编译好的AArch64版本，如果没有，会直接下载源代码重新编译。

## 3 编译代码

工具准备好了，就可以开始尝试编译项目。一些项目不存在任何依赖，直接configure后make即可。但一些项目存在大量依赖，就需要先用brew命令安装。这里拿libopenshot这个开源项目作为例子。

首先按照官方文档描述，用之前准备好的原生Homebrew安装下列依赖：

```
brew install ffmpeg librsvg swig doxygen unittest-cpp qt5 cmake zeromq libomp python@3.7
```

但安装完上述依赖还不够，还需要一个homebrew无法提供的库——libopenshot-audio。因此从官方拉下来这个项目的代码到一个新目录里。然后使用cmake进行构建。然而马上就会遇到问题，我们会遇到一个函数未声明的错误。该函数位于juce\_osx\_ObjCHelpers.h中：

![函数未声明]({{site.url}}/assets/img/M1/juce_issue_01.jpeg){:class="img-responsive"}

通过Apple官方文档可知，这该函数是用来解决x86架构下返回浮点值的函数在ABI上无法和返回整型值的函数兼容，因此在x86平台下必须使用objc\_msgSend\_fpret来返回非整型值，但非x86平台是不需要这个函数的：

[https://developer.apple.com/documentation/objectivec/1456697-objc\_msgsend\_fpretdeveloper.apple.com](https://developer.apple.com/documentation/objectivec/1456697-objc_msgsend_fpret)

但libopenshot-audio的这处代码显然是将iOS认为是目前苹果唯一的AArch64架构的平台，因此我们可以做下面这样的改动：

![JUCE代码的改动]({{site.url}}/assets/img/M1/juce_issue_02.jpeg){:class="img-responsive"}

我们将这里用于判断目标平台的宏定义加了一个JUCE\_ARM，来彻底把这段代码限制到只存在于x86平台上。（这里我不知为何一开始就不用ARM宏来限制，毕竟这些宏在最开始是一起定义好了的）再次编译，成功。接下来的libopenshot就比较好办了，只需要设置好指定了libopenshot-audio安装根目录的环境变量，即可编译成功。

## 4 总结

M1下的原生应用确实比Rosetta2运行效率高很多，拿Java来说，我使用Zulu的AArch64版本的JDK做一个最简单的循环累加测试，最高会比使用Rosetta2的JDK性能会有40%的提升。不过，并非所有的软件都能像libopenshot一样只修改少数地方即可编译成M1原生应用，下面这个清单列出了目前常用的软件在M1下的支持情况：

[https://github.com/ThatGuySam/doesitarmgithub.com](https://github.com/ThatGuySam/doesitarm)

可以作为一个使用前的评估依据。

> 这份清单也具有实效性问题，因此如果在对某个具体的项目存在疑惑，建议去官方网站查看其支持情况。
