## Introduction



### Terminology and Convections

四个官方C++版本：C++98，C++03，C++11，C++14

C++11最为普遍的特征是使用了move语义（以rvalue和lvalue为基础），rvalue表明对象可以执行move操作，lvalue不可以。

区分lvalue和rvalue的一个启发性方法是看能否取得他的地址，可取得则往往为lvalue。这可以帮助我们记住一个表达式的类型独立于其是lvalue还是rvalue，即给出类型T，可以有lvalue的类型T和rvalue的类型T。

对于rvalue reference类型的参数，他也是一个lvalue，可以取得rhs的地址，因此rhs为一个lvalue，rhs的类型为rvalue reference

```C++
class Widget {
public:
	Widget(Widget&& rhs); // rhs is an lvalue, though it has
     ...                        // an rvalue reference type
};
```

书中在描述一个一个对象来初始化同样类型的另一个对象时，称新对象是原有对象的copy，即使这个新对象是由copy或者move构造而成，c++中没有术语区分copy-constructed copy和move-constructed copy

argument是函数调用中传给函数的表达式，用于初始化函数的parameter。parameter是lvalue，而arguments可能是lvalue或者rvalue

perfect forwarding是将传给函数的arguments再次传给另一个函数，并且最初的arguments的rvalueness或者lvalueness性质得以保持

设计良好的函数是exception safe的，即函数提供基本异常安全保证（the basic guarantee），即使抛出异常，程序不变量保持完好并且没有资源泄露。提供强异常安全保证（the strong guarantee）的函数，即使抛出异常，程序状态和调用函数前相同

function object往往是指支持operator()调用的对象

通过lambda创建的函数对象称为closure

function template可以产生template function，class template可以产生template class

声明引入名称和类型，定义明确了storage和具体实现

函数签名包含了函数的参数类型和返回值（official定义往往忽略返回值），不包含函数名和参数名，也不包含函数声明中的其他部分（例如noexcept，constexpr）

尽量避免deprecate特性，例如std::auto_ptr

smart pointer往往重载了pointer dereferencing operator（operator*和operator->）,std::weak_ptr例外



## Chapter 4: Smart Pointers

### Item 18: use std::unique_ptr for exclusive-ownership resourse management

默认情况下，Unique_ptr和raw ptr大小一样，大多数操作执行指令一样

Move-only type

使用std::forward转发new指令（item 25）

可自定义deleter，传入raw ptr，增加大小（函数deleter增加至少一个函数指针大小；非闭包lambda不增加大小）

可类型转换至std::shared_ptr

两种类型，unique_ptr<T>和unique_ptr<T[]>

### Item 19: Use std::shared_ptr for shared-ownership resource management

引用计数的存储必须动态分配

计数增减是原子操作

move op不影响引用计数

unique_ptr的deleter是其类型的一部分，shared_ptr不是

shared_ptr的deleter与引用计数等一起作为control block动态分配


	Std::shared_ptr<T>
	===================           ==========
	| Ptr to T                       |   ----> | T Object |
	------------------------------            ==========
	| Ptr to Control Block |   ---
	===================    |
	                                              ---> Control Block
	                                                     ==============================
	                                                     | Reference Count                              |
	                                                     -------------------------------------------------
	                                                     | Weak Count                                      |
	                                                     -------------------------------------------------
	                                                     | Other Data (e.g., custom deleter) |
	                                                     ==============================


如果从shared_ptr或者weak_ptr创建一个shared_ptr，复用control block；如果从raw ptr或者unique_ptr/auto_ptr或者std::make_shared创建一个shared_ptr，分配新的control block（资源本身并不能得知自己已经被一个shared_ptr指向了）

一个资源的shared_ptr拥有多control block会导致资源被释放多次！！！

如果需要给shared_ptr构造函数传入raw ptr，直接使用new而非引入一个raw ptr，避免直接传入raw ptr来构造shared_ptr

自定义类A需要使用shared_ptr包装的this，需要继承Std::enable_shared_from_this类（一个CRTP类，含有share_from_this成员函数，返回一个shared_ptr包装的this指针）

Share_from_this返回的shared指针会使用一个已有control block，因此需要在shared_from_this函数外存在一个已有shared_ptr指向this（通过enable_shared_from_this的ctor为私有，使用工厂函数create返回一个shared_ptr）

一般control block内含有虚函数（确保正确销毁资源）、院子引用计数操作、动态分类cblock、自定义资源的allocator和deleter

shared_ptr解引用开销等同于raw ptr，操作开销需要几个原子操作

无法从shared_ptr生成unique_ptr

没有shared_ptr<T[]>类型，不能将数组视作T类型放入shared_ptr，需要使用（std::array等）

shared_ptr不支持operate[]；

shared_ptr提供了derived-to-base指针转换（unique_ptr不支持），对数组使用会导致混乱

### Item 20: Use std::weak_ptr for std::shared_ptr like pointers that can dangle.

Weak_ptr正确处理悬空指针

Weak_ptr不能销毁，不影响引用计数（不参与shared ownership），不能独立存在，是shared_ptr的一个augmentation，由shared_ptr创建而来

Weak_ptr通过expired()函数检查是否悬空

原子操作从weak_ptr测试是否dangle并创建Shared_ptr两种方式

	1. Auto shared_ptr_w = weak_ptr_w.lock() // 如果悬空返回null
	2. shared_ptr<W> shared_ptr_w(weak_ptr_w) // 如果悬空抛出std::bad_weak_ptr异常

接受资源id返回unique_ptr的工厂函数可能会有很大开销，并为此在内部使用cache，需要维护这些cache将工厂函数修改为内部使用unordered_map<IDType, weak_ptr<W>> cache，并返回shared_ptr<W>

Observer模式，subject管理状态，在状态变更时通知所有observer，因此subject需要维护一个weak_ptr<Observer>的容器以确保他们的可用

A和C共享资源B，都通过shared_ptr指向B，B若要链接到A则需要用weak_ptr（B可以探测指向A的weak_ptr是否悬空），若用

	1. Raw ptr，若A被销毁，C仍然指向B，那么B指向A的raw ptr悬空
	2. Shared_ptr，A和B形成环，各自有一个引用计数为1的shared_ptr，均无法销毁

若在一个树种，父节点拥有子节点的所有权，那么父节点通过unique_ptr指向子节点，子节点通过raw ptr指向父节点即可；如果所有权并不是如此严格，那么需要考虑weak_ptr

Weak_ptr和shared_ptr占用同样大小空间，复用shared_ptr的control block


### Item 21: Prefer std::make_unique and std::make_shared to direct use of new.

Make_shared是c++11，make_unique是c++14，自定义make_unique为
	
	template<typename T, typename... Ts>
	std::unique_ptr<T> make_unique(Ts&&... params)
	{
		return std::unique_ptr<T>(new T(std::forward<Ts>(params)...));
	}
	
make_unique只是将参数完美转发给构造函数（不支持数组和自定义deleter），不要将其放入std命名空间中

三个make函数（传入参数集合，完美转发给动态创建的对象，并返回智能指针）为make_unique，make_shared，allocate_shared（该函数接收allocator object用于动态内存分配）

#### 多数情况下使用make函数而非智能指针创建函数

	1. 少打一个类型名称
	2. 类型安全
	3. 效率（对make_shared和allocate_shared而言）

少打一个类型名称的情况
	
	auto upw1(std::make_unique<Widget>()); // with make func
	std::unique_ptr<Widget> upw2(new Widget); // without make func 
	auto spw1(std::make_shared<Widget>()); // with make func
	std::shared_ptr<Widget> spw2(new Widget); // without make func
	
类型安全的情况
	
	processWidget(std::shared_ptr<Widget>(new Widget), computePriority());
	
编译器按照 new Widget，computePriority()，std::shared_ptr<>()的顺序创建，则computePriority产生异常则会使new Widget内存泄露
	
效率（对make_shared和allocate_shared而言）
	
Shared_ptr<W> spw(new W)分配两次内存（W类型和control block）
	
make_shared<W>()允许编译器优化为分配一次内存（W类型和control block分配在一起）

#### 某些情况下不使用make函数（对于make_unique和make_shared而言）

	1. 不允许自定义deleter
	2. 句法细节
	
句法细节

（item7）通过大括号创建的对象倾向于使用std::initializer_list参数的ctor，小括号倾向于non-std::initializer_list的ctor。在make函数中使用完美转发处理这些参数，倾向于使用小括号的ctor；即需要braced initializer的地方需要使用new+braced initializer；（item30）braced initializer不能被完美转发，但是可以通过auto类型推导获得initializer_list对象，并传给make函数
	
	auto initList = { 10, 20 };
	auto spv = std::make_shared<std::vector<int>>(initList);
	
#### 某些情况下对于make_shared而言还存在更多的问题

	1. 对于重载了op new和op delete的类，不要使用make函数（这些往往被设计为处理sizeof(W)大小的内存，但是make函数会分配sizeof(W)+sizeof(control block)大小的内存）
	2. 使用make_shared函数，当引用计数为0，调用对象的dtor，但是需要等到control block的内存被释放时才能释放整个占用的内存
	3. Control block上含有ref count和weak count，weak_ptr通过检查ref count（而非weak count）判断是否expired，如果ref count为0则调用dtor，weak_ptr已经expired。只要weak_ptr联系了control block（weak count > 0），那么control block需要存在，即make函数分配的内存（包括已经销毁的对象和需要存在的control block）仍然不能被释放（直到所有shared_ptr和weak_ptr都销毁）。即当对象很大并且销毁最后一个shared_ptr和weak_ptr之间的时间很长，使用make_shared会出现内存问题；而直接使用new语句并构造shared_ptr不会出现这种问题

在不能使用make函数时，最好做法是

> new语句创建指针并在一个语句中立即将其传给smart point的构造函数，并且这个语句中不做任何别的事情。

	processWidget(std::shared_ptr<Widget>(new Widget, customDeleter), computePriority());

需要修改为

	std::shared_ptr<Widget> spw(new Widget, customDeleter);
	processWidget(spw, computePriority()); // lvalue

一定条件下可以优化为

	std::shared_ptr<Widget> spw(new Widget, customDeleter);
	processWidget(std::move(spw), computePriority()); // rvalue
	

### Item 22: When using the Pimpl Idiom, define special member functions in the implementation file.

Pimpl Idiom一种在对类进行修改后加速编译的方法
	
	#include <string>
	#include <vector>
	Class Widget {
	private:
		std::string name;
		std::vector<double> data;
	};

修改为

	#include "Impl"
	Class Widget {
	Public:
		Widget() : pImpl(std::make_unique<Impl>()) {}
	private:
		Struct Impl;
		Impl * pImpl;
	};

声明未定义的类型称为incomplete type

????





## Chapter 5: Rvalue References, Move Semantics, and Perfect Forwarding

move语义：使用廉价的move代替昂贵的copy，并且允许创造move-only类型（例如std::unique_ptr，std::future,std::thread）

Perfect forwarding：可以写出接受抽象参数类型的函数模板，并将这些抽象类型参数forward给其他函数，使得这些函数接收的参数类型和我们传给forward函数的类型完全一致

参数永远是lvalue，即使他的类型是rvalue reference

	Void f(Widget&& w);
	
w是lvalue，即使他的类型是rvalue-reference-to-Widget


### Item 23: Understand std::move and std::forward

move和forward并不产生任何可执行代码，均为cast函数

move无条件将参数转为rvalue，forward仅当确定条件满足执行这个转换

c++14下的move的类似实现

	template<typename T>
	decltype(auto) move(T&& param)
	{
		using ReturnType = remove_reference_t<T>&&;
		return static_cast<ReturnType>(param);
	}


如果要move一个对象，不要将他声明为const（const对象的move操作会转换为拷贝操作）

move不保证move操作cast的对象是可以move的

	class Annotation {
	public:
		explicit Annotation(const std::string text) // lvalue const string
			: value(std::move(text)) {} // 应用move之后是rvalue const string
	}

string的构造函数

	string(const string& rhs); // copy ctor
	string(string&& rhs); // move ctor

Move ctor接受non-const类型，上述构造函数使用了copy ctor

	void process(const Widget& lvalArg); // lvalue
	void process(Widget&& rvalArg); // rvalue

如果形参通过rvalue初始化（形参是由lvalue或rvalue初始化记录在T中，item28），则forward将其cast为rvalue

	template<typename T> 
	void logAndProcess(T&& param) 
	{
		// 所有函数的实参均为lvalue
		process(std::forward<T>(param)); // 调用process(Widget&& rvalArg)
	}
	
	logAndProcess(w); // call with lvalue
	logAndProcess(std::move(w)); // call with rvalue

纯技术角度上说forward可以干一切需要move的活

forward需要加上type模板参数，move不需要

forward参数必须为non-reference，如果传入string&会发生拷贝构造而非移动构造（because that’s the convention for encoding that the argument being passed is an rvalue (see Item 28).？？？）

### Item 24: Distinguish universal references from rvalue references.

Universal reference出现的两个例子，都出现了type deduction

	template<typename T>
	void f(T&& param); // param为universal reference
	auto&& var2 = var1; // var2为universal reference

Universal reference需要被初始化，lvalue进行初始化则为lvalue reference，rvalue初始化则为rvalue reference

	template<typename T>
	void f(T&& param); // param is a universal reference
	Widget w;
	f(w); // 左值传给f，param类型为左值引用Widget& (i.e., an lvalue reference)
	f(std::move(w)); // 右值传给f，param类型为右值引用Widget&& (i.e., an rvalue reference)

作为universal reference，不仅需要涉及类型推导，还需要精确出现T&&（即f被调用时T会被类型推导，Corner case为除非调用者显式指定T）

	template<typename T>
	void f(std::vector<T>&& param); // 右值引用而非universal reference
	template<typename T>
	void f(const T&& param); // 右值引用而非universal reference

Corner case：

	template<class T, class Allocator = allocator<T>> // from C++
	class vector { // Standards
	public:
	void push_back(T&& x); // x为右值引用而非universal reference
	};
	
Push_back函数的存在依赖vector的实例化，即声明了vector<Widget>导致push_back实例化为push_back(Widget&& x);

与之相反

	template<class T, class Allocator = allocator<T>> // still from
	class vector { // C++
	public: // Standards
		template <class... Args>
		void emplace_back(Args&&... args); //不依赖于vector，调用时进行类型推导，为universal reference
	};

个人总结为，“调用时需要进行类型推导”的“形如T&&”的类型为universal reference

Auto universal reference，auto&&可以绑定到任何抽象类型

	auto timeFuncInvocation = [](auto&& func, auto&&... params) // C++14
	{
		start timer;
		std::forward<decltype(func)>(func)( // invoke func
			std::forward<decltype(params)>(params)... // on params
		);
		stop timer and record elapsed time;
	};
	
func可以绑定到任何callable object，lvalue，rvalue，args可以绑定到任何抽象类型

### Item 25: Use std::move on rvalue references, std::forward on universal references.

move用于rvalue reference（右值引用被move无条件cast为右值）

	class Widget {
	public:
		Widget(Widget&& rhs) // rhs is rvalue reference
			: name(std::move(rhs.name)), p(std::move(rhs.p)) { … }
			
forward用于universal reference（universal reference被forward有条件的cast为右值）

	class Widget {
	public:
		template<typename T>
		void setName(T&& newName) // newName is universal reference
		{ name = std::forward<T>(newName); } 

move用于universal reference会带来灾难

	class Widget {
	public:
		template<typename T>
		void setName(T&& newName) // universal reference
		{ name = std::move(newName); }
	};
	Auto n = …;
	w.setName(n);
	// 现在n的值不确定

内部需要将参数传给其他函数的函数，不要为其重载参数为const string&或者string&&这两个版本，而是使用universal reference并在内部使用forward将参数传递给其他函数

对于最后一次使用的rvalue/universal reference，应用move（右值引用）或者forward（universal reference）

Return by value函数，并且返回右值引用或者universal reference，则使用move或者forward返回
Move

	Matrix operator+(Matrix&& lhs, const Matrix& rhs) // lhs为右值，重用其存储空间从而保留矩阵和
	{
		lhs += rhs;
		return std::move(lhs); // 使用move将lhs cast为右值
		Return lhs; // 拷贝lhs至return value
	}
	
若matrix不支持move操作，return std::move(lhs)将调用拷贝构造函数

Forward

	template<typename T>
	Fraction reduceAndCopy(T&& frac) // universal reference param
	{
		frac.reduce();
		return std::forward<T>(frac);
	}
	
frac为右值则move，左值则拷贝

Return by value函数，但是返回函数内的局部变量，不要应用move或者forward（会使用C++的返回值优化RVO）

	Widget makeWidget() // Moving version of makeWidget
	{
		Widget w;
		return std::move(w); // move w into return value，(don't do this!)
		return w; // 编译器可能使用RVO
	} 
	
满足以下两个条件，编译器会忽略对函数return by value的局部变量的拷贝或者移动操作：

	1. 局部变量类型和返回值类型一样
	2. 返回这个局部变量

RVO根据局部变量的是不是named（临时变量）又可看做RVO和NRVO（unamed）

若上述例子使用了return std::move(w)；则返回的不是这个局部变量，使得编译器无法进行RVO

如果编译器满足上述两个条件但并没有使用RVO（满足RVO条件但不应用 RVO），那么被返回的局部变量必须被视作右值，因此在RVO可以应用的情况下，编译器：

	1. 应用RVO
	2. 视返回值为rvalue，隐式应用move：
	
	Widget makeWidget()
	{
		Widget w;
		return std::move(w);
	}

同样的，对于by-value函数参数，They’re not eligible for copy elision with respect to their function’s return value（返回值不能使用copy取消RVO？？？）, but compilers must treat them as rvalues if they’re returned（如果返回编译器要将他们视作右值？？？）. 

	Widget makeWidget(Widget w) // by-value参数与返回值类型一样
	{ return w; }
	
视作被写为

	Widget makeWidget(Widget w) // w视作右值
	{ return std::move(w); }

### Item 26: Avoid overloading on universal references 

	std::multiset<std::string> names;     // global data structure
	void logAndAdd(const std::string& name)
	{
	     names.emplace(name);
	}
	logAndAdd(petName); // pass lvalue std::string, emplace处执行copy ctor
	logAndAdd(std::string("Persephone")); // pass rvalue std::string，name是lvalue，emplace处执行copy ctor
	logAndAdd("Patty Dog"); // 直接将string literal传给emplace函数，emplace处使用string literal来在multiset的内部copy ctor一个string类型

	template<typename T> 
	void logAndAdd(T&& name) {
		auto now = std::chrono::system_clock::now(); log(now, "logAndAdd");     names.emplace(std::forward<T>(name));
	}
	logAndAdd(petName); // pass lvalue std::string, emplace处执行copy ctor
	logAndAdd(std::string("Persephone")); // move右值进入multiset
	logAndAdd("Patty Dog"); // create std::string in multiset instead of copying a temporary std::string

如果给logAndAdd添加一个参数为string id的重载函数

	void logAndAdd(int idx)
		names.emplace(nameFromIdx(idx));
	}
	logAndAdd(22); // 调用此重载版本
	short id;
	logAndAdd(id); // 精确匹配universal reference参数版本

使用universal reference的函数是C++中最贪婪的函数，几乎可以精确匹配任何类型（item 30几个例外），因此不应该使用含有universal reference参数的重载函数

	class Person {
	public:
	  template<typename T>
	  explicit Person(T&& n)
	  : name(std::forward<T>(n)) {} // perfect forwarding ctor
	
	explicit Person(int idx);
	  Person(const Person& rhs); // 编译器自动生成
	  Person(Person&& rhs); // 编译器自动生成
	... };

	Person p("Nancy");
	auto cloneOfP(p);
	
T 实例化为Person&，而编译器生成的copy ctor含有一个const，匹配完美转发ctor

	const Person p("Nancy");
	auto cloneOfP(p);
	
T 实例化为 const Person&，在重载决议时若模板函数和普通函数均同等匹配，优先选择普通函数，匹配copy ctor

涉及到继承时

	class SpecialPerson: public Person {
	public:
		SpecialPerson(const SpecialPerson& rhs)  // copy ctor，调用基类的转发构造函数
			: Person(rhs) {... }
		SpecialPerson(SpecialPerson&& rhs) // move ctor，调用基类的转发构造函数
			: Person(std::move(rhs))  {... }
	};
	
子类总是将类型为SpecialPerson的参数传给父类


### Item 27: Familiarize yourself with alternatives to overloading on universal references

避免重载universal reference

#### Pass by lvalue-reference-to-const(const T &)

（item26）但是效率有所下降

#### Pass by value

	class Person {
	public:
		explicit Person(std::string n) // replaces T&& ctor; see
			: name(std::move(n)) {} // Item 41 for use of std::move
		explicit Person(int idx) // as before
			: name(nameFromIdx(idx)) {}
	Private:
		std::string name;
	};

#### Tag dispatch

如果继续使用universal reference并且使用重载，则使用tag dispatch（将需要重载的函数写成logAndAddImpl，使用一个带有tag参数的函数根据tag将调用分发给某个对应的impl）
原有错误重载形式

	template<typename T> 
	void logAndAdd(T&& name) { … }
	void logAndAdd(int idx) { … }
	
Tag dispatcher为

	template<typename T>
	void logAndAdd(T&& name) {
		logAndAddImpl(std::forward<T>(name), 
			std::is_integral<typename std::remove_reference<T>::type>()); 
	}
	
原有universal reference non-integral函数为

	template<typename T> // non-integral
	void logAndAddImpl(T&& name, std::false_type) { // compile time value（true和false是runtime values）
		names.emplace(std::forward<T>(name));
	}
	void logAndAddImpl(int idx, std::true_type) { // integral
		logAndAdd(nameFromIdx(idx)); 
	}

logAndAdd(stringId)会被正确分发至logAndAddImpl(int idx)函数中


#### 约束使用universal reference的模板

即使使用tag dispatch写了一个ctor处理重载问题，对于编译器自动生成的copy和move ctor，还是会有ctor调用绕过tag dispatch机制的情况：提供一个universal reference的ctor会使得拷贝non-const lvalue时调用universal reference版本（此时期望调用copy ctor），或者调用子类拷贝或移动ctor时调用了父类的universal reference ctor版本

使用std::enable_if

如果希望ctor的参数不是Person类型时调用universal reference ctor版本（Person类型期望调用copy/move ctor）

	class Person {
	public:
		template<typename T, 
				typename = typename std::enable_if<condition>::type> // 参考SFINAE
		explicit Person(T&& n);
	};
	
condition为!std::is_same<Person, T>::value // 仍然错误

给出

	Person p("Nancy");
	auto cloneOfP(p); 
	
这里构造cloneOfP的过程中uniref中的T被推导为Person&，跟Person不是一类，condition为true就仍然调用uniref，错误！因此准确地说应该忽略（使用std::decay<T>::type）

	1. 类型是非为ref，Person Person& Person&&均视作同样类型
	2. 类型为const或volatile，const Person/volatile Person等视作同类型
	
并且，如果由子类的copy/move ctor调用父类的对应ctor，父类ctor的传入参数是子类类型，因此需要当类型为Person或者Person的子类时，不调用uniref版本（使用std::is_base_of<T1, T2>::value，is_base_of<T, T>::value为真）

condition应该修改为

	!std::is_base_of<Person, typename std::decay<T>::type>::value

正确的Person类如下

	class Person {
	public:
		template<typename T,
				typename = std::enable_if_t<
					!std::is_base_of<Person, std::decay_t<T>>::value &&
					!std::is_integral<std::remove_reference_t<T>>::value
					>
				>
		explicit Person(T&& n) // ctor for std::strings and
			: name(std::forward<T>(n)) // args convertible to std::strings
		{
			// 如果类型（和string）匹配失败，用于产生良好的错误信息
			// 但是这些信息仍然在成员初始化列表中的forward产生的复杂的内部错误信息之后
			static_assert( 
				std::is_constructible<std::string, T>::value,
				"Parameter n can't be used to construct a std::string"
			);
		
		}
		
		explicit Person(int idx) // ctor for integral args
			: name(nameFromIdx(idx)) { … }
		
		… // copy and move ctors, etc.
	private:
		std::string name;
	};

Tag dispatch和constrained template都使用了perfect forward技术，有两个缺点：

	1. 有些参数不能perfect forward
	2. 错误信息复杂，只有在真正不匹配的位置（经过层层函数调用）产生错误信息

### Item 28: Understand reference collapsing.

template<typename T> void func(T&& param);会将参数是lvalue还是rvalue编码进uniref的T的类型推导中， 即如果是lvalue，T为lvalue reference，rvalue则T为non-reference

	Widget widgetFactory(); // 返回rvalue
	Widget w; // lvalue
	func(w); // T为Widget&
	func(widgetFactory()); // T为Widget

在c++中引用的引用（int x; auto & & refx = x;）是非法的，对于lvalue传给uniref，T为Widget&，函数为void func(Widget& && param); 经过reference collapsing（编译器在特定场景例如模板实例化时进行的将引用的引用collapse为一个引用的操作），最终变成void func(Widget& param);

Lvalue和rvalue一共有四种collapse可能

> 如果有一个引用为lvalue reference，则结果为lvalue reference；如果全为rvalue reference，则结果为rvalue reference

Reference collapse发生于四种场景中：

	1. 模板实例化
	2. auto的类型生成
	3. typedef和alias声明
	4. decltype


#### 模板实例化

不规范的forward实现

	template<typename T> 
	T&& forward(typename remove_reference<T>::type& param) {
		return static_cast<T&&>(param);
	}
	
Lvalue reference传给forward，T为Widget&

	Widget& && forward(typename remove_reference<Widget&>::type& param)
	{ return static_cast<Widget& &&>(param); }
	
结果为

	Widget& forward(Widget& param)
	{ return static_cast<Widget&>(param); }

Rvalue reference传给forward，T为Widget&&

	Widget&& && forward(typename remove_reference<Widget&&>::type& param)
	{ return static_cast<Widget&& &&>(param); }
	
结果为

	Widget&& forward(Widget& param)
	{ return static_cast<Widget&&>(param); }

	
#### auto类型生成

	Widget widgetFactory(); // 返回rvalue
	Widget w; // lvalue
	Auto && w1 = w; // lvalue初始化w1，auto为Widget&，auto &&为Widget&
	Auto && w2 = widgetFactory(); //  rvalue初始化w2，auto为Widget&&，auto&&为Widget&&

因此uniref并不是新的ref类型，而是

	1. 类型推导，区分lvalue和rvalue（lvalue的T推导为T&，rvalue的T推导为T）
	2. 发生Reference collapse


#### typedef和alias声明

一个typedef

	template<typename T>
	class Widget {
	public:
		typedef T&& RvalueRefToT;
	};
	
调用

	Widget<int&> w;
	
其中typedef int& && RvalueRefToT; 被collapse为typedef int& RvalueRefToT;

#### decltype

查看Item3

### Item 29: Assume that move operations are not present, not cheap, and not used

对于没有特别为c++11优化过的c++98程序，move带来的改进并不一定很大。例如move op的自动生成要求类型没有声明copy op/move op/dtor，这些类如果没有显式对move的支持，并不能得到性能提升。

对于将内容存储在堆区的容器，move只需要移动他的指针并将原有指针至空即可，和设置一对指针一样高效；对于std::array来说，内容是直接存储在对象中的，如果存储元素的move比copy快很多，array的move也比copy快，但总的来说，move和copy都是线性时间，move比设置一对指针耗时很多。

string的move常数时间，copy线性时间，但是对于应用了SSO（small string optimization，如将长度小于15的string直接存储在对象内的buffer上而非分配在堆区）的string，move和copy一样耗时

（Item14）一些容器提供了strong exception safe guarantee，为了从C++98升级到C++11时这些guarantee仍然成立，只有当move确定不会抛出异常才会使用move。因此对于高效的move实现，如果没有noexcept声明，仍然可能调用copy

C++11中的move语义不会带来好处：

	1. 不支持move操作：被move的对象不支持move，会转为copy请求
	2. move并不高效：效率可能并不比copy更高
	3. move不可用：需要noexcept的move时，没有声明noexcept
	4. 源对象是lvalue：只有rvalue才能作为move的源对象（例外见item25）

对于不清楚move情况的代码，当做c++98处理（除非明确得知move相关的信息）


### Item 30 Familiarize yourself with perfect forwarding failure cases

forward仅用于处理参数为reference的情况（by-value参数会导致copy，指针参数要求调用方传递指针）

	template<typename T>
	void fwd(T&& param) { // accept any argument
		f(std::forward<T>(param)); // forward it to f
	}
	template<typename... Ts>
	void fwd(Ts&&... params) { // accept any arguments
		f(std::forward<Ts>(params)...); // forward them to f
	}

有几种参数类型会导致perfect forward失败（调用目标函数f和转发函数fwd产生不同表现）：

	1. Braced initializers
	2. 0 or NULL as null pointers
	3. Declaration-only integral static const data members
	4. Overloaded function names and template names
	5. Bitfields

#### Braces initializers

	Void f(const vector<int&> v);

调用f函数，编译器比较传入参数和f函数的参数声明是否匹配，如需要会发生隐式转换

	f({ 1, 2, 3}); // 可行，隐式转换为vector<int>

对fwd的调用不比较传入参数和fwd函数参数声明，而是推导传给fwd的参数的类型，并将推导类型与f的参数声明进行比较

	fwd({ 1, 2, 3}); // 编译错误
	
没有声明为std::initializer_list的braced initializer是一个non-deduced context，编译器无法进行类型推导

	auto il = { 1, 2, 3 }; // auto推导为std::initializer_list<int>
	fwd(il); // 可行，fwd将il完美转发给f

Perfect forward失败：

	1. 编译器无法推导出类型
	2. 编译器推导出错误类型


#### 0 or NULL as null pointers

0和NULL会被错误推导为整形而不是null pointer，请使用nullptr

#### Declaration-only integral static const data members

对于static const整形成员，声明即可（不需要定义它，因为对于这些成员的值编译器执行const propagation，不需要额外设置这些成员的内存，编译器将所有使用这些成员的地方直接用值替换）。但是只有定义了这些变量，才能够获取他们的地址。

	class Widget {
	public:
		static const std::size_t MinVals = 28;
	};
	f(Widget::MinVals); // fine, treated as "f(28)"
	fwd(Widget::MinVals); // error! shouldn't link
	
没有获取MinVals的地址，但是使用了uniref，在编译器生成代码中往往将reference视为指针，并且在程序的二进制码和硬件上，指针和引用是相同的。但也有编译器可能执行正确

	
#### Overloaded function names and template names

重载函数名情况
定义f为

	void f(int (*pf)(int)); // 等同于void f(int pf(int));
	
两个重载函数

	int processVal(int value);
	int processVal(int value, int priority);
	
调用

	f(processVal) // 可行，选择第一个processVal函数
	Fwd(processVal) // 错误，fwd不包含任何信息指明需要哪种类型

模板名情况
函数模板代表许多函数

	template<typename T>
	T workOnVal(T param) { … }
	fwd(workOnVal); // 错误，不知道使用哪一个实例化函数
	
正确做法为

	using ProcessFuncType = int (*)(int); 
	ProcessFuncType processValPtr = processVal;
	fwd(processValPtr); // 可行
	fwd(static_cast<ProcessFuncType>(workOnVal)); // 可行


#### Bitfields

（bitfield表示机器字中的一部分，例如32位int的第3-5位，不可能直接获取这个bitfield的地址，通常引用和指针在硬件层面上等同）

	struct IPv4Header {
		std::uint32_t version:4,
				IHL:4,
				DSCP:6,
				ECN:2,
				totalLength:16;
	…
	};
	
定义

	void f(std::size_t sz); // function to call
	IPv4Header h;
	f(h.totalLength); // fine
	fwd(h.totalLength); // 错误
	
h.totalLength是non-const bitfield，fwd的参数是reference，c++标准要求

> non-const reference不能绑定到bitfield上

没有函数能将引用绑定到bitfield上，或者获取bitfield的地址，因此bitfield参数只能是by-value或者reference-to-const（标准要求在一个标准整形的对象上存储这个bitfield的拷贝，并将reference绑定到这个标准整形的对象上？？？）

正确做法为

	auto length = static_cast<std::uint16_t>(h.totalLength); // 拷贝bitfield值
	fwd(length); // forward这个拷贝



## Chapter 6: Lambda Expressions

Lambda expression只是一个expression

closure是lambda创建的runtime对象，持有captured data的拷贝或者引用

closure class是用以实例化closure的类，编译器为每一个lambda产生一个独一无二的closure class

一个单独的lambda可以对应一个closure type的多个closure（closure可以）

编译期：lambda，closure classs
运行期：closure

### Item 31: Avoid default capture modes.

C++11的capture mode有by-reference和by-value两种。by-reference可能产生悬空引用，by-value不一定self-contained

by-reference的悬空引用

	using FilterContainer = std::vector<std::function<bool(int)>>;
	FilterContainer filters;
	filters.emplace_back([](int value) { return value % 5 == 0; } ); // filter函数容器添加一个函数
	
使用

	void addDivisorFilter()
	{
		auto divisor = …;
		filters.emplace_back([&](int value) { return value % divisor == 0; }); // divisor会变为悬空指针
		filters.emplace_back([&divisor](int value) { return value % divisor == 0; }); // divisor会变为悬空指针
	}

相比默认方式[&]而言，显式列出lambda中的局部变量[&divisor]会容易发现这个lambda依赖于这个引用并可能悬空

将divisor修改为by-value capture防止悬空引用

	filters.emplace_back([=](int value) { return value % divisor == 0; }); // divisor为by-value capture

lambda只能使用于在定义该lambda的作用域内可见的non-static local variable（包含参数）。类的non-static成员变量在访问时通过隐式添加this指针，因此lambda中的[=]只能捕获this指针，使用non-static成员变量的方式为

	void Widget::addFilter() const
	{
		auto currentObjectPtr = this; // 获得this指针
		filters.emplace_back(
			[currentObjectPtr](int value) // 捕获this指针
			{ return value % currentObjectPtr->divisor == 0; } );
	}
	
this变量生存期

	using FilterContainer = std::vector<std::function<bool(int)>>;
	FilterContainer filters;
	void doSomeWork() {
		auto pw = std::make_unique<Widget>(); 
		pw->addFilter(); // addFilter中的lambda捕获了pw的this指针存入filters
	}
	// 这里pw失效，filters中的捕获pw的this的lambda产生悬空指针
	
使用by-value捕获pw->divisor，将addFilter改为

	void Widget::addFilter() const {
		auto divisorCopy = divisor; // copy data member
		filters.emplace_back(
			[divisorCopy](int value) // capture the copy
			{ return value % divisorCopy == 0; } // use the copy
		);
	}

使用默认by-value capture同样正确（不推荐）A default capture mode is what made it possible to accidentally capture this when you thought you were capturing divisor in the first place.？？？

	void Widget::addFilter() const {
		auto divisorCopy = divisor; // copy data member
		filters.emplace_back(
			[=](int value) { return value % divisorCopy == 0; } // use the copy
		);
	}

C++14中generalized lambda capture可以捕获成员变量

	void Widget::addFilter() const
	{
		filters.emplace_back( // C++14:
			[divisor = divisor](int value) // copy divisor to closure
			{ return value % divisor == 0; } // use the copy
		);
	}

lambda不仅依赖于（可捕获的）局部变量和参数，还依赖于（不可捕获的）objects with static storage duration（定义在全局或名空间作用域下的对象，或者在类、函数、文件中声明为static的对象），因此by-value capture的lambda并不一定self-contained

	void addDivisorFilter()
	{
		static auto divisor = …;
		filters.emplace_back(
			[=](int value) // captures nothing!，并没有对任何对象作出by-value capture
			{ return value % divisor == 0; } // refers to above static
		);
		++divisor;
	}




### item 32: Use init capture to move objects into closures

C++14能够将move-only对象move到closure中，C++11无法做到

C++11中的新的捕获机制init capture能够做C++11的几乎所有捕获形式（除了默认捕获模式等），并且还能够可以实现capture-by-move，又称作generalized lambda capture

Init capture可以指定：

	1. Closure class中一个data member的名字
	2. 一个用来初始化这个data member的表达式

使用init capture将unique_ptr移动至closure中

	auto pw = std::make_unique<Widget>(); 
	auto func = [pw = std::move(pw)] // init capture，（少了个()参数列表？？？）
		{ return pw->isValidated() && pw->isArchived(); }; 

在init capture [pw = std::move(pw)]中，=左边是指定的closure的data member名称，作用域是closure class；=右边是初始化表达式，作用域和该lambda的定义位置一样

局部变量pw并不是必须的，因为closure class的data member可以直接被make_unique初始化

	auto func = [pw = std::make_unique<Widget>()]
		{ return pw->isValidated() && pw->isArchived(); };
		
可以通过C++11实现上述含有move的lambda效果

	class IsValAndArch {
	public:
		using DataType = std::unique_ptr<Widget>;
		explicit IsValAndArch(DataType&& ptr) : pw(std::move(ptr)) {} // use of std::move
		bool operator()() const
			{ return pw->isValidated() && pw->isArchived(); }
	private:
		DataType pw;
	};
	auto func = IsValAndArch(std::make_unique<Widget>());

或者使用C++11的lambda实现上述效果：

	1. 将对象move到std::bind产生的function对象中（被函数对象“捕获”）
	2. 使用by-reference capture捕获上述被“捕获”的对象

例如

	std::vector<double> data; 
	auto func = [data = std::move(data)] // C++14 init capture
			{ /* uses of data */ };
	auto func = std::bind( // C++11 实现版本
		[](const std::vector<double>& data) { /* uses of data */ },
		std::move(data)
	);

Std::bind返回bing object，第一个参数为callable object

一个bing object包含所有传入bind函数参数的拷贝，包括第一个lambda产生的closure参数的拷贝（这个closure的生存期和bind object一样），如果参数为lvalue使用copy ctor，rvalue使用move ctor

上述bind版本跟C++14版本区别在于bind版本的lambda中的data是一个指向bing object中data的lvalue reference（std::move(data)是rvalue，但是bind object中的data是lvalue）

默认closure class中operator()成员函数是const的，在C++14中lambda体内部所有data member都是const的，为了C++11版本bind object内的move-ctor的data不被修改，将参数声明为reference-to-const。若lambda声明为mutable，则无需使用const

	auto func = std::bind(
		[](std::vector<double>& data) mutable { /* uses of data */ },
		std::move(data)
	);

注意的几点：

	1. C++11中，不可能move-ctor一个对象到closure中，但是可以到bind object中
	2. C++11中，模拟move-capture的方法是将对象move ctor到一个bind object中，并将这个对象by reference传给lambda
	3. Closure和bind object生存期相同，可以将bind object中的对象视作closure中

对make_unique被捕获，C++14为

	auto func = [pw = std::make_unique<Widget>()] 
			{ return pw->isValidated() && pw->isArchived(); };

C++11的模拟为

	auto func = std::bind(
		[](const std::unique_ptr<Widget>& pw)
			{ return pw->isValidated() && pw->isArchived(); },
		std::make_unique<Widget>()
	);

Item34指出倾向于使用lambda代替bind，除了几个例外情况（例如C++11下bind模拟C++14特性）



### item 33: Use decltype on auto&& parameters to std::forward them.


C++14中引入了generic lambdas（lambda中可以使用auto进行参数指定）

	auto f = [](auto x){ return func(normalize(x)); };

这个lambda对应的closure class类似如下

	class SomeCompilerGeneratedClassName {
	public:
		template<typename T> 
		auto operator()(T x) const // auto return type item3
			{ return func(normalize(x)); }
		… // other closure class
	};

这里即使lambda的传入参数为rvalue，lambda仍然将x作为lvalue传给normalize函数。需要将x修改为uniref，并将其通过forward传给normalize函数，即

	auto f = [](auto&& x) { return func(normalize(std::forward<???>(x))); };
	
如果作为模板函数，使用std::forward<T>即可，在generic lambda中，需要使用decltype（传入lvalue时产生lvalue reference，传入rvalue时产生rvalue reference）

> If the expression e refers to a variable in local or namespace scope, a static member variable or a function parameter, then the result is that variable's or parameter's declared type
> e is lvalue -> decltype(e) is T&
> e is xvalue -> decltype(e) is T&&; 
> e is prvalue -> decltype(e) is T. 
> (wiki: decltype)

Item28中C++14的forward实现

	template<typename T> 
	T&& forward(remove_reference_t<T>& param) {
		return static_cast<T&&>(param);
	}

如果T被设置为Widget（T=Widget），那么为

	Widget&& forward(Widget& param) { // T = Widget
		return static_cast<Widget&&>(param); 
	}
	
如果T为Widget的右值引用

	Widget&& && forward(Widget& param) { // T = Widget&&
		return static_cast<Widget&& &&>(param); 
	}

即

	Widget&& && forward(Widget& param) { // T = Widget&&
		return static_cast<Widget&&>(param); 
	}

T=Widget和T=Widget&&完全一样，即使用rvalue reference或者non-reference类型实例化forward，结果一样

如果lvalue传给lambda，decltype(x)将customary type传给forward，如果rvalue传给lambda，decltype(x)使用与否结果一样，因此decltype(x)总能得到想要的结果。

因此正确的lambda为

	auto f = [](auto&& param) { return func(normalize(std::forward<decltype(param)>(param))); };

C++14的lambda可以是可变参数，即

	auto f = [](auto&&... params) { return func(normalize(std::forward<decltype(params)>(params)...)); };
	

### item 34: Prefer lambdas to std::bind

Bind最早在2005的TR1中引入，C++11进入std

倾向于使用lambda而不是bind

lambda比bind更可读

设置闹钟的lambda版本，从调用setSoundL内部的setAlarm函数开始记时

	using Time = std::chrono::steady_clock::time_point;
	enum class Sound { Beep, Siren, Whistle };
	using Duration = std::chrono::steady_clock::duration;
	void setAlarm(Time t, Sound s, Duration d);
	
	auto setSoundL = [](Sound s) {
		using namespace std::chrono;
		setAlarm(steady_clock::now() + hours(1), s, seconds(30));
	};

对应的bind版本

	using namespace std::chrono; // as above
	using namespace std::placeholders;
	
	auto setSoundB = std::bind(
		setAlarm,
		steady_clock::now() + 1h, // 错误，从调用bind函数开始记时
		_1,
		seconds(30)
	);

正确做法为（从调用setSoundB函数内部的setAlarm函数开始记时？？？）

	auto setSoundB = std::bind(
		setAlarm,
		std::bind( std::plus<steady_clock::time_point>(), steady_clock::now(), hours(1) ), // 正确
		_1,
		seconds(30)
	);

当setAlarm函数重载时，lambda仍然正确

	enum class Volume { Normal, Loud, LoudPlusPlus };
	void setAlarm(Time t, Sound s, Duration d);
	void setAlarm(Time t, Sound s, Duration d, Volume v);
	
	auto setSoundL = [](Sound s) {
		using namespace std::chrono;
		setAlarm(steady_clock::now() + 1h, s, 30s); // 调用三个参数版本
	};
	
bind错误

	auto setSoundB = std::bind(
		setAlarm,  // 错误，无法确定调用哪一个重载函数
		std::bind(std::plus<steady_clock::time_point>(), steady_clock::now(), 1h),
		_1,
		30s
	);
	
需要显式指定函数指针类型

	using SetAlarm3ParamType = void(*)(Time t, Sound s, Duration d);
	auto setSoundB = std::bind(
		static_cast<SetAlarm3ParamType>(setAlarm), 
		std::bind(std::plus<steady_clock::time_point>(), steady_clock::now(), 1h),
		_1,
		30s
	);

上述函数指针访问方法使得编译器更难将其inline处理，即setSoundL(Sound::Siren); 相比setSoundB(Sound::Siren);  更容易被inline处理

对于更加复杂的lambda

	auto betweenL = [lowVal, highVal] (int val) { return lowVal <= val && val <= highVal; }; //  C++11 版本
	auto betweenL = [lowVal, highVal] (const auto& val) { return lowVal <= val && val <= highVal; }; // C++14版本

bind版本为

	using namespace std::placeholders;
	auto betweenB = std::bind( // C++14
		std::logical_and<>(), 
		std::bind(std::less_equal<>(), lowVal, _1),
		std::bind(std::less_equal<>(), _1, highVal)
	);
	auto betweenB = std::bind( // C++11
		std::logical_and<bool>(),
		std::bind(std::less_equal<int>(), lowVal, _1),
		std::bind(std::less_equal<int>(), _1, highVal)
	);
	
Placeholder（例如_1，_2）加重理解困难

lambda对于by-value或是by-reference显式标出

	auto compressRateL = [w](CompLevel lev) { return compress(w, lev); }; // w以by-value形式存储

bind总是拷贝他的参数（如果是rvalue？？？）

bind将w参数以by-value形式存储在compressRateB中（w是可变的，以by-value还是by-reference存储会影响compressRateB的调用结果）

	using namespace std::placeholders;
	enum class CompLevel { Low, Normal, High }; 
	Widget compress(const Widget& w, CompLevel lev);
	Widget w;
	auto compressRateB = std::bind(compress, w, _1); 

如果想要以by-reference方式存储则使用

	auto compressRateB = std::bind(compress, std::ref(w), _1);

在C++14中，没有任何理由使用bind而不非lambda，但是在C++11中，仍然有两个例外需要使用bind：

	1. Move capture（item32）
	2. Polymorphic function objects.

Polymorphic function objects，bindobject的函数调用操作使用perfect forwarding，它可以接收任何类型（一些限制见item30），可用于将对象绑定至模板化函数调用

	class PolyWidget {
	public:
		template<typename T>
		void operator()(const T& param);
	};
	PolyWidget pw;
	auto boundPW = std::bind(pw, _1);
	boundPW(1930); // 将int传给PolyWidget::operator()
	boundPW(nullptr); // 将nullptr传给PolyWidget::operator()
	boundPW("Rosebud"); // 将string传给PolyWidget::operator()

C++14中对应做法为

	auto boundPW = [pw](const auto& param) { pw(param); };
	
