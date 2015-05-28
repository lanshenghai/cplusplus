
#C++ 右值引用解释

翻译自http://thbecker.net/articles/rvalue_references/section_01.html

介绍
====
右值引用是在C++11标准里加入到C++语言一个的特性。右值引用最引起大家费解的原因是当你第一次看到它的表示方式，它无法清晰的表达它们的作用或者它们想解决什么问题。因此，我不打算直接跳进去去解释什么是右值引用。相反，我打算从它要解决的问题开始，然后给出右值引用是如何提供解决方案。以这种方式，右值引用的定义会以一种合理和自然的方式呈现给你。

右值引用至少解决下面两个问题：

1. 实现move语法
1.  完美实现传递

如果你不熟悉这些问题，也不用担心。它们两个会在下面被详细解释。我们先开始move语法。但是在我们准备开始之前，我需要提醒你：在C++里什么是左值、什么是右值。给出一个严格的定义非常难，但是下面的解释足够好的实现这个目的。

最早的左值和右值定义来自于早期的C语言：左值是可以出现在赋值语句的左边或者右右，而右值只能出现在赋值语句的右边。例如：

    int a = 42;
    int b = 43;

    // a and b are both l-values:
    a = b; // ok
    b = a; // ok
    a = a * b; // ok

    // a * b is an rvalue:
    int c = a * b; // ok, rvalue on right hand side of assignment
    a * b = 42; // error, rvalue on left hand side of assignment

    // a and b are both l-values:
    a = b; // ok
    b = a; // ok
    a = a * b; // ok

    // a * b is an rvalue:
    int c = a * b; // ok, rvalue on right hand side of assignment
    a * b = 42; // error, rvalue on left hand side of assignment

在C++里面，作为第一、直观对左右值的理解，这些定义还是有用的。然而，在C++世界里，由于引入了用户自定义的数据类型，可修改和可赋值特性发生了微妙的变化，从而导致这些定义不再正确。我们不需要再深入的探讨这部分内容。这儿给出一个替代的定义，尽管还是很有争议，但它可以帮助你解决右值引用问题：**左值是对一个内存地址的引用的表达式，并且允许我们通过&运算符得到内存地址。右值是一个不是左值的表达式**。例如：

    // lvalues:
    //
    int i = 42;
    i = 43; // ok, i is an lvalue
    int* p = &i; // ok, i is an lvalue
    int& foo();
    foo() = 42; // ok, foo() is an lvalue
    int* p1 = &foo(); // ok, foo() is an lvalue

    // rvalues:
    //
    int foobar();
    int j = 0;
    j = foobar(); // ok, foobar() is an rvalue
    int* p2 = &foobar(); // error, cannot take the address of an rvalue
    j = 42; // ok, 42 is an rvalue

如果你对左值和右值的严格定义感兴趣的话，可以参考Mikael Kilpel鋓nen's [ACCU article](http://accu.org/index.php/journals/227)。

Move语法
========
假设X是一个类，它包含一个指向一些资源的指针或句柄，命名为m_pResource。这里的资源的意思是：我们需要考虑构造、复制和析构的成本。一个好的例子是std::vector，它是一个对象的集合，在内存中以数组方式存放。然后，逻辑上，X的拷贝赋值操作如下：

    X& X::operator=(X const & rhs)
    {
      // [...]
      // Destruct the resource that m_pResource refers to. 
      // Make a clone of what rhs.m_pResource refers to, and 
     // attach it to m_pResource.
     // [...]
    }

类似的原因应用到拷贝构造函数上。现在假设X的用法如下：

    X foo();
    X x;
    // perhaps use x in various ways
    x = foo();
 
 上面最后一行：
  
 * 销毁x掌握的资源
 * 从foo返回来的临时变量里面复制资源
 * 销毁临时变量，并且释放它指向的资源

相对明显看出，直接交换x和临时变量指向的资源指针（句柄），并且让临时变量销毁x原来的资源，这种做法也是正确的，并且更有效。换句话说，在某些特殊场景下，赋值语句的右侧是一个右值，我们按下面的方式实现拷贝赋值操作：

    // [...]
    // swap m_pResource and rhs.m_pResource
    // [...] 
 
 这就叫做**move语句**。在C++11里面，可以通过重载来实现这种假定的行为：

    X& X::operator=(<mystery type> rhs)
    {
      // [...]
      // swap this->m_pResource and rhs.m_pResource
      // [...]  
    }

因为我们重载了拷贝赋值操作，我们的"mystery type"必须在本质上是一个引用：我们一定想让右手边是通过引入的方式传递进来的。而且，我们希望“mystery type”的行为如下：当一个类里有两个重载操作，一个是普通引用，另一个是mystery type，那么右值必须选择mystery type，而且左值必须选择普通引用。

如果现在你把上面的“右值引用”替换成"mystery type"，你就基本看到了右值引用的定义。

右值引用
========
如果X是任意类型，那么X&&被称为X的*右值引用*。为了更好的区分，普通引用X&现在也被叫做*左值引用*。

除了几种特殊情况，一个右值引用的行为非常像普通引用X&。最重要的一点是：但它被用来选择重载函数的时候，左值倾向于选择老格式的左值引用，右值倾向于选择新右值引用。

    void foo(X& x); // lvalue reference overload
    void foo(X&& x); // rvalue reference overload

    X x;
    X foobar();

    foo(x); // argument is lvalue: calls foo(X&)
    foo(foobar()); // argument is rvalue: calls foo(X&&)

因此它的要点是：
>在“我正在被左值调用还是右值？”的情况下，右值引用允许在编译阶段确定函数分支（通过重载函数的确定）。

你可以通过这种方式重载任何函数，就像上面一样。但是在大部分、主要的情况里，这种重载只应该发生在拷贝构造和赋值操作重载，从而实现move语法：

    X& X::operator=(X const & rhs); // classical implementation
    X& X::operator=(X&& rhs)
    {
      // Move semantics: exchange content between this and rhs
      return *this;
    }

为拷贝构造实现一个右值引用也类似。

>**警告**：因为在C++里面这种情况经常发生，第一眼看上去是正确的，但是它离完美还有一些距离。在一些情况下，在实现拷贝赋值符时，简单地交换this和rhs的内容不是很好。我们在在下面的第4节“强制move语句”回来。     

.

>**注意**: 如果你实现：
void foo(X&);
但不是：
void foo(X&&)
那么当然行为是不变的：foo可以被称为左值，但不是右值。如果你实现
void foo(X const &);
但不是
void foo(X&&);
那么还是行为是不变的:foo可以被称为左值和右值，但不可能通过左值和右值来区分它。只能通过实现：
void foo(X&&);
一样。最后，如果你实现：
void foo(X&&);
但不是下面任何一种
void foo(X&);
和
void foo(X const &);
那么，根据C++11的最终版本，foo可以被称为右值，但是如果在左值的地方调用它，就会引起编译错误。

强制move语法
============









 > Written with [StackEdit](https://stackedit.io/).