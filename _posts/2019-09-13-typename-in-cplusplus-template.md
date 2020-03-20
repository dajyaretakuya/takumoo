---
layout: post
title:  "C++模板的一些二义性语法讨论"
date:   2019-09-13 11:38:00 +0800
categories: [Computers, Coding]
tags: [C++ template]
---
首先大家看下面一段代码：
```cpp
template<typename T>
std::vector<T>& arrayRotate(const std::vector<T>& src)
{
	std::vector<T> dst(src.size());
	for (std::vector<T>::iterator iter = src.begin(); src.end() != iter; iter++)
	{
		dst.push_back(*iter);
	}
	return dst;
}
```
如果能一眼发现哪里写错了，那么可以不用往下看了（手动狗头）

如果在VC中编译会提示`for (std::vector<T>::iterator iter = src.begin(); src.end() != iter; iter++)`这一行少了个`;`，但这明显不可能，仔细看下行中每个`token`的`Code IntelliSense`会发现在`iter`这个词上会提示`<error-type> iter`，那么问题很明显了，`std::vector<T>::iterator`这个类型无法别编译器确定。解决方案就很简单了，直接在前面加上`typename`以表明这是一个**类型名字**：
```cpp
for (typename std::vector<T>::const_iterator iter = src.begin(); src.end() != iter; iter++)
```
再次编译，问题解决。（《C++ Primer》提到这里不能用`class`来代替`typename`[3]，但我在VC2019中测试过这里不用`typename`而使用`class`也是可以的，可能是微软为了兼容早期`C++`中并没有`typename`的情况，在其他编译器中不一定可以通过，因此你也应当尽量只使用`typename`）

那么接下来我们就来探讨一下这个`C++ Template`中比较容易被忽略的问题之一：`typename`关键字。

## 1 关键字`typename`
看下面这个例子：
```cpp
template <typename T>
class SomeClass {
    typename T::SubType * ptr;
}
```
这里和本文开头有问题程序中的`std::vector<T>::iterator`要使用`typename`的目的是一样的，`typename`是为了说明**模板内部的标识符可以是一个类型**。`SubType`是定义在类`T`内部的一种类型。因此，`ptr`是一个指向`T::SubType`类型的指针。如果不使用`typename`，那么`SubType`就会被认为是一个静态成员，那么它就会被解析成具体的变量或者对象，于是`T::SubType * ptr`就会被看作是类T的静态成员`SubType`和`ptr`的乘积[1]。

## 2 `.template`和`->template`
看下面这个例子：
```cpp
template <int N>
void printBitset (std::bitset<N> const& bs)
{
    std::cout << bs.template to_string<char, char_traits<char>, allocator<char> >();
}
```
其实这段代码本意是想调用`bs`的`to_string`成员函数，这个成员函数原型是这样的(注：在`C++ 11`以前，这个函数叫做`basic_string`，于`C++ 11`改名为`to_string`)：
```cpp
template<
    class CharT = char,
    class Traits = std::char_traits<CharT>,
    class Allocator = std::allocator<CharT>
> std::basic_string<CharT,Traits,Allocator>
    to_string(CharT zero = CharT('0'), CharT one = CharT('1')) const;
```
那么这里为何不直接调用`bs.to_string`而要插入一个奇怪的`.template`声明呢？原因在于**这里传入参数bs是依赖于模板参数N构造的**，如果没有这个声明，编译器无法确定后面的小于号(`<`)并不是数学中的小于号，而是模板是参列表的起始符号[1]。

因此，我们不难总结出，**只有当前面存在依赖于模板参数的对象时，才需要在模板内部使用`.template`或者`->template`来避免产生`二义性`**

## 3 产生二义性的原因
其实前文已经提到，产生二义性的原因主要是因为这些地方都是跟模板参数有关的。在引入模板以后，我们可以把`C++`中的`name`分为两种：`Dependent names`和`Non-dependent names`[2]。所谓`Non-dependent names`顾名思义就是完全独立的名字，而`Dependent names`就是受制于模板参数影响的名字，在模板`具象化`之前是是无法真正确定这个名字含义的。(有的地方可能会使用`实例化`这个词，但我更偏好于`具象化`，以区别于类的实例化)因此就需要一些额外的语法来保证编译器正确工作。

## 4 其他语言的处理方式
这套规则查看下总感觉没那么智能，而且有传言本来`C++ 11`是要优化这个问题的[2]，但似乎现在其实没有太大改变。那么我们来看看同样具有template语法的Java是如何处理这个问题的吧。
```java
class Pair<T>
{
    private T value;
    private SubType subObject;

    public Pair(T val) {
        value = val;
        subObject = new SubType();
        subObject.subValue = 1000;
    }

    public class SubType {
        public int subValue;
    }

    public SubType getSubObject() {
        return subObject;
    }
}

public class Main {
    public static void main(String[] args) {
        Pair<Integer> pair = new Pair<>(666);
        Pair<Integer>.SubType subObject = pair.getSubObject();
        int result = subObject.subValue;
        System.out.println(result);
    }
}

```
这里并没有类似`C++`中的`typename`这样的关键字，编译，运行，成功获得结果：
```shell
1000
```
看来`Java`中并没有`C++`那么多的约束，但这实际上是因为`Java`中没有`模板特化(Template Specialize)`这种东西，呃，这又是一个很大的话题了，以后有机会再说吧（笑）。

画外音：这次标题怎么又是这个句式Orz

参考文献：
[1] 《C++ Templates》, [美]David Vandevoorde [德]Nicolai M. Josuttis, 陈伟柱 译, 人民邮电出版社, 2013
[2] 《A Description of the C++ typename keyword》, Evan Driscoll, http://pages.cs.wisc.edu/~driscoll/typename.html
[3] 《C++ Primer 5e》, Stanley B. Lippman, 《电子工业出版社》,  2013.9
