## chapter 5 Rvalue References, Move Semantics, and Perfect Forwarding

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
