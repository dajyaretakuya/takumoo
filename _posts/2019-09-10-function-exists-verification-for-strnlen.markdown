---
layout: post
title:  "从一个HAVE_STRNLEN宏定义说起"
date:   2019-09-10 21:27:00 +0800
categories: [Computers, Coding]
tags: [C++]
---
最近碰到了一个奇怪的跨平台代码：
```cpp
// no strnlen on some OSes (Mac OS)
#if !HAVE_STRNLEN
size_t strnlen ( const char * s, size_t iMaxLen )
{
	if ( !s )
		return 0;

	size_t iRes = 0;
	while ( *s++ && iRes<iMaxLen )
		++iRes;
	return iRes;
}
#endif
```
看注释的意思似乎是在防止某些操作系统下没有`strnlen`函数而设置的跨平台代码。看括号里举的例子，似乎是`OSX`中不存在。查了一下`strnlen`是一个`GNU`扩展函数，并且在[POSIX (IEEE Std 1003.1-2008)](https://pubs.opengroup.org/onlinepubs/9699919799/functions/strnlen.html)中有明确规范。不过确实是早期的OSX中并没有这个函数，因此才会有这样的写法。（从`OSX 10.7`以后才有）

既然这样就好说了，我也不用担心在哪个平台，如果操作系统没有这个函数自然就会自己定义一个。但奇怪的事情出现了，在Windows下编译竟然会出现如下错误：
```shell
fatal error LNK1169: 找到一个或多个多重定义的符号
```
查了一下是`strnlen`被重复定义了。这么看来`Windows`下应该是已经有这个函数了，但为何这段跨平台代码失效了呢？很明显这里问题出在`HAVE_STRNLEN`的宏其实没有被定义。

那么问题来了，`HAVE_STRNLEN`这个宏是怎么来的？很明显这应该是在项目编译前的`config`时定义的。看了一下项目的`cmake`的写法：
```cpp
/* Define to 1 if you have the `strnlen' function. */
#cmakedefine HAVE_STRNLEN ${HAVE_STRNLEN}
```
有这么一段,可以看出`cmake`里是通过确认变量`${HAVE_STRNLEN}`来检查函数是否存在的。而这个变量又是通过下面代码进行赋值：
```cpp
include (CheckFunctionExists)
macro (ac_check_funcs _FUNCTIONS)
	foreach (it ${_FUNCTIONS})
		string(TOUPPER "${it}" _it)
		check_function_exists ("${it}" "HAVE_${_it}")
	endforeach(it)
endmacro(ac_check_funcs)
```
并能看到如下代码：
```cpp
AC_CHECK_FUNCS([dup2 gethostbyname gettimeofday memmove memset select socket strcasecmp strchr strerror strncasecmp strnlen strstr strtol logf pread poll])
```
其中可以看到函数`strnlen`的检测需求。

然而在Windows下，默认是不会使用`make`或者`cmake`的。那么Windows下如何能识别到这个宏定义？

原来，该项目通过直接`_WIN32`这个宏来定义一些不同平台所需的变量：
```cpp
#ifdef _WIN32
	#define USE_MYSQL		1	/// whether to compile MySQL support
	#define USE_PGSQL		0	/// whether to compile PgSQL support
	#define USE_ODBC		1	/// whether to compile ODBC support
	#define USE_LIBEXPAT	1	/// whether to compile libexpat support
	#define USE_LIBICONV	1	/// whether to compile iconv support
	#define	USE_LIBSTEMMER	0	/// whether to compile libstemmber support
	#define	USE_RE2			0	/// whether to compile RE2 support
	#define USE_RLP			0	/// whether to compile RLP support
	#define USE_WINDOWS		1	/// whether to compile for Windows
	#define USE_SYSLOG		0	/// whether to use syslog for logging

	#define UNALIGNED_RAM_ACCESS	1
	#define USE_LITTLE_ENDIAN		1
#else
	#define USE_WINDOWS		0	/// whether to compile for Windows
#endif
```
通过检查源代码，并没有发现`HAVE_STRNLEN`的定义，那么解决方案就很简单了，通过查阅`MSDN`，`strnlen`是肯定存在的，所以直接加上即可：
```cpp
#define HAVE_STRNLEN	1
```
再次编译，通过。

比起`cmake`或者`make`的方案，`Windows`总显得有点蠢是不是？是的，如果是`C Runtime`的程序，很可惜目前没有其他更合适的方法。如果是`Win 32`程序倒是可以用`GetProcAddress`和`LoadLibrary`来解决这个问题，但这不是本篇文章要讨论的课题了。