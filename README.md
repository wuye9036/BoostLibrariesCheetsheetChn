Boost库的分类与简介

本文的目的在于帮助大家梳理一下Boost中有哪些库，包括那些库是不再需要的，类似的库之间有什么共同点和不同点，以免胡造轮子。 Boost这些库就和鸡啄米一样散地到处都是……

元编程

糊弄编译器

Identity

类型萃取和修饰

Type Traits

这个大家应该很熟悉了，通过它们可以判断类型的特性，比如是不是一个整数类型，是否有符号。它也可以为类型做一些修饰，比如变换到引用类型、常类型等。这个库C++11之后基本上就完全std中的type traits取代了。

但是这个库对一些类型，如成员函数支持不足，也不能做一些更加精细的检查，比如探测类中是否存在某个成员，这时就需要下面几个库进行补充。

Call Traits

这个库可以把一些不能作为函数参数的类型，如数组类型修饰成可以作为参数的类型，比如：

template <class T>
struct A {
   void foo(T t);
};

template <class T>
void A<T>::foo(T t){
   T dup(t); // doesn't compile for case that T is an array: A<int[2]>
}

// 使用call_traits

template <class T>
struct A {
   void foo(typename call_traits<T>::value_type t);
};

template <class T>
void A<T>::foo(typename call_traits<T>::value_type t) {
   typename call_traits<T>::value_type dup(t); // OK even if T is an array type.
}

实际上在使用中这个库并不常用，因为大部分情况下A<T>这种都会要求T是值语义的比较多，为啥非要用一个数组或者一个引用进去作死呢？

Callable Traits

这个库是1.66中的新加入库，可以为成员函数活、一般函数或者任意函数对象调整类型修饰，比如

struct foo {};
using pmf = void(foo::*)();
using expect = void(foo::*)() const volatile;
using test = ct::add_member_cv_t<pmf>;
static_assert(std::is_same<test, expect>::value, ""); // True

C++11/14中的add_cv是没办法调整函数的类型修饰的。

TTI

这个库从1.54开始加入，也算是个比较新的库了。它算是对Type Traits的一个补充，一看几个头文件名就知道它是做什么的了：

// ...
#include <boost/has_member_data.hpp>
#include <boost/has_member_function.hpp>
// ...

不过从C++11开始，因为有了Expression SFINAE，以及decltype/declval之类的存在，它的应用场景就没有那么广泛了。

自省

Typeof

获得变量的类型，并支持自动类型变量。C++11之前比GCC的typeof要好用，C++11/14以后它的用途已经可以完全被 decltype, decltype(auto) 和 auto 所取代。

Type Index

C++11开始，标准库中引入了type_index。所以接下来又到了做下面几个概念做近义词辨析的时候了：typeid, std::type_index 和 boost::type_index

最基本的是C++98时代就有的typeid。它在运行时返回一个type_info，其中保存了类型的一些信息，比如name。但是这个对象平时真的没什么卵用，所以C++11开始，加入了type_index，作为type_info的封装，它能够支持比较和hash，这样就能把类型信息存在字典中了 —— 比如字典中可以放上factory method，这样就相当于有了一个比较挫但是能用的动态创建对象的功能。

不过typeid需要打开编译器的RTTI，而且它生成的名字也不太好（特别是函数和模板类型），所以Boost搞了一个自己的Type_index来解决这一些问题，算是是std::type_info的一个增强。

类型检查与约束

Concept Check

这个库从1.19就加入了boost （1.19是2000年发布的 —— 这都TM快20年了），目的就是做Concept的检查，然而用处真的很小 —— 你检查了又能怎么样，出错信息还是巨难看，就和C++11之前的Static Assert一样。

看，Concept是C++98时代的东西了，C++20都还进不来。这么难产的特性，要么没卵用，要么编译器难实现，要么程序员难用；依我看，Concept三个毛病都占齐了。

Enable If

Boost中要多一个disable_if，其余直接用 std::enable_if 即可。

元编程数据结构与算法

MPL

C++11之前用于元编程的库。提供了一系列的类型的容器、算法 (find， count， sort 等）以及辅助函数。它在类型系统上，提供了类STL的库以像变量那样操作类型。

比如下面这个例子中，floats是一组类型的“vector”，可以通过push back加入新的类型，并使用at取出对应的类型并判断。

typedef vector<float,double,long double> floats;
typedef push_back<floats,int>::type types;

BOOST_MPL_ASSERT(( is_same< at_c<types,3>::type, int > ));

在C++11之后，应使用Boost 1.66新加入的MP11。

MP11

C++11版的MPL。在C++11之后应该使用它，编译速度要快一些。

运行时泛化

Type Erasure

宏元编程

Preprocessor (PP)

MPL提供了一套模板的库运算，PP则提供了宏上的元编程库。它也提供了一些类似于容器的东西，比如List，Tuple等，容器中的元素就是标识符。

感受一下它的用法：

#include <boost/preprocessor/cat.hpp>
#include <boost/preprocessor/list/for_each.hpp>

#define LIST (w, (x, (y, (z, BOOST_PP_NIL))))

#define MACRO(r, data, elem) BOOST_PP_CAT(elem, data)

BOOST_PP_LIST_FOR_EACH(MACRO, _, LIST) // expands to w_ x_ y_ z_

这东西看起来非常异端，不过它对代码展开会很有帮助 —— 比如可以用它来控制把for循环展开若干层深度；或者生成若干个变量的名字。

高级技能可以不学，不过 STRINGIZE，CONCAT，COMMA 要记住，以后总能用得上。

VMD

VMD是对PP的一个扩充，提供了对PP中数据和数据结构的操作。比如把PP中的List转换成PP中的Tuple；判断一个输入是不是一个PP的列表；给PP提供了push back, pop back等操作函数。要想用它，你得先用得到PP。

杂耍

Hana

这个库需要C++14及以上的支持。这个库骨骼比较惊奇，基本思路是把类型变成值，经过运算后，再把值变成类型。文档自己也说了，它是MPL和Fusion的混合体。如果你是想实现类似于ducking type这样的特性，那是Hana好用许多。

举个例子：

boost::any a = 'x';
std::string r = switch_(a)(
  case_<int>([](auto i) { return "int: "s + std::to_string(i); }),
  case_<char>([](auto c) { return "char: "s + std::string{c}; }),
  default_([] { return "unknown"s; })
);

assert(r == "char: x"s);

文件格式

文本

PropertyTree

可以以树的结构来读写XML, JSON, INI和INFO文件。

图像

GIL

图像文件的读写库。这个库是Adobe贡献的。

这个库把图像的读写抽象成了Image View和Pixel Iterator/Allocator，这样可以灵活处理像素格式。本身只支持Jpeg，Tiff和Png格式，不过可以挂接其它扩展。
控制流

Context

自从有了Lambda之后，Boost心就野了，想支持现代语言普遍支持的并发机制Co-routine。Co-routine的基本思想就是让多个函数共享执行线程，而且可以主动打断自己的执行。对于C++来说，一个基本的实现方法就是到打断的时候，把整个栈（当然还有保存下来的寄存器状态）拷贝出去，然后跳转到其他要执行的函数上；然后要继续执行的时候，再把栈拷贝回来。

Context这个库，就是帮你做上下文保存和切换的事情（当然还有call/cc的实现）。

C++的栈和C相近，基本上直接就是机器和OS上直接支持的栈。所以很明显，这里的Context代码，应该每一代CPU，每个OS、甚至是每个编译器都要来一份的 —— 而且基本上还得用汇编。

然后Boost就真的这么干了。

如你们所见，从1.51开始，Boost正式加入了Context。

有了Context之后，更高层的库就容易实现多了（我不信）。

Coroutine

1.53 开始加入了Coroutine，这个对C++11不是强制要求的。

Coroutine2

1.55 加入了Coroutine2，并且 Coroutine 停止维护了。只要编译器支持C++11，就应该用后面这个库。

关于这个库其实就三板斧，push type，pull type，yield。但是这文档当年我看了大概快半天的时间，才画出了一个葫芦。这里有个例子：
https://theboostcpplibraries.com/boost.coroutine​
theboostcpplibraries.com

Chapter 51. Boost.Coroutine
Chapter 51. Boost.Coroutine​
theboostcpplibraries.com

Fiber

Fiber和Co-routine在概念上非常相近，API也比较类似。它们都有调用方主动将自己切换出去的过程；并且在Boost上，都是由Context库实现状态保存和切换的。

二者主要的区别在于，Fiber和线程类似是有调度器的，yield之后，控制权交由调度器处理接下来调用哪个fiber；而co-routine是没有调度器的。Co-routine A代码需要把自己yield到Co-routine B上，Co-routine B则可以在适当时候，主动切换回A，或者等执行结束返回A继续执行。
http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4024.pdf​
www.open-std.org

Scope Exit


数据流

Iostreams

Iterator

Range
网络

Beast

Http协议的基础库，可以用于开发Http协议的客户端和服务器端。
Socket部分是用ASIO实现。

ASIO
容器与数据结构

单数据存储

Any

Variant

静态长度结构

Array

Multi-Array

类C++98 STL容器

Intrusive

Container

Pointer Container

Poly Collection

Tuple

Unordered

Fusion

Fusion类似于STL中的容器、迭代器和算法，只不过它可以工作在 Boost.Tuple和Fusion Map之上。看下面的例子：

// 一个 “vector”（实际上就是tuple），有两个元素一个int一个float
vector<int, float> v(12, 5.5f);
std::cout << at_c<0>(v) << std::endl;
std::cout << at_c<1>(v) << std::endl;

// 这里是一个 "map"，不过这个map的key实际上是一个类型而不是一个变量或值。
typedef map<pair<int, char>, pair<double, std::string>> map_type;
map_type m(make_pair<int>('X'), make_pair<double>("Men"));
std::cout << at_key<int>(m) << std::endl;
std::cout << at_key<double>(m) << std::endl;

它提供了和 struct 和 std::tuple 相兼容的视图，同时还能做类似于joint和zip之类的操作。


非 C++98 STL的数据结构

Compressed Pair

Heap

ICL

Property Map

Bimap

Circular Buffer

Dynamic Bitset

Lockfree

Multi-Index
函数与高阶函数

C++14上没用的

Bind

Function

Function Types

Lambda

Local Function

Member Function

Functional

Functional Forward

上面这些库，一些是用来生成匿名函数的，比如bind，lambda和local function，这些用C++11的lambda就能搞定，但是C++11之前就得各种奇技淫巧；还有一些是用于保存不同来源的函数对象的，比如function，member function之类，这些基本上就是C++11的function搞定。最后一个Functional Forward，就是瘸腿版的C++11 perfect forward。

C++14上还有用的

Functional/Overloaded functions

这个东西用于把多个签名的函数打包到一个名字中。看例子就一目了然了：

boost::overloaded_function<
      const std::string& (const std::string&)
    , int (int)
    , double (double)
> identity(identity_s, identity_i, identity_d);

// All calls via single `identity` function.
BOOST_TEST(identity("abc") == "abc");
BOOST_TEST(identity(123) == 123);
BOOST_TEST(identity(1.23) == 1.23);

但是说老实话，函数重载到底要调用谁难道自己写的时候心理还没有点逼数吗。。。

Phoenix
平台、硬件、操作系统服务

系统配置

Config

Predef

Compatibility

时间

Chrono

Date Time

Timer

硬件特性

Align

Atomic

Compute

Endian

操作系统服务

System

Filesystem

DLL

Thread

Process

Interprocess

日志

Log
数学、数值与物理

基本数值

Math/Common Factor

Integer

Multiprecision

Numeric Conversion

Rational

QVM

Math / Octonion

Math / Quaternion

Interval

线性代数

uBlas

Odeint

函数求值

Math/Special Functions

几何

Geometry

Polygon

离散

CRC

Functional/Hash

Sort

标准库里面的sort是基于比较排序的，这个库搞了点trick比如基数排序什么的，和标准库中的sort比，它对整数、字符串和浮点都提供了更快的排序实现。

Graph

GraphParallel

随机、概率与统计

Random

Min-max

Accumulator

Statistical Distributions

单位换算

Units

Ratio
文本分析与处理

字符串与值的转换

Convert

Format

Lexical Cast

本地化

Locale

一般字符串处理算法

String Algo

正则

Xpressive

Regex

Boost中一共两套Regular Expression的实现，一个是Xpressive，一个是Regex。前者的特点是，有一套方言能够在编译期给整一个DFA出来（当然也能在运行时compile出来，不过这个就没特色了）；而后者则只能在运行时整了。

下面是个整个编译期DFA的例子。

sregex re = '$' >> +_d >> '.' >> _d >> _d;
// 这货等价于： $+\d.\d\d

这TM看着就是Spirit的文法啊。

按照上古时代的性能评测，Xpressive要快很多。

不过大部分情况下，Regex就够用了，而且这个库在C++11时代也成为了标准库的一部分，不需要用到Boost中的实现了。

文法分析

Spirit

特定语法分析器

Wave

这个库提供了C++中预处理器（Preprocessor）的实现。如果用户在开发一个新的语言编译器/解释器，并且想让这个语言拥有等同于C++11预处理器的功能（包括include，条件编译，宏替换），就可以直接使用这个库。
语法糖与语言扩展

语法糖

Assign

Foreach

Operators

Optional

Parameter

Ref

Swap

Tribool

方言

Metaparse

Proto
设计模式与习惯用法

教做人

Value Initialized

惯用法

Serialization

Signals（废弃）

Signals2

GoF设计模式

Flyweight

Functional/Factory

In-place factory

Mate State Machine

Statechart

Pool
调试、测试与诊断

Assert

Static Assert

Stacktrace

ThrowException

Test

Conversion

Core

Exception

IO State Savers

Move

Program Options

Python

Smart Ptr

Utility

Uuid

* 附录

** 简介模板
关键字，承继关系，语言特性，库依赖，描述（可能包括原理，实现，优缺点，使用建议）

** 待更新列表
列表更新至 1.70.0，除了：Contract, HOF, YAP, Safe Numerics。
