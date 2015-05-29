
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
就像我们所有人知道的，C++标准曾在第一次修正里面说道：“委员会不应该制定规则去阻止C++程序员做搬起石头砸自己脚的事情。”这种说法不是开玩笑，在选择给程序员更多的控制权，还是从他们自己的粗心大意中挽救他们的时候，C++倾向于更多的控制权这边。基于这种精神，C++11不仅允许你把move语法用在右值上，还允许你用在左值上。一个好的例子是std库里的swap函数。按照之前，假设X是一个类，并且我们重载了拷贝构造函数和拷贝赋值操作去实现右值的move语法。

    template<class T>
    void swap(T& a, T& b) 
    { 
      T tmp(a);
      a = b; 
      b = tmp; 
    } 

    X a, b;
    swap(a, b);

在这儿没有右值。因此，在swap里面的三行没有利用move语法。但是我们知道用move语法会好些：不论什么时候，当一个变量作为拷贝构造或者赋值操作的源出现的时候，那个变量后面不会再被使用，或只会被作为赋值的目标源。

在C++里面，有一个std库函数名叫std::move可以拯救我们。它作为一个函数将它的输入参数转化成一个右值，其它什么事都不做。因此，在C++11里，std库函数swap看起像这样：

    template<class T> 
    void swap(T& a, T& b) 
    { 
      T tmp(std::move(a));
      a = std::move(b); 
      b = std::move(tmp);
    } 

    X a, b;
    swap(a, b);

现在swap里面的三行语句利用了move语法。注意对那些没有实现move语法的类型（也就是说，没有重载他们右值版本的拷贝构造函数和赋值操作），那么新的swap的行为和旧的那个一样。

std::move是一个非常简单的函数。不幸的是，我不能给你列出它的实现。我们会在后面回到这个话题上。

在任何你可以使用std::move的地方使用它，就像上面列出的swap函数那样，会给我们带来如下的重要优势：

 * 对那些事先了move语法的类型，很多标准算法和操作将会使用move语法，并且会带来潜在重要的性能提升。一个重要的例子是原地排序：原地排序算法除了交换元素，不做其他事情，并且这些交换行为将会受益于所有类型提供的move语法。
 * STL经常要求特定类型可拷贝，例如，类型可以被用作容器元素。通过严格的观察，它演变成：在很多情况下，可移动就足够了。因此，在很多地方，我们现在可以使用只具备可以移动，而不具备可拷贝的类型（让我想到unique_pointer），但之前，这是不被允许的。例如，这些类型现在可以被作为STL容器元素。

现在我们已经知道了std::move，我们来看看，为什么实现了右值引用的拷贝赋值符重载以后，就像我之前展示的，还有一些问题。考虑一个变量间简单的赋值，就像：

    a = b; 

你期望这里会发生什么？你期望a里面保存的对象被b的拷贝来代替，并且在替换期间，你期望a之前保存的对象被析构。现在考虑这行：

    a = std::move(b); 

如果move语法是按照一个简单的交换来实现，那么它的副作用就是a和b保存的对象被交换。然后，其他什么东西都没有被析构。a之前保存的对象当然最终会被销毁，换句话说，当b超出它的作用域时。除非，当然，b变成另一个move的目标端，在这种情况下，a之前保存的对象会被再一次传递。因此，只要拷贝赋值运算符被实现，它是不知道a之前保存的对象什么时候会被析构。

因此在某种意义上说，我们已经飘进了不确定析构的阴间：一个变量被赋值，但是这个变量保存的之前的对象仍然在某个地方。只要这个对象对外部世界可见的情况下，没有其他副作用，那就没有什么问题。但有时候那么做，析构是有一些副作用的。一个例子是在析构函数里释放锁。因此，一个对象的析构函数有副作用的话，在拷贝赋值符的右值引用重载里面，它的任何部分应该被显式调用：

    X& X::operator=(X&& rhs)
    {

      // Perform a cleanup that takes care of at least those parts of the
      // destructor that have side effects. Be sure to leave the object
      // in a destructible and assignable state.

      // Move semantics: exchange content between this and rhs
  
      return *this;
    }

一个右值引用是一个右值吗？
========================
按照之前，假设X是一个类，我们重载了它的拷贝构造函数和拷贝赋值符，并且用利用move语法来实现。现在考虑：

    void foo(X&& x)
    {
      X anotherX = x;
      // ...
    }

一个有趣的问题是：在foo函数体内，X的哪个重载构造函数会被调用？这儿，x是一个变量，被声明为一个右值引用，也就是说，一个合适的、典型的（尽管没有必要）引用指向一个右值。因此，它是似乎非常合理的期望：x自己应该也是像一个右值那样被对待，也就是说，

    X(X&& rhs);

应该被调用。换句话说，一个人可能期望：任何东西被声明成右值引用，那么它自己就是右值。右值引用的设计者已经选出解决方案，它比这个要更微妙：

> 一个事物被声明成右值引用，它可以是左值或右值。区分它的关键是：如果它是一个名字，那么他是一个左值。否则，它是一个右值。

在上面的例子里，事物被声明成一个右值引用，有一个名字，因此，它是一个左值：

    void foo(X&& x)
    {
      X anotherX = x; // calls X(X const & rhs)
    }

这儿的例子里，它被声明成一个右值引用，并且没有名字，因此，它是一个右值：

    X&& goo();
    X x = goo(); // calls X(X&& rhs) because the thing on
                 // the right hand side has no name

这里，隐藏在设计后面的逻辑是：允许move语法被隐式的用到有名字的事物上，就像

      X anotherX = x;
      // x is still in scope!

将会变得危险、混乱和易于出错，因为被我们移动过的事物，也就是，我们刚剽窃的事物，在后续的代码中，仍然可以被访问。但是move语法的整个关键点是：只有在它“没有关系”的地方使用它。意思是说：我们移动过的事物已经无效，并且移动之后就离开了。因此，规则是，“如果它有一个名字，那么它就是左值”。



 > Written with [StackEdit](https://stackedit.io/).