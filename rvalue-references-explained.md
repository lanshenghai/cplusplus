
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

那么剩下部分呢？“如果它没有名字，那么它是一个右值？”看上面的goo例子，它是技术上可行的，尽管非常不现实，例子的第二行goo()表达式引用的事物，在它被移动以后仍然可见。但是回顾前面章节：有时这就是我们想要的！我们想基于我们自己的判断来决定是否可以在左值上强制使用move语法，并且精确的规则定义是：“如果它没有名字，那么它是一个右值”，允许我们自己可以控制这种行为。这就是函数std::move的工作原理。尽管有时候它太早地呈现给我们确切的实现方式，我们仅仅对理解std::move靠近了一步。它通过引用传递正确的参数，其它什么事情都不做，并且它的结果类型是一个右值引用。所以表达式

    std::move(x)

被声明成一个右值引用，并且没有名字。因此，它是一个右值。所以，std::move“把它的输入参数转化成右值，即使输出参数不是一个右值”，它是通过“隐藏名字”实现了这个目的。

这儿有一个例子给出了if-it-has-a-name规则是有多么的重要。假设有已经写了一个类Base，并且你已经通过重载Base的拷贝构造函数和赋值运算符实现了move语法：

    Base(Base const & rhs); // non-move semantics
    Base(Base&& rhs); // move semantics

 现在你写一个类Derived，它继承自Base。为了确保move语法会被用到你的Derived对象的Base部分上，你必须同样重载Derived的拷贝构造和赋值运算符。让我们看看拷贝构造函数。拷贝赋值运算符的处理方式类似。左值版本比较直白：

    Derived(Derived const & rhs) 
      : Base(rhs)
    {
      // Derived-specific stuff
    }
 
 右值版本比较大，也比较微妙。一个没有意识到if-it-has-a-name规则的人可能写成：

    Derived(Derived&& rhs) 
      : Base(rhs) // wrong: rhs is an lvalue
    {
      // Derived-specific stuff
    }

如果我们把代码写成这样，那么Base拷贝构造函数的非move语法的版本会被调用，因此rhs有一个名字，它是一个左值。我们想调用的是Base拷贝构造函数的move语法版本，可以把代码写成这样来达到目的：

    Derived(Derived&& rhs) 
      : Base(std::move(rhs)) // good, calls Base(Base&& rhs)
    {
      // Derived-specific stuff
    }
 
move语法和编译优化
==================
考虑下面的函数定义：
    X foo()
    {
      X x;
      // perhaps do something to x
      return x;
    }

现在按照之前的假设，X是一个类，我们重载了它的拷贝构造函数和负值运算符来实现了move语法。如果你按照函数定义的表面来理解，你可能会说，等一等，那儿有一个值拷贝发生在从x到foo的返回值。相应的，让我们确保我们在使用move语法：

    X foo()
    {
      X x;
      // perhaps do something to x
      return std::move(x); // making it worse!
    }

不幸的是，这会让事情变得更糟糕，而不是更好。任何一个现代编译器都是使用*返回值优化*到原始的函数定义上。换句话说，它并不是在本地构造一个X，然后把它拷贝出去，编译器会直接在返回值得地方构造一个X对象。很明显，这个比move语法更好。

因此就像你看到的，为了以优化的方式用到右值引用和move语法，你必须完全了解并考虑今天编译器的“副作用”，就像返回值优化和节省拷贝。在Scott Meyers的“Effective Modern C++”一书的第25和41条例有详细的讨论。它变得很微妙，但是，嗨，我们选择C++作为我们的语言是有原因的，对吗？我们制造自己的床，所以，现在让我们躺在上面。

完美传递：问题
================
move语法的另一个问题是右值引用是被设计用来解决完美的传递问题。考虑下面简单的工厂函数：

    template<typename T, typename Arg> 
    shared_ptr<T> factory(Arg arg)
    { 
      return shared_ptr<T>(new T(arg));
    } 
 
 很明显，这儿的意思是从工厂方法里传递参数arg到T的构造函数。理想情况下，一切事情应该就像工厂函数不存在那样，拷贝构造在客户代码的地方被直接调用：完美传递。可悲的是上面的代码会失败在：它通过值引入了一个额外的调用，如果构造函数参数是一个引用，它就会变得很糟糕。

对常见的解决方案是，像boost::bing的那样，让外面的函数以引用方式接受参数：

    template<typename T, typename Arg> 
    shared_ptr<T> factory(Arg& arg)
    { 
      return shared_ptr<T>(new T(arg));
    } 

这会好些，但是不完美。问题是：现在，工厂函数不能被以右值方式调用：

    factory<X>(hoo()); // error if hoo returns by value
    factory<X>(41); // error

这个问题可以通过提供常量引用作为参数的重载来解决：

    template<typename T, typename Arg> 
    shared_ptr<T> factory(Arg const & arg)
    { 
      return shared_ptr<T>(new T(arg));
    } 

这个方法有两个问题。第一，如果factory不是只有一个参数，而是几个，你必须提供所有非常量和常量引用的组合作为参数的重载。因此，对于有几个参数的函数，这个方案变得非常糟糕。

第二，这种传递变得不那么完美是因为他阻碍了使用move语法：在factory函数体内，T的构造函数参数是一个左值。因此，move语法永远不会发生，即使没有那个包装函数。

右值引用可以被用来解决这些问题。在不需要使用重载的情况下，它可以达到完美的传递。为了理解原理，我们需要看一下另外两条右值引用的规则.

完美传递：解决方案
=================
剩余两条右值引用规则的第一条也会影响老格式的左值引用。回忆一下在C++11之前，引用的引用是不被允许的：优势像A& &会导致一个编译错误。相反，C++11引入了下面的引用塌陷原则[^1]：

 * A& &变成A&
 * A& &&变成A&
 * A&& &变成A&
 * A&& &&变成A&

[^1]: 这儿描述的引用塌陷原则是正确的，但不是完整的。我忽略了一个和我们上下文无关的小细节，也就是说，在引用塌陷时，const和volatile修饰符的消失。在Scott Meyer's "Effective Modern C++"的modification history and errata页面有完整解释。

第二，对一个把右值引用作为模板参数的函数模板，有一个特殊的模板参数推演原则：

    template<typename T>
    void foo(T&&);

适用于下面情况：

 1. 当foo以A类型的左值被调用时，T会被解释成A&，并且，参考上面的引用塌陷原则，参数类型有效的变成了A&。 
 2. 当foo以A类型的右值被调用时，T会被解释成A，并且，参数类型变成了A&&

按照这些原则，我们现在可以使用右值引用来解决完美传递问题，就像前面章节提到的。解决方案看起来像这样：

    template<typename T, typename Arg> 
    shared_ptr<T> factory(Arg&& arg)
    { 
      return shared_ptr<T>(new T(std::forward<Arg>(arg)));
    } 
    where std::forward is defined as follows:
    template<class S>
    S&& forward(typename remove_reference<S>::type& a) noexcept
    {
      return static_cast<S&&>(a);
    } 
 
 （现在不用注意noexcept关键字。它让编译器知道，对于特定优化目的，这个函数永远不会抛出异常。我们在第9章会回到这点。）为了看到上面的代码怎么实现完美传递，我们会分开讨论当我们的工厂函数分别被左值调用和右值调用时会发生什么情况。设A和X是类型。假设factory<A\>会被类型X的左值方式调用：

    X x;
    factory<A>(x);

然后，通过上面特殊模板推演原则，factory的模板参数Arg被解析成X&。因此，编译器把factory和std::forward实例化成下面的样子：

    shared_ptr<A> factory(X& && arg)
    { 
      return shared_ptr<A>(new A(std::forward<X&>(arg)));
    } 

    X& && forward(remove_reference<X&>::type& a) noexcept
    {
      return static_cast<X& &&>(a);
    } 

在解析remove_reference之后和应用完引用塌陷原则，会变成：

    shared_ptr<A> factory(X& arg)
    { 
      return shared_ptr<A>(new A(std::forward<X&>(arg)));
    } 

    X& std::forward(X& a) 
    {
      return static_cast<X&>(a);
    } 

这一定可以完成左值的完美传递：工厂函数的参数arg通过两级间接地传递给A构造函数，都是老风格的左值引用。

下一步，通过上面的特殊模板推演原则，factory的模板参数Arg解析成X。因此，编译器现在会创建下面的函数模版实例：

    shared_ptr<A> factory(X&& arg)
    { 
      return shared_ptr<A>(new A(std::forward<X>(arg)));
    } 

    X&& forward(X& a) noexcept
    {
      return static_cast<X&&>(a);
    } 

这简直是一个右值的完美传递：工厂函数的参数通过两级间接地传递给A的构造函数，都是引用。此为，A的构造函数会看到它的参数是一个被声明成右值引用的表达式，并且没有名字。通过no-name原则，这个事物是一个右值。因此，A的构造函数通过右值被调用。这意思是相比不提供工厂包装，这个传递节省了很多move语法。

它可能什么都不值，保留move语法实际上只是为了std::forward的上下文。没有使用std::forward的话，一切都工作的非常正常，除了A的构造函数总是看到它的参数有一个名字，并且它是一个左值。另一种解释是std::forward的目的是传递信息，不论在什么调用地方，包装器总是看到一个左值或右值。

如果你为了更多信心想去挖的更深，问你自己一个问题：为什么需要remove_reference在std::forward的定义里面？答案是，它根本不是真正必须的。在std::forward的定义里，如果你仅仅用S&代替remove_reference<S\>::type&，你能重复上面的情况来说服自己完美传递仍然工作。然而，它完美工作的前提是我们显式的特殊化Arg为std::forward的一个模板参数。在std::forward定义里，remove_reference的目的是强制我们那么做。

庆祝一下，我们基本快研究完毕了。只剩下查看std::move的实现了。记住，std::move的目的是把一个参数通过引用传递给它，然后绑定成一个右值。这儿是它的实现：

    template<class T> 
    typename remove_reference<T>::type&&
    std::move(T&& a) noexcept
    {
      typedef typename remove_reference<T>::type&& RvalRef;
      return static_cast<RvalRef>(a);
    } 

假定我们用类型X的左值调用std::move：

    X x;
    std::move(x);

通过新的特殊模板推演原则，模板参数T会被解析成X&。因此，编译器会实例化成：

    typename remove_reference<X&>::type&&
    std::move(X& && a) noexcept
    {
      typedef typename remove_reference<X&>::type&& RvalRef;
      return static_cast<RvalRef>(a);
    } 

在解析完remove_reference和应用新的引用塌陷原则之后，这个变成：

    X&& std::move(X& a) noexcept
    {
      return static_cast<X&&>(a);
    } 

完成了功能：我们的左值x会被绑定到左值引用上，它是参数类型，并且函数透传它，把它变成了一个没有名字右值引用。

我把怎么说服你自己关于std::move在右值的时候工作也正常留给你。当你可以跳过：为什么一些人想在右值上调用std::move，但他们目的仅仅是把事物转成右值？另外，到现在，你可能注意到除了：

    std::move(x);

你也可以写成：

static_cast<X&&>(x);

然而，std::move还是强烈推荐，因为它非常有表现力。



右值引用和异常
==============
一般性，当你用C++开发软件的时候，你可以选择是否想对异常的安全性特殊留意，或者完全不管。在这点儿上，右值引用有一点点麻烦。当你为了使用move语法，重载一个类的拷贝构造函数和拷贝赋值符时，非常建议你做到如下几点：

 1. 努力让重载函数不要抛出异常。如果那样做得话就非常诡异，因为move语法一般来说除了交换指针或资源句柄，其他什么都不做。
 2. 如果你已经没有让你的重载函数抛出异常，那么通过新的noexcept关键字确保没有。

如果你做不到这两点，那么说明至少有一种情况下你的move语法不合适，尽管你期望可以：当修改std::vector的尺寸时，它里面已经存在的元素需要被重新分配到新的内存块中，这时你确实想使用move语法。但是它不会发生，除非做到上面的两条要求。

你真的不需理解它背后的原因。遵守这两条建议也足够的非常，而且这也是真正需要知道的所有内容。然而，相比知道这些必须了解的，理解的更加深入一些绝对不会对任何人有伤害。我建议读Scott Meyers的书“Effective Modern C++.”的第14条（其实，还有所有其它条）。

隐式move的情况
==============
在讨论右值引用（经常是复杂和有争议的）的时候有一点，标准委员会决定move构造函数和move赋值运算符，也就是，拷贝构造函数和拷贝赋值运算符右值引用重载，当用户没有提供的时候，应该被编译器自动产生。这看起来像一个自然、合理的要，考虑编译器总是为普通的拷贝构造函数和拷贝赋值运算做同样的事情。在2010年8月，Scott Meyers在[comp.lang.c++上发的一个帖子](http://groups.google.com/group/comp.lang.c++/browse_thread/thread/d51d6282f098b3ca)，在里面，他解释了编译器自动产生的move构造函数是怎样破坏以一种非常严重的方式已经存在的代码。

委员会然后决定这应该引起注意，并且在某种情况下它限制了自动产生move构造函数和move赋值运算，虽然在那种情况下不太可能（但不是不可能）破坏已有的代码。这件事情的最后结果在Scott Meyers的书“Effective Modern C++”的第9条有详细的描述。

直到标准的最后定稿，隐式moving这件事一直有争议（参见Dave Abrahams的[论文](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2010/n3153.htm)）。一个讽刺而扭曲的命运是，委员会为什么考虑把隐式move放到第一位的原因仅仅是：他们试图解决在第9部分提到的右值引用和异常问题。这个问题随后通过新的关键字noexcept被满意解决。如果不是noexcept解决方案被提前发现仅仅几个月，隐式move可能永无出头之路。好吧，它出来了。

好了，那就是整个右值应用的完整故事。就像你看到的，增益是可观的。细节是残酷的。作为一个C++专家，你可能必须去理解这些细节。否则，你必须放弃全面理解你职业里面最核心的工具。虽然，你可以找一些安慰，想到在你日复一日的编程中，你可能仅仅必须记住关于右值引用的三件事：

 1. 通过重载一个函数，像这样：

    void foo(X& x); // lvalue reference overload
    void foo(X&& x); // rvalue reference overload

    你可能在编译的时候遇到“foo正在被按照左值方式还是右值方式调用？”。最主要（从使用角度来看是仅仅）的应用场景是重载一个类的拷贝构造函数和考虑赋值运算，来实现move语法。如果并且你那么做了，请确保注意异常的处理，并且尽最大可能地用新关键词noexecpt。

  2. std::move把它的入参转化为右值。
  3. std::forward允许你获得完美传递，前提是你完全按照第8节里工厂函数例子里的用法。

享受它吧！

鸣谢和进一步阅读
================
我非常感激Thomas Witt对右值引用的洞察和信息，以及分享给我。谢谢Valentin David的仔细阅读和提供有价值的改动和思考。很多读者帮助提升这篇文章。感谢每一位贡献者，并且请继续发送反馈。所有剩余的不准确和缺陷给我。

我非常推荐阅读在C++ Source里关于C++右值引用的[Howard E. Hinnant，Bjarne Stroustrup和Bronek Kozicki的文章](http://www.artima.com/cppsource/rvalue.html)。这篇文章有非常多和不错的例子，并且它还有指向提案和技术论文的链接列表，你会发现一些有趣的内容。作为折中，文章没有深入到每个细节；例如，它没有显式描述右值引用中的新的引用塌陷原则或特殊模板参数推演原则。

最后，最主要的C++11和C++14的参考源代码应该在Scott Meyers的书“Effective Modern C++.”里。同时我在这篇文章里展现的让你更加关注于右值引用，Scott Meyers的书描绘了整个现代C++的全貌，在细节上没有遗漏任何一个角落。




