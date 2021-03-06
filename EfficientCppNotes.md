# Effective C++ 摘要




## Chapter 1 Accustoming yourself to C++


### Item 1: View C++ as a federation of languages

C++是个多重范型编程语言（multiparadigm programming language），支持过程形式（procedural），面向对象形式（object-oriented），函数形式（functional），泛型形式（generic），元编程形式（metaprogramming）

C++中的主要次语言（sublanguage）有四个：C；object-oriented C++；template C++；STL


### Item 2: Prefer consts, enums, and inlines to #defines

宏#define被预处理器处理，不进入符号表

尽量使用std::string而非char*-based

以常量替换宏

class专属的static的整数类型常量，如果不取他们的地址且不需要看到他们的定义式，可以声明并使用而无需提供定义式

在声明时获得初值的class常量，不能在定义时再设置初值

宏#define不重视作用域，因此不能用来定义class专属常量

In-class初值设定只能对整数常量进行，如果不支持整数常量的in-class初值设定，则可以使用“the enum hack”

```C++
class GamePlayer {
private:
	enum { NumTurns = 5 };
	int scores[NumTurns];
};
```

Enum hack中，不能对该enum取地址，类似#define

使用template inline函数替代形似函数的宏


### Item 3: Use const whenever possible

const可以用于：class外部修饰global或namespace作用域中的常量，文件、函数、区块作用域中被声明为static的对象，class内部的static和non-static成员变量，指针自身或指针所指物等

迭代器表现类似于指针

```C++
const std::vector<int>::iterator iter; // 类似于T* const
std::vector<int>::const_iterator cIter; // 类似于const T*或T const *
```

Const成员函数确认该成员可以作用于const对象身上

只有常量性不同的两个成员函数可以被重载

const成员函数的意义有两种：

	1. Bitwise constness（physical constness），const成员函数不改变对象内任何non-static成员变量，即不更改对象的任何bit；好处是容易侦测违反点；C++对常量性的定义即为bitwise constness；但是对于持有指针的对象来说，bitwise constness并不能保证指针指向的内容的constness

```C++
class CTextBlock {
public:
	char & operator[] (std::size_t position) const { // bitwise const，即pText指向内容不是const
		return pText[position];
	}
	
	char * const & getText() const { // bitwise const，即pText为const，但是指向内容不是const
		return pText;
	}
	
private:
	char * pText;
};
```

	2. Logical constness，const成员函数可以修改对象内部某些bit，只要客户端侦测不出即可；可以使用mutable（effective modern C++ item 16）
	
对于const和non-const成员函数中的重复，使用non-const成员函数调用其const版本，其中需要借助static_const和const_cast

```C++
class TextBlock {
public:
	const char & operator[] (std::size_t position) const;
	
	char & operator[] (std::size_t position) {
		return const_cast<char&>(static_cast<const TextBlock&>(*this)[position]);
	}
};
```


### Item 4: Make sure that objects are initialized before they're used

区分初始化和赋值

总是在member initialization list中初始化类的成员变量，（有一个规则是：列出所有成员变量）；成员变量为const或者reference的，只能初始化，不能被赋值

成员初始化顺序：base class更早于其derived class，成员变量总以类中的成员变量的声明顺序被初始化，而非member initialization list中的出现顺序，最好总是成员初值列最好依照声明次序

编译单元（translation unit）指产出单一目标文件（single object file）的源码，基本上是单一源码文件加上其所包含的头文件

C++对于不同编译单元内的non-local static对象的初始化次序没有明确定义（当多个编译单元内的non-local static对象经由模板隐式实例化（implicit template instantiation）形成是，往往很难寻找正确的初始化次序）

```C++
// FileSystem.h
extern FileSystem tfs; // non-local static对象

// Directory.h
class Directory {
public:
	Directory() {
		std::size_t disks = tfs.numDisks();
	}
};
Directory tempDir(); // non-local static对象，用到别的编译单元内的non-local static对象tfs，可能tfs还未初始化
```

应该使用reference-returning函数修改为（Singleton模式）

```C++
// FileSystem.h
FileSystem& tfs() {
	static FileSystem fs;
	return fs;
}
	
// Directory.h
class Directory {
public:
	Directory() {
		std::size_t disks = tfs().numDisks();
	}
};
Directory& tempDir() {
	static Directory td;
	return td;
}
```

C++保证对于函数内的local static对象会在该函数被调用期间首次遇上该对象的定义式的时候被初始化，因此对于不同编译单元内的non-local static对象，将其移入自己的专属函数内，并声明为static，这样变成了一个local static对象，保证能够获得经历初始化的对象

任何non-const static对象（不论local还是non-local）在多线程情况下都会有问题

多线程环境下往往在单线程启动阶段手工调用所有reference-returning函数，然后启动多线程








## Chapter 2 Constructors, destructors, and assignment operators


### Item 5: Know what functions C++ silently writes and calls

class的base class自身声明有virtual析构函数是，编译器产生的析构函数也为virtual的，virtualness来自base class；否则编译器产生的析构函数是non-virtual的

编译器生成的copy构造函数和copy assignment函数，只是将对象的每一个non-static成员变量拷贝到目标对象

当类含有reference成员，或者内含const成员，或者他的base class将copy assignment声明为private，编译器不会为这个类生成一个copy assignment


### Item 7: Declare destructors virtual in polymorphic base classes

一般情况下，只有当class内含至少一个virtual函数时，才为他声明virtual析构函数

带有pure virtual析构函数的类是抽象类，往往被当做base class使用，因此当derived class对象被析构时需要调用base class的析构函数，因此需要给pure virtual析构函数提供一个定义，即

```C++
class AWOV {
public:
	virtual ~AWOV() = 0; // pure virtual析构函数的声明
};
AWOV::~AWOV() {} // pure virtual析构函数的定义
```

为base class声明一个virtual析构函数只适用于polymorphic base class，即多态性质的基类，为了用来通过base class接口处理derived class对象

Uncopyable和input_iterator_tag等被设计作为base class，但并不为了经由base class接口处理derived class对象，因此不需要virtual析构函数

std::string等并不应该被作为base class使用


### Item 8: Prevent exceptions from leaving destructors

C++11中析构函数默认noexcept

如果某个操作可能在失败时抛出异常，又必须处理这个异常，让这个异常来自于析构函数以外的普通函数


### Item 9: Never call virtual functions during construction or destruction

在derived class对象的base class对象构造期间，对象的类型是base class而derived class；在析构过程中的base class析构期间，对象的类型也是base class，对virtual函数的调用不会下降至derived class，即不会调用客户端期望调用的virtual函数版本（derived class版本）


### Item 10: Have assignment operators return a reference to *this

为了实现连锁赋值


### Item 11: Handle assignment to self in operator=

使用简单的证同测试能够具备自我赋值安全性，但是不能保证异常安全性

保证异常安全性往往自动获得自我赋值安全性，因此保证异常安全性的情况下往往不去管自我赋值安全性（如果添加证同测试，会使代码变大，并降低执行速度，并且带来的效率提高由于自我赋值频率很低往往得不偿失）

使用copy and swap来代替手工排列语句实现异常安全

```C++
Widget& Widget::operator= (const Widget& rhs) {
	Widget tmp(rhs); // copy
	swap(temp); // swap
	return temp;
}
```


### Item 12: Copy all parts of an object

让derived class的copy函数调用相应的base class函数 

当copy构造函数和copy赋值函数有重复代码时，应当将重复代码放入第三个函数中，然后让这两者共同调用；因为copy构造函数只能用于初始化对象，而copy赋值函数只能施行于已初始化对象






## Chapter 3 Resource Management



### Item 13: Use objects to manage resources

获得资源立即放入管理对象内，资源取得时机就是初始化时机（RAII，Resource Acquisition Is Initialization）

管理对象运动析构函数确保资源被释放

使用shared_ptr，不要用auto_ptr

并没有针对动态数组设计的shared_ptr，使用vector、string替换动态数组



### Item 14: Think carefully about copying behavior in resource-managing classes

资源的复制行为决定RAII对象的复制行为

RAII对象的复制处理通常使用##禁止复制##或者##对底层资源使用引用计数法（reference-count）##实现

对底层资源使用引用计数法（reference-count）可以使用shared_ptr成员变量实现，并且对于Mutex这类资源，可以定制其引用计数为0时的行为（以unlock为删除器）；析构函数不需要涉及shared_ptr成员变量，资源的释放行为已经设置为shared_ptr的删除器，并且在引用计数为0时自动调用了删除器执行资源释放行为

RAII对象的复制还可以采用##复制底部资源##或者##转移底部资源的拥有权##的方式实现



### Item 15: Provide access to raw resources in resource-managing classes

将RAII对象转换为内含的原始资源有显式转换和隐式转换两种方式

shared_ptr提供get成员函数实行显式转换

shared_ptr重载了指针取值（pointer dereferencing）操作符（operator->和operator*），允许隐式转换为原始指针

RAII设计者还可以提供一个隐式类型转换函数直接转换为内含的原始资源类型，但是容易被误用

```C++
class Font {
public:
	explicit Font(FontHandle fh) : f(fh) {} // 获得资源FontHandle
	~Font() { releaseFont(f); }
	
	FontHandle get() const { return f; } // 显式转换函数
	operator FontHandle() const { return f; } // 隐式转换函数
};
```



### Item 16: Use the same form in corresponding uses of new and delete

new返回的出来的单一对象的内存布局不同于数组的内存布局，数组所用的内存通常还包括数组大小的记录

使用delete[]使得delete认定指针指向一个数组

成对使用new和delete，以及new[]和delete[]

不要对数组形式使用typedef，容易忘记使用delete[]


### Item 17: Store newed objects in smart pointers in standalone statements

Effective modern C++ Item 21








## Chapter 4 Designs and Declarations


### Item 18: Make interfaces easy to use correctly and hard to be incorrectly

Shared_ptr自动使用每个指针专属的删除器，防止cross-DLL problem（对象在DLL中被new创建，却在另一个DLL中被delete销毁），缺省的删除器来自shared_ptr诞生所在的那个DLL中的delete


### Item 19: Treat class design as type design

新的type对象应该如何被创建和销毁

对象的初始化和对象的赋值该有什么样的差别

新type的对象如果被passed-by-value意味着什么

什么是新type的合法值

新type需要配合某个继承图系么

新type需要什么样的转换

什么样的操作符和函数对此新type而言是合理的

什么样的标准函数应该驳回

谁取去用新type的成员

什么事新type的未声明接口

新type有多么一般化

真的需要一个新type么


### Item 20: Prefer pass-by-reference-to-const to pass-by-value

缺省情况下c++以by value方式（继承自C）传递对象

以by reference方式传递参数可以避免对象切割（slicing）问题（derived class对象以by value方式传递并被视作一个base class时， 调用base class的拷贝构造函数，新对象丢失了derive部分的所有信息）

C++编译器往往底层使用指针实现reference，因此对于内置类型，往往pass by value比pass by reference to const效率高

STL的迭代器和函数对象习惯上被设计为pass by value



### Item 21: Don't try to return a reference when you must return an object

返回reference to local对象会出现悬空引用问题

不要让函数返回一个指针或引用指向local stack对象，也不要返回一个引用指向一个heap-allocated对象

合理的返回指向local static对象的引用的例子是item4中的singleton模式

一个必须返回新对象的函数的正确写法就是让函数返回一个新对象



### Item 22: Declare data members private

将成员变量隐藏在函数接口背后可以为所有可能的实现提供弹性，相当于Delphi和C#中的properties

protected成员封装性并不高于public（item23）

封装性与当其内容改变时可能造成的代码破坏量成反比



### Item 23: Prefer non-member non-friend functions to member functions


封装性越好，越多东西被封装，则越少人可以看到他，反之亦然

成员变量应该为private，否则就会有无穷多的函数可以访问，带来零封装性

对于一个类，要在提供相同功能的member函数和non-member non-friend函数中做取舍，选择较大封装性的non-member non-friend函数，他不会增加能访问class内部private成员的函数数量，不会降低封装性（这里non-member函数不意味着一定要是非成员函数，而是指针对这个类而言，不是这个类的成员函数；这个non-member函数也可以是别的类的函数，例如实现大量遍历功能函数的util类的一个功能函数）

```C++
Class WebBrowser {
Public:
	Void clearCache();
	Void clearHistory();
	Void removeCookies();
	
	Void clearEverything() { // member函数版本
		clearCache();
		clearHistory();
		removeCookies();
	}
};

Void clearBrowser(WebBrowser & wb) { // non-member non-friend版本
	wb.clearCache();
	wb.clearHistory();
	wb.removeCookies();
}
```

对于一个类以及相关的便利函数，可以将这些便利函数按照功能放入多个头文件内，隶属于同一个名空间中，方便客户轻松扩展新的便利函数，这是在类中使用member函数无法做到的，class定义式对客户而言无法扩展，C++标准库即是按照这种组织形式


### Item 24: Declare non-member functions when type conversions should apply to all parameters

通常让class支持隐式类型转换不是好方案

对于支持和整数进行乘法运算的有理数类

```C++
class Rational {
public:
	Rational(int numerator = 0, int denominator = 1); // 支持整数到有理数类的类型转换
	const Rational operator* (const Rational& rhs) const; // 不支持2 * rational
};

const Rational operator* (const Rational& lhs, const Raional& rhs); // 支持2 * rational
```


### Item 25: Consider support for a non-throwing swap

对于自行提供的swap版本（应用于Impl手法实现的类等场景），需要：

	1. 提供一个public swap成员函数真正执行置换
	2. 在类或类模板所在名空间内提供一个non-member swap，令其调用上述public swap成员函数
	3. 如果是类而非类模板，还要为类特化std::swap，并让特化版本调用swap成员函数（这样能让专属版swap能在尽可能多的语境下被调用），通常不允许改变std名空间内的任何东西，但被允许为标准template制造特化版本
第2点对于类模板应该在所在名空间中使用重载swap方式提供non-member swap

```C++
namespace WidgetStuff {
	template<class T>
	class Widget;

	template<typename T> 
	void swap(Widget<T> & a, Widget<T> & b) { a.swap(b); } // 对std::swap的重载，swap后没有<…>
}
```

第3点不能应用于类模板，特化std::swap会导致对函数模板的偏特化

```C++
Template<class T>
class Widget;

namespace std {
	template<typename T> 
	void swap<Widget<T>>(Widget & a, Widget & b) { a.swap(b); } // 错误，函数模板不允许偏特化
}
```


客户在调用swap时，先`using std::swap;`使得std内的swap可见，再直接调用没有任何名空间修饰的swap，让编译器根据C++的名称查找法则决定调用哪一个swap，加上名空间修饰符的swap会影响编译器的名称查找

成员版swap是noexcept的，是帮助提供强异常安全性的保证（item29）；非成员版swap可能抛出异常，因为默认情况下会调用copy构造函数和copy赋值函数

自定义的swap往往是对内置类型的操作，因此需要尽量提供高效置换的方法，并且尽量不抛出异常









## Chapter 5 Implementations


### Item 26: Postpone variable definitions as long as possible

尽量延后变量的定义，避免构造和析构不必要的对象

尝试延后变量的定义直到能够给他初值实参为止，避免无意义的default构造行为，而是直接使用给出的构造参数进行构造，顺带起到说明构造对象的信息的目的

循环中使用的变量，放到内可能会有构造和析构的开销，放到外部会使变量的作用域扩大降低可维护性，需要权衡


### Item 27: Minimize casting

C++风格转型

const_cast<T>(expr)
dynamic_cast<T>(expr)
reinterpret_cast<T>(expr)
static_cast<T>(expr)

相比旧式转型，新式转型容易辨识，并且将转型动作目标

尽量使用新式转型

类型转换（转型操作的显示转换或者编译器完成的隐式转换）往往会零编译器编译出运行期的执行码，例如int转型为double会由于底层表述不同，derived*转型为base*可能会由于两个指针值的不同在指针上施行偏移量

唯一使用旧式转型的时机是调用explicit构造函数将对象传递给函数

在dereived class的virtual函数中调用base class的对应函数，需要调用base class名称修饰的函数；而不能对*this进行static_cast类型转换，这个类型转换会创造一个当前对象的base class成分的副本对象

替代dynamic_cast（即不使用base class的指针或引用转型成derived class的指针或引用，意图调用只有derived class才会具有的能力）的方式：

	1. 直接存储指向derived class对象的指针或引用
	2. 通过base class接口处理所有可能的各种派生类，即在base class中提供virtual函数实现所有派生类可能拥有的功能，对于不具有该功能的base class，提供一份空的virtual函数实现

必须要转型，则将其隐藏于某个函数之后，而非让转型出现在客户代码之中


### Item 28: Avoid returning "handles" to object internals

Item3中的例子，成员的封装性最多只等于返回其reference的函数的访问级别，如果const成员函数传出一个reference，后者并不存储于对象中（不是对象的成员变量，但是对象的成员变量指向他，并且逻辑上视作对象的内部）

返回一个handle代表对象的内部部分往往会导致悬空指针或引用，但是有时不得不避免这类问题，例如operator[]函数等需要存取容器内部的元素


### Item 29: Strive for exception-safe code

异常安全的函数会：不泄露资源；不允许数据被破坏

异常安全函数提供三个保证之一：

	1. 基本承诺，如果抛出异常，程序内任何事物仍然有效状态下，没有对象或者数据结构因此破坏，但是程序的现实状态不可以聊
	2. 强烈保证，如果抛出异常，程序状态不改变，即如果函数失败，程序回到调用前的状态
	3. nothrow保证，即从不抛出异常

Copy and swap：为要修改的对象制作副本，然后在副本上做修改，待一切修改成功后才将副本和原对象使用nothrow的swap操作置换

Copy and swap往往可以实现强安全保证

函数f依次调用了f1和f2，即使f1和f2都是强类型安全的，也不能保证f是强类型安全的，如果f1成功而f2失败，则状态永远退回不到f1被调用之前的状态


### Item 30: Understand the ins and outs of inlining

inline是对编译器的申请

函数定义与class内部是一种隐式inline，通常是成员函数，也可能是friend函数

一般情况下inline和template都置于头文件中，但是template的实例化和inline无关；inline通常是编译期行为，inline通常一定被置于头文件内，编译器为了进行函数调用需要知道函数的定义；template通常被置于头文件内，一旦被使用，编译器为了实例化需要知道他的定义，通常实例化是编译期行为

virtual函数往往不会被inline，需要运行期决定调用那个函数

取inline函数的地址往往不会被inline，而是会生成一个outlined函数本体

编译器往往会在编译期间产生对象创建和销毁相关的代码并插入到构造函数和析构函数中，因此构造函数和析构函数往往不会被inline

inline无法随着程序库的升级而升级，一旦f被改变需要重新编译，而non-inline函数只要重新连接即可



### Item 31: Minimize compilation dependencies between files

C++没有做好“接口从实现中分离”，class的定义式不仅包含接口（成员函数）还包含实现细目（成员变量）；编译一个class需要取得其所有成员变量类型的定义式（需要在编译期间决定对象的大小），因此需要include所有定义式所在的头文件，带来文件之间的依赖性

C++可以通过pimpl手法，通过成员指针指向实现细节，实现实现细节和接口分离；以声明的依存性替换定义的依存性；类似java等语言，成员变量均是引用而非值

编译依存性最小化的本质：让头文件尽可能自我满足，如果不能满足就让他与其他文件内的声明式（而非定义式）相依

编译依存性最小化：

	1. 如果使用object reference或object pointer可以就不要使用object
	2. 尽量以class声明式代替class定义式 ???
	3. 为声明式和定义式提供不同的头文件，但是需要保持一致性 iosfwd ???

使用pimpl idiom的class通常称为handle class，将所有函数转交给相应的实现类

interface class也可以将声明式和定义式分离，是一种特殊的abstract class，通常不带成员变量，没有构造函数，只有一个virtual析构函数（item7）和一组pure virtual函数，用于描述整个接口；interface class通常使用factory函数（item13）或virtual构造函数？？？构造真正的derived class对象

程序库头文件应该以完全且仅有声明式的形式存在，不论是否涉及template都适用







## Chapter 6 Inheritances and Oriented-Object Designs


### Item 32: Make sure public inheritance models "is-a"

public继承以为这is-a关系，B对象可用的任何地方，D对象也可以，D对象也是一个B对象



### Item 33: Avoid hiding inherited names

当derived class成员函数内refer to base class中的内容，编译器将derived class作用于嵌套在base class作用域内，进行名称查找

作用域名称遮掩规则，base class和derived class中函数名相同，即使他们参数类型不同，derived class函数也会遮掩base class函数，原理是为了防止新的derived class从base class继承重载函数

使用using声明式继承base class中被遮掩的重载函数，using会使得base class中所有同名函数都可见（不同参数，不同constness等）

public继承中不要让derived class内的名称遮掩base class中的名称（如果derived class重写了base class中的函数需要使用using声明式继承base class中被覆盖的函数，不然会违反item22中is-a的关系，即derived class不再拥有base class被覆盖的特性）

private继承中，如果仅想继承一个特定的base class重载函数，定义一个名称相同的derived class函数，并令其调用base class中想要继承的那个函数（使用using会使得所有同名函数都可见），称为forwarding函数



### Item 34: Differentiate between inheritance of interface and inheritance of implementation

public继承由函数接口继承和函数实现继承两部分组成

可以为prue virual函数提供定义，但是只能通过明确指出其class名称的方式被调用

impure virtual函数提供了一份函数的缺省实现，声明impure virtual函数的目的是让derived class继承其接口和缺省实现

可以使用“pure virtual函数替代并给出一个pure virtual函数的定义或给出一个普通的进行缺省行为的函数的定义“替代impure virtual函数，并且这种替代方式切断了virtual函数接口和其缺省实现的连接；如果提供pure virtual函数提供定义则不能单独对其实行访问控制；如果提供一份进行缺省行为的函数的定义，则带来更多的class命名空间的污染

+ 声明一个pure virtual函数是为了让derived class只继承函数接口
+ 声明一个impure virtual函数是为了让derived class继承函数的接口及一份缺省实现
+ 声明一个non-virtual函数是为了让derived class继承函数的接口及一份强制性实现，non-virtual函数绝不该在derived class中被重新定义



### Item 35: Consider alternatives to virtual functions

derived class可以重新定义virtual函数，即使derived class无法调用他

使用non-virtual interface手法实现template method模式（与C++template无关），即non-virtual函数作为一个接口（称为wrapper）不变，调用实际进行工作的private（或protected） virtual函数

函数指针或者tr1::function实现的strategy模式，同一类型不同对象可以有不同的做法，并且可以在运行期替换

传统的strategy模式，将对应功能的函数做成一个分离的继承体系中的virtual成员函数

将实现机能的部分作为外部函数或仿函数则不能访问non-public成员



### Item 36: Never redefine an inherited non-virtual function

如果在derived class中重新定义base class中的non-virtual函数，由于静态绑定，使得d->f()和((Base *)d)->f()调用结果不同

不应该重新定义一个继承而来的non-virtual函数



### Item 37: Never redefine a function's inherited default parameter value

virtual函数是动态绑定的，但是缺省函数是静态绑定的

运行期效率是的C++采用了缺省参数值的静态绑定

可以使用item35中NVI手法，让non-virtual函数指定缺省参数



### Item 38: Model "has-a" or "is-implemented-in-terms-of" through composition

在应用域，复合意味着”has-a”，在实现域，复合意味着“is-implemented-in-terms-of”


### Item 39: Use private inheritance judiciously

只有在public继承中，编译器会自动将一个derived class对象转换成一个一个base class对象（reference或指针转换）

private继承而来的base class中的所有成员都是private的

private继承意味着implemented-in-terms-of，意味着只继承实现，不继承接口，只是一种实现技术而非设计意义（继承而来的每样东西都是private，都是实现细节），为了采用base class中已经备妥的某些特性而非base class对象和derived class对象存在任何观念上的关系

对于is-implemented-in-terms-of，尽量使用composition（item38），必要时才使用private继承

使用private继承的一个情形是，在对空间利用非常激进的时候，对于一个不带任何non-static成员变量和virtual函数的类，使用private继承会获得空白基类优化，使用composition会由于C++对独立对象有非零大小的要求，有稍稍的空间浪费

当derived class需要访问base class的protected成员，或者需要重新定义继承而来的virtual函数时，private继承是合理的（但是总是与设计无关，只是需要使用实现继承）


### Item 40: Use multiple inheritance judiciously

钻石型继承`File <- InputFile / OutputFile <- IOFile`

对于File类中的fileName成员，IOFile的缺省做法是从InputFile和OutputFile中分别继承fileName；如果使得File称为虚基类函数，即`File <-{virtual}- InputFile / OutputFile <- IOFile`，那么IOFile中只含有一个fileName

正确行为来看，public继承应该总是virtual的

virtual继承会使体积比non-virtual继承的体积大；virtual base class的初始化规则很复杂。因此非必要不使用virtual继承；如果必须使用virtual继承，则尽量避免在virtual base class中放置数据，避免初始化和赋值的问题（类似java和C#中的interface）







## Chapter 6 Inheritances and Oriented-Object Designs


### Item 40: Use multiple inheritance judiciously

钻石型继承`File <- InputFile / OutputFile <- IOFile`

对于File类中的fileName成员，IOFile的缺省做法是从InputFile和OutputFile中分别继承fileName；如果使得File称为虚基类函数，即`File <-{virtual}- InputFile / OutputFile <- IOFile`，那么IOFile中只含有一个fileName

正确行为来看，public继承应该总是virtual的

virtual继承会使体积比non-virtual继承的体积大；virtual base class的初始化规则很复杂。因此非必要不使用virtual继承；如果必须使用virtual继承，则尽量避免在virtual base class中放置数据，避免初始化和赋值的问题（类似java和C#中的interface）





## Chapter 7 Templates and Generic Programming

### Item 41: Understand implicit interfaces and compile-time polymorphism

面向对象编程中：显式接口（explicit interface）可以看到接口的定义；运行期多态（runtime polymorphism）在运行期根据对象类型调用对应函数

templates和泛型编程：隐式接口（implicit interface）由有效表达式组成；编译器多态（compile-time polymorphism）对函数的调用会使得编译期发生模板实例化


### Item 42: Understand the two meanings of typename

声明模板类型参数时，typename和class完全相同

模板内出现的名称如果依赖某个模板参数，则其为dependent name，否则为non-dependent name；如果dependent name在类中嵌套，则为nested dependent name；如果nested dependent name指涉一个类型，则其为nested dependent type name

如果要在模板中指涉一个nested dependent type name，需要前置关键字typename；但是typename不可以出现在base classes list内的nested dependent type name之前，也不能在member initialization list中作为base class的修饰符

```C++
Template<typename T>
Class Derived: public Base<T>::Nested { // base classes list中不能出现typename
Public:
	Explicit Derived(int x)
	: Base<T>::Nested(x) { // member initialization list中不能出现typename
		Typename Base<T>::Nested temp; // 需要出现typename
	}
};
```


### Item 43: Know how to access names in templatized base classes

类的base class template的不用特化版本可能不一同和一般性base class template版本相同的接口，，因此这个类往往拒绝在templatized base class（模板化基类）中寻找继承而来的名称

对于templatized base class，使用

	1. Base class的函数调用动作前加上`this->`
	2. 使用using声明式，`using MsgSender<Company>::sendClear;`
	3. 指出被调用的函数位于base class内，`MsgSender<Company>::sendClear(info);`，但是会关闭virtual绑定行为


### Item 44: Factor parameter-independent code out of templates

模板可能带来代码膨胀

非类型模板参数（non-type template parameters，例如int参数），适当情况下可以以函数参数或成员变量代替模板参数，但是会损失编译器常量优化

模板类型参数（type parameters）造成的代码膨胀，可以让完全相同二进制表述的实例化类型共享实现码来解决


### Item 45: Use member function templates to accept "all compatible types"

继承关系的B和D两类型分别实例化某个模板，产生的两个实例化体`SmartPtr<Base>`和`SmartPtr<Derived>`并不带有继承关系

`SmartPtr<Base> ptr = SmartPtr<Derived>(new Derived);`对于SmartPtr的拷贝构造函数，应当写为兼容所有构造类型的版本

```C++
Template<typename T>
Class SmartPtr {
Public:
	Template<typename U>
	SmartPtr(const SmartPtr<U> & other); // 生成U能够接受所有兼容类型的拷贝构造函数
};
```
上例为非explicit以支持兼容类型模板参数的SmartPtr之间的隐式类型转换

对于兼容的赋值操作

```C++
Template<class T>
Class shared_ptr {
Public:
	template<class Y> explicit shared_ptr(Y* p);
	template<class Y> shared_ptr(shared_ptr<Y> p);
	template<class Y> explicit shared_ptr(weak_ptr<Y> p);
	template<class Y> explicit shared_ptr(Y* p);
	
	Template<class Y> shared_ptr& operator=(shared_ptr<Y> const& r);
};
```
大部分为explicit强调不能从裸指针类型或其他智能指针类型隐式类型转换为shared_ptr类型

成员模板并不改变语言规则，即声明泛化的拷贝构造模板并不会阻止编译器生成他们自己的拷贝构造函数



### Item 46: Define non-member functions inside templates when type conversions are desired

template实参推导只施行于function templates上，class templates并不依赖template实参推导

friend能够在类内部声明一个non-member函数

Class template内，template名称可作为template和其参数的简略表达方式

类似Item 24，需要类型转换时要为模板定义非成员函数，但是非成员函数模板在推导时不考虑任何隐式类型转换，需要借助class template的实例化提供一个实例化的非成员函数（而非模板），因此借助friend实现声明在class template内部（随着class template实例化而实例化为非成员函数）的non-member 函数


```C++
Template<class T>
Class Rational {
Public:
	// 实例化为non-member函数（而非模板），支持隐式类型转换，支持下面的调用
	friend const Rational<T> operator * (const Rational<T> & lhs, const Rational<T> & rhs); 
};
// 函数模板，不支持隐式类型转换，无法支持下面的调用
Template <typename T>
Const Rational<T> operator * (const Rational<T> & lhs, const Rational<T> & rhs);

Rational<int> result = 2 * Rational<int>(1, 2);
```


## Chapter 8 Customizing new and delete

### Item 49: Understand the behavior of the new-handler

operator new无法满足内存分配需求时，会先调用一个通过set_new_handler函数制定的错误处理函数，然后抛出异常（item51）

```C++
namespace std {
	typedef void (*new_handler) ();
	new_handler set_new_handler(new_handler p) throw(); // 表示该函数不抛出异常
}
```

set_new_handler函数将p设置为operator new无法分配足够内存被调用的函数，返回set_new_handler被调用前的new_handler函数；只有当指向new_handler函数的指针是null才会抛出bad_alloc异常，否则会不断调用new_handler函数，operator new不止一次尝试分配内存，new_handler函数也能能够做某些动作将内存释放出来；

CRTP 



### Item 51: Adhere to convection when writing new and delete

operator new当分配内存成功时，返回一个指向内存的指针；当global的new_handler为null时，抛出bad_alloc异常。因此operator new实际上并不一定只尝试一次内存分配，而是在未能成功分配内存时不断地调用new_handler

当自定义new操作的类被继承时，如果没有显式定义继承类的operator new函数，会产生问题

所有非附属（freestanding）对象必须有非零大小（不是以某对象的基类成分存在的对象）

如果要控制类的array new操作，那么需要分配一块未加工内存，因为不知道每个对象多大（operator new[]有可能经由继承被调用）；并且有可能因为需要存放元素个数导致传给operator new[]的size_t参数比将被填充对象的内存更大

C++保证删除null指针永远安全

如果class自定义的operator new将大小有误的分配行为转交::operator new执行，那么必须将大小有误的删除行为转交::operator delete执行

```C++
// C++98
class Base {
public:
	static void* operator new(std::size_t size) throw(std::bad_alloc) {
		if (size != sizeof(Base)) // 大小错误，不是分配Base大小的内存（例如Derive类）
			return ::operator new(size); //
		... // 否则在这里自定义处理
	}
	
	static void operator delete(void * rawMemory, std::size_t size) throw() {
		if (rawMemory == 0) return;
		if (size != sizeof(Base)) { // 大小错误，不是删除Base大小的内存（例如Derive类）
			::operator delete(rawMemory); //转交给标准operator delete处理
			return;
		}
		... // 否则在这里归还rawMemory指向的内存
	}
};
```

如果被删除对象派生自Base类，并且Base类欠缺virtual dtor，那么C++传给operator delete的size_t数值可能不正确？？？，operator delete可能无法正确运作




### Item 52: Write placement delete if you write placement new

写一个new表达式调用两个函数：用以分配内存的operator new，和类的构造函数

如果内存分配成功但是构造函数发生异常，那么需要归还内存，否则会发生内存泄漏。runtime中系统会调用operator new的相应的operator delete版本。因此写出了placement new需要写对应的placement delete

placement new为接受除了size_t标准参数之外的额外参数的new操作，

在内存分配成功而构造函数抛出异常时，运行期系统会寻找参数类型和个数都与operator new相同的某个operator delete，找到则调用释放分配的内存空间；如果没找到，那么什么都不做，这会导致内存泄漏

没有在new中的构造函数中抛出异常并且客户代码中含有`delete ptr;`语句，调用的是正常形式是operator delete，即对一个指针调用delete绝不会导致调用placement delete；只有在new中的构造函数中抛出异常，才会调用对应版本的delete操作，即placement new调用对应版本的placement delete

因此如果有placement new版本可能发生内存泄漏，需要提供一个正常版本的operator delete和额外操作与placement new一样的placement delete

默认情况下C++在global作用域中提供如下operator new

```C++
void* operator new(std::size_t size) throw (std::bad_alloc);
void* operator new(std::size_t size, void *) throw ();
void* operator new(std::size_t size, const std::nothrow_t &) throw ();
```

class中定义的new操作掩盖会其他new操作（包括global作用域中的new），需要提供需求要求的所有new操作并提供对应的delete操作，如果需要有平常的行为则令其调用global版本即可

如果要自定形式的placement new，例如其他参数的版本，则可继承StandardNewDeleteForms类型，并使用`using StandardNewDeleteForms::operator new`和``using StandardNewDeleteForms::operator delete`来获得标准形式的new和delete
