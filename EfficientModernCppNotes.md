# Effective Modern C++ 抄书笔记


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


## Chapter 1: Deducing Types

C++98仅包含一种类型推导，即函数模板类型推导

C++11修改了函数模板类型推导，并加入了auto和decltype类型推导；C++14扩展了auto和decltype的使用场景

### Item 1: Understand template type deduction

函数模板和调用

```C++
template<typename T> 
void f(ParamType param);
f(expr);
```

编译器使用expr推导出T和ParamType（一般包含 const之类的装饰）的类型

```C++
template<typename T> 
void f(const T & param);
int x = 0;
f(x); // T -> int, ParamType -> const int &
```

对类型T的推导依赖于ParamType和expr，三种case为

	1. ParamType为指针或引用类型，但是不是universal reference（item24）
	2. ParamType为universal reference
	3. ParamType不是指针也不是引用

#### ParamType为指针或引用类型，但是不是universal reference（item24）

	1. 如果expr的类型是引用，忽略引用部分
	2. 然后将忽略引用的expr的类型与ParamType进行pattern-match来决定T

```C++
template<typename T>
void f(T& param); // param为引用类型
int x = 27;
const int cx = x;
const int& rx = x;
f(x); // T -> int, ParamType -> int&
f(cx); // T -> const int, ParamType -> const int&
f(rx); // T -> const int, ParamType -> const int&
```

因此将const对象传给T&参数的函数模板是安全的，推导出的T类型包含const

lvalue reference类型的param和rvalue reference类型的param效果相同

const T&类型的param

```C++
template<typename T>
void f(const T& param);  
int x = 27;
const int cx = x;
const int& rx = x;
f(x); // T -> int, ParamType -> const int&
f(cx); // T -> int, ParamType -> const int&
f(rx); // T -> int, ParamType -> const int&
```

指针类型的Param

```C++
template<typename T> 
void f(T* param);
int x = 27;
const int *px = &x;
f(&x); // T -> int, ParamType -> int* 
f(px); // T -> const int, ParamType -> const int*
```

#### ParamType为universal reference

universal reference声明类似rvalue reference，但是在传入lvalue时表现不同（item24）

如果expr是lvalue，T和ParamType推导为lvalue reference，唯一的T被推导为引用的情况；如果expr是rvalue，使用case1的情形

```C++
template<typename T> 
void f(T&& param);
int x = 27;
const int cx = x;
const int& rx = x;
f(x); // x为lvalue, T -> int&, ParamType -> int&
f(cx); // x为lvalue, T -> const int&, ParamType -> const int&
f(rx); // x为lvalue, T -> const int&, ParamType -> const int&
f(27); // x为rvalue, T -> int, ParamType -> int&&
```

universal reference的类型推导对于lvalue和rvalue的param表现不同，lvalue reference和rvalue reference的类型推导不适用于universal reference（item24）

#### ParamType既不是指针也不是引用

即pass-by-value，param会是传入参数的一个拷贝

	1. 如果expr是引用，忽略引用部分
	2. 忽略引用部分的expr的类型，如果是const或者volatile同样忽略（volatile只是用于实现设备驱动item40）

```C++
template<typename T>
void f(T param); 
int x = 27;
const int cx = x;
const int& rx = x;
f(x); // T -> int, ParamType -> int
f(cx); // T -> int, ParamType -> int
f(rx); // T -> int, ParamType -> int
```

const和volatile只有在pass-by-value时被忽略

```C++
template<typename T>
void f(T param); // param is still passed by value
const char* const ptr =  "Fun with pointers"; // 两个const
f(ptr); // T -> const char*, ParamType -> const char *，忽略了第二个const
```

#### Array arguments

数组类型和指针类型不一样，在特定场景下数组类型会decay成指向第一个元素的指针类型

函数的param不会是一个数组类型，如函数声明

```C++
void myFunc(int param[]); // 合法
```

数组声明会被decay为指针

```C++
void myFunc(int * param);
```

将数组类型传给by-value的param的模板

```C++
template<typename T>
void f(T param); 
const char name[] = "J. P. Briggs";
f(name); // name是指针类型，但是T -> const char *
```

虽然函数不能声明数组类型的param（合法但会被decay），但是可以声明数组引用

```C++
template<typename T>
void f(T& param); // template with by-reference parameter
f(name); // T -> const char [13], ParamType -> const char (&) [13]
```

注意推导出的T包含数组大小

```C++
template<typename T, std::size_t N> 
constexpr std::size_t arraySize(T (&)[N]) noexcept { 
	return N;
}
```

constexpr使得结果编译器可获得（item15），从而可以使用这个函数声明另一个数组

```C++
int keyVals[] = { 1, 3, 7, 9, 11, 22, 35 };
int mappedVals[arraySize(keyVals)];
std::array<int, arraySize(keyVals)> mappedVals; // modern C++方式
```

#### Function Arguments

不止数组类型可以decay为指针类型，函数类型也可以decay为函数指针，对于数组类型的类型推导对函数类型也适用

```C++
void someFunc(int, double); // 类型为void(int, double)
template<typename T>
void f1(T param); // pass by value
template<typename T>
void f2(T& param); // pass by ref
f1(someFunc); // param deduced as ptr-to-func, type is void (*) (int, double)
f2(someFunc); // param deduced as ref-to-func, type is void (&) (int, double) 
```

### Item 2: Understand auto type deduction

除了一个例外，auto和函数模板类型推导完全一样

存在一个可以模板类型推导和auto类型推导之间的直接映射

```C++
template<typename T>
void f(ParamType param);
f(expr);
```

使用expr推导出T和ParamType的类型

对于auto类型推导，auto相当于模板中 的T，变量的type specifier相当于ParamType

```C++
auto x = 27;
const auto cx = x;
const auto & rx = x;
```

类比函数模板类型推导，auto类型推导也有三种情况

	1. type specifier是指针或者引用，但不是uniref
	2. type specifier是uniref
	3. type specifier不是指针也不是引用

```C++
auto x = 27; // case 3
const auto cx = x; // cast 3
const auto& rx = x; // case 1
```

case 2

```C++
auto&& uref1 = x; // x is x and lvalue, uref1类型为int&
auto&& uref2 = cx; // cx is const int and lvalue, uref2类型为const int&
auto&& uref3 = 27; // 27 is int and rvalue，uref3类型为int&&
```

数组和函数类型

```C++
const char name[] =   "R. N. Briggs";
auto arr1 = name; // arr1类型为const char *
auto& arr2 = name; // arr2类型为const char (&) [13]

void someFunc(int, double); 
auto func1 = someFunc; // func1的类型为void (*) (int, double)
auto& func2 = someFunc; // func2的类型为void (&) (int, double)
```

auto类型推导不同于函数模板类型推导的情况

声明一个int类型变量

```C++
int x1 = 27; // C++ 98
int x2(27); // C++ 98

int x3 = { 27 }; // C++ 11
int x4 { 27 }; // C++ 11
```

如果使用auto声明这四个变量

```C++
auto x1 = 27; // type is int, value is 27
auto x2(27); // ditto
auto x3 = { 27 }; // type is std::initializer_list<int>, value is { 27 }
auto x4{ 27 }; // ditto
```

如果auto声明的初始化被包在大括号内，则推导类型为std::initializer_list，如果这个类型无法被推导（例如initializer_list中类型不同），则不能推导

```C++
auto x5 = { 1, 2, 3.0 }; // 不能推导出std::initializer_list<T>中的类型T
```

这其中包含两步推导，第一步是由于x5的初始化被包括在大括号内，x5就被推导为std::initializer_list；第二步推导出std::initializer_list<T>中的模板类型T（上例中这一步失败）

如果需要函数模板类型推导为initializer_list，则需要显式声明

```C++
auto x = { 11, 23, 9 }; // x类型为std::initializer_list<int>

template<typename T>
void f(T param); 
f({ 11, 23, 9 }); // error! can't deduce type for T

template<typename T>
void f(std::initializer_list<T> initList);
f({ 11, 23, 9 }); // T -> int, initList的类型为std::initializer_list<int>
```

另外，在C++14中，auto可以作为待推导的函数返回值类型（item3），lambda也可以将auto声明为param的类型。但是这些场景中auto使用模板类型推导而非auto类型推导

```C++
auto createInitList() {
	return { 1, 2, 3 }; // 错误，无法对{ 1, 2, 3 }进行类型推导
}

std::vector<int> v;
auto resetV = [&v](const auto& newValue) { v = newValue; }; // C++14
resetV({ 1, 2, 3 }); // 错误，无法对{ 1, 2, 3 }进行类型推导
```


### Item 3: Understand decltype

decltype一般直接返回传入名称或表达式的精确的类型

```C++
const int i = 0; // decltype(i) is const int

bool f(const Widget& w); // decltype(w) is const Widget&
				// decltype(f) is bool(const Widget&)
struct Point {
	int x, y; // decltype(Point::x) is int
}; 

Widget w; // decltype(w) is Widget
if (f(w)) … // decltype(f(w)) is bool

template<typename T> // simplified version of std::vector
class vector {
public:
	T& operator[](std::size_t index);
};
vector<int> v; // decltype(v) is vector<int>
if (v[0] == 0) … // decltype(v[0]) is int&
```

C++11中decltype的主要应用是函数返回值类型依赖于函数param类型的情况，例如元素类型为T的容器的operator[]一般返回T&，但是std::vector<bool>返回一个新的对象而非bool&（item6）

```C++
template<typename Container, typename Index>
auto authAndAccess(Container& c, Index i) -> decltype(c[i]) {
	…
	return c[i];
}
```

authAndAccess前的auto并不进行类型推导，而是表明使用C++11的trailing return type语法（函数返回值类型在参数列表之后紧跟->），这样函数参数可以用于指定返回值类型，如果返回值类型放在函数名以前，函数参数均未声明，无法获得

C++11允许推导single-statement lambda的返回值类型，C++14允许对所有lambda和函数的返回值类型进行类型推导

Item2指出使用auto作为返回值类型，编译器对其使用模板类型推导，如果在C++14中使用

```C++
template<typename Container, typename Index> // C++14;
auto authAndAccess(Container& c, Index i) { // 错误
	…
	return c[i]; // return type deduced from c[i]
}
```

根据item1中模板类型推导，对T的推导忽略类型的引用部分，这里auto为c[i]的忽略引用部分的类型，则

```C++
std::deque<int> d;
authAndAccess(d, 5) = 10; // d[5]类型为int&，auto为int，编译错误
```

应当使用decltype获得与c[i]类型完全一样的类型，即在C++14中

```C++
template<typename Container, typename Index> // C++14
decltype(auto) authAndAccess(Container& c, Index i) { // 正确，仍待改进
	…
	return c[i]; // c[i]类型为T&，返回值类型也为T&
}
```

需要返回值可以修改这个container，因此authAndAdd函数的参数container通过lvalue-reference-to-non-const传入，使得一个rvalue的容器不能绑定到这个参数上

将rvalue的容器绑定到函数参数上，会使得T&类型返回值在函数返回时成为悬空引用。不过需要传入rvalue的临时容器并返回该临时容器的元素的一个拷贝时，需要authAndAdd能够接受lvalue或者rvalue

```C++
std::deque<std::string> makeStringDeque(); // factory function
auto s = authAndAccess(makeStringDeque(), 5); // makeStringDeque()的返回值是一个rvalue
```

如果不使用重载，可以使用universal reference（item24）

```C++
template<typename Container, typename Index>  // C++14最终版本
decltype(auto) authAndAccess(Container&& c, Index i) {
	…
	return std::forward<Container>(c)[i];
}

template<typename Container, typename Index> // C++11最终版本
auto authAndAccess(Container&& c, Index i) -> decltype(std::forward<Container>(c)[i]) {
	…
	return std::forward<Container>(c)[i];
}
```

decltype也可以用在其他地方

```C++
Widget w;
const Widget& cw = w;
auto myWidget1 = cw; // myWidget1's type is Widget
decltype(auto) myWidget2 = cw; // myWidget2's type is const Widget&
```

Decltype的一些特殊情况

对一个name（是一个lvalue expression）使用decltype表明声明这个name的类型；如果是比name复杂得多的lvalue expression（大多数内在包含一个lvalue reference qualifier），decltype确保声明的类型是lvalue reference，即

> 如果lvalue expression（不是name）具有类型T，decltype声明类型为T&

```C++
int x = 0; // x是一个变量的name，decltype(x) -> int，decltype((x)) -> int&

decltype(auto) f1() { // C++14
	int x = 0;
	return x; // decltype(x) is int, so f1 returns int
}

decltype(auto) f2() { // C++14
	int x = 0;
	return (x); // decltype((x)) is int&, so f2 returns int&
}
```

使用decltype(auto)时候需要格外小心，确保推导出预期结果需要使用item4中的技术

### Item 4: Know how to view deduced types

#### 编译器诊断

声明一个未定义类模板，并查看x和y的类型

```C++
template<typename T> class TD; // TD == "Type Displayer"

TD<decltype(x)> xType; // elicit errors containing
TD<decltype(y)> yType; // x's and y's types
```

编译输出错误信息为

```
error: aggregate 'TD<int> xType' has incomplete type and cannot be defined
error: aggregate 'TD<const int *> yType' has incomplete type and cannot be defined
```

#### Runtime output

```C++
std::cout << typeid(x).name() << '\n';
std::cout << typeid(y).name() << '\n';
```

对于输出不同编译器形式不同

Std::type_info::name往往不一定正确

```C++
template<typename T>
void f(const T& param) {
	cout << "T = " << typeid(T).name() << '\n';
	cout << "param = " << typeid(param).name() << '\n';
}
std::vector<Widget> createVec(); // factory function
const auto vw = createVec();
if (!vw.empty()) f(&vw[0]); // call f
```

微软的编译器结果为

```
T = class Widget const *
param = class Widget const *
```

根据item1中所述，如果类型为引用，则引用部分被忽略，在此基础上如果含有const或者volatile，同样忽略，因此param的类型由const Widget * const &变为const Widget *

使用Boost.TypeIndex可以避免这个问题

```C++
#include <boost/type_index.hpp>
template<typename T>
void f(const T& param) {
	using boost::typeindex::type_id_with_cvr;
	cout << "T = " 
		<< type_id_with_cvr<T>().pretty_name() 
		<< '\n';
	cout << "param = " 
		<< type_id_with_cvr<decltype(param)>().pretty_name() 
		<< '\n';
}
```

调用前述代码，微软编译器输出结果为

```
T = class Widget const *
param = class Widget const * const &
```



## Chapter 2: auto

### Item 5

使用auto定义变量保证变量的初始化（使用initializer对变量进行类型推导）

使用auto可以省略复杂的类型声明

使用auto可以表示只有编译器知道的类型

```C++
// C++11
auto derefUPLess = [](const std::unique_ptr<Widget>& p1, const std::unique_ptr<Widget>& p2)  { return *p1 < *p2; };

// C++14
auto derefLess =[](const auto& p1, const auto& p2) { return *p1 < *p2; }; 
```

如果使用C++11的std::function则表示为
```C++
std::function<bool(const std::unique_ptr<Widget>&, const std::unique_ptr<Widget>&)> derefUPLess = 
	[](const std::unique_ptr<Widget>& p1, const std::unique_ptr<Widget>& p2) { return *p1 < *p2; };
```

并且使用auto声明的变量即是这个lambda的closure，而使用std::function声明的变量表示的closure是std::function模板实例化后的一个closure，对于任何signature他的大小是固定的，因此std::function的ctor为了存储这个closure允许进行堆内存分配。使得std::function声明的closure比auto对象closure消耗更多的空间；并且std::function相比auto声明的closure较难进行内联处理

auto可以防止错误的变量初始化，不需要担心变量的具体类型，防止类型不匹配问题

```C++
std::vector<int> v;
unsigned sz = v.size(); // 64位windows系统unsigned占32位，std::vector<int>::size_type占64位，产生错误

auto sz = v.size(); // sz的类型为std::vector<int>::size_type
```

对于循环体

```C++
std::unordered_map<std::string, int> m;

for (const std::pair<std::string, int>& p : m) 
{
}
```

上述写法中，unordered_map的成员应该为std::pair<const std::string, int>，因此每次迭代编译器会使用m中的一个成员（类型为std::pair<const std::string, int>）拷贝构造出一个（类型为const std::pair<std::string, int>）临时变量，并将p绑定到这个临时变量上

使用auto不会有这个问题

```C++
for (const auto& p : m)
{
}
```

### Item 6: Use the explicitly typed initializer idiom when auto deduces undesired types

Auto会获得不想要的类型

```C++
std::vector<bool> features(const Widget& w);
```

当vector<T>中类型不是bool时，operator[]返回对特定元素的引用T&，vector<bool>的operator[]函数返回std::vector<bool>::reference，使用auto带来问题（C++不允许对单个bit的引用，std::vector<bool>::reference是vector中bool类型元素的打包形式，每个元素一个bit，这个类型表现类似bool&）

```C++
bool highPriority = features(w)[5]; // highPriority类型为bool
```

返回std::vector<bool>::reference，并被隐式转换为bool

```C++
auto highPriority = features(w)[5]; // highPriority类型为std::vector<bool>::reference
```

调用完成后返回的类型为vector<bool>的临时变量被销毁，highPriority成为悬空引用

这里std::vector<bool>::reference是一种proxy class，用于模仿或扩充另一个类的功能，分为visible proxy（std::shared_ptr和std::unique_ptr）和invisible proxy（这里的std::vector<bool>::reference）

Expression template使用了这类模式来提高数值计算效率，例如矩阵加法

```C++
Matrix sum = m1 + m2 + m3 + m4;
```

并不一个一个的计算中间结果，而是返回一个Sum<Matrix, Matrix>的proxy class，最后在=赋值的时候使用Sum<Sum<Sum<Matrix, Matrix>, Matrix>, Matrix>来初始化sum矩阵（Eigen矩阵库中大量使用这种方式）

通用法则是，invisible proxy class和auto不能很好的共存

对于invisible proxy class，auto的主要问题是不能推断出需要的类型，但是问题并不在于auto，而是通过强制使用the explicitly typed initializer idiom来保证auto推导出正确类型

```C++
auto highPriority = static_cast<bool>(features(w)[5]);
auto sum = static_cast<Matrix>(m1 + m2 + m3 + m4);
```

the explicitly typed initializer idiom也可以用来计算强调构造一个特定的类型的对象（从不同类型的对象）

```C++
double calcEpsilon();
float ep = calcEpsilon(); // impliclitly convert double → float，但是这样并不能表明类型转换的故意性

auto ep = static_cast<float>(calcEpsilon()); // 表明故意进行了类型转换
```

或者使用值域为[0.0, 1.0]的double类型来表示某一个像素在c.size()分辨率下的行c中的相对位置，获得对应像素下标index的方式为

```C++
int index = d * c.size();
auto index = static_cast<int>(d * c.size()); // double转为int的故意性
```

注意，不要写成（不能体现类型转换的故意性）

```C++
auto ep = float(calcEpsilon());
auto index = int(d * c.size());
```



## Chapter 3: Moving to Modern C++

### Item 7: Distinguish between () and {} when creating objects

C++11初始化一个int类型

```C++
int x(0); 
int y = 0; 
int z{ 0 }; 
```

另外{}语法通常还可以写作（不特殊说明视作等同于直接使用{}版本）

```C++
int z = { 0 }; // 通常等同于braces-only版本
```

对于用户定义的类型，需要区分初始化还是赋值

```C++
Widget w1; // 调用默认构造函数
Widget w2 = w1; // 初始化，调用拷贝构造函数
w1 = w2; // 赋值，调用拷贝赋值
```

C++11引入uniform initialization（a single initialization syntax that can, at least in concept, be used anywhere and express everything），braced initialization is a syntactic construct.

```C++
std::vector<int> v{ 1, 3, 5 }; // v's initial content is 1, 3, 5
```

{}也可以用于对non-static data member进行初始化，也可以使用=初始化语法，但是不能使用()初始化语法

```C++
class Widget {
private:
	int x{ 0 }; // fine, x's default value is 0
	int y = 0; // also fine
	int z(0); // error!
};
```

**C++不允许在类中使用()初始化**的原因在于会和函数声明冲突（根据[blue的专栏](https://zhuanlan.zhihu.com/p/21102748)）

```C++
class Widget {
private:
	typedef int x;
int z(x); // 这是一个函数声明
};
```

{}可以用于不能用于uncopyable对象的初始化（例如std::atomic），也可以使用()语法，但是不能使用=初始化语法

```C++
std::atomic<int> ai1{ 0 }; // fine
std::atomic<int> ai2(0); // fine
std::atomic<int> ai3 = 0; // 错误
```

对于C++的三种指定初始化表达式的方式，只有{}可以通用，因此称作uniform

{}初始化禁止built-in类型的narrowing conversions，()初始化和=初始化允许（为了兼容历史代码）

```C++
double x, y, z;
int sum1{ x + y + z }; // 错误
int sum2(x + y + z); // 可行
int sum3 = x + y + z; // 可行
```

另外{}初始化对C++的most vexing parse免疫，即C++要求任何能被parse为声明的东西均被解释为一个声明

```C++
Widget w1(10); // 调用Widget的构造函数，参数为10
Widget w2(); // most vexing parse，声明一个名为w2返回值为Widget的函数
Widget w3{}; // 调用Widget的构造函数，没有参数（函数不能使用{}声明）
```

没有引入initializer_list时，()和{}初始化相同

```C++
class Widget {
public:
	Widget(int i, bool b); 
	Widget(int i, double d); 
};

Widget w1(10, true); // calls first ctor
Widget w2{10, true}; // also calls first ctor
Widget w3(10, 5.0); // calls second ctor
Widget w4{10, 5.0}; // also calls second ctor
```

当声明了使用initializer_list为参数的构造函数之后，一旦有任何方式使得编译器能够调用initializer_list为参数的构造函数，编译器就会使用这个构造函数

```C++
class Widget {
public:
	Widget(int i, bool b); 
	Widget(int i, double d); 
	Widget(std::initializer_list<long double> il); // added
};

Widget w1(10, true); // calls first ctor
Widget w2{10, true}; // 调用std::initializer_list ctor (10和true均转换为long double)
Widget w3(10, 5.0); // calls second ctor
Widget w4{10, 5.0}; // 调用std::initializer_list ctor // (10和5.0转换为long double)
```

initializer_list为参数的构造函数甚至会劫持copy ctor或者move ctor

```C++
class Widget {
public:
	Widget(int i, bool b); 
	Widget(int i, double d); 
	Widget(std::initializer_list<long double> il); 
	operator float() const; // convert to float
};

Widget w5(w4); // 调用calls copy ctor
Widget w6{w4}; // w4转换为float，再调用std::initializer_list ctor
Widget w7(std::move(w4)); // 调用move ctor
Widget w8{std::move(w4)}; // std::move(w4)转换为float，再调用std::initializer_list ctor
```

即使在最匹配的std::initializer_list constructor不能被调用时，仍然会匹配std::initializer_list constructor

```C++
class Widget {
public:
	Widget(int i, bool b); 
	Widget(int i, double d); 
	Widget(std::initializer_list<bool> il); // bool类型的initializer_list
	// 没有隐式类型转换
};

Widget w{10, 5.0}; // 错误，requires narrowing conversions
```

首先由于存在10和5.0到bool类型的转换（是一种narrow conversion），因此调用initializer_list ctor；其次，{}初始化不能进行narrowing conversion（将10和5.0转换为bool），因此错误

只有当{}中的类型无法被转换为initializer_list中的类型时，才会使用普通的重载决议

```C++
class Widget {
public:
	Widget(int i, bool b); 
	Widget(int i, double d); 
	Widget(std::initializer_list<std::string> il); // string类型的initializer_list
}; 

Widget w1(10, true); // 调用first ctor
Widget w2{10, true}; // 调用calls first ctor
Widget w3(10, 5.0); // 调用second ctor
Widget w4{10, 5.0}; // 调用second ctor
```


对于空的{}初始化，如果意为提供空参数，应该调用默认构造函数，如果意为提供一个空的{}初始化，那么应该调用initializer_list ctor版本。事实上空的{}意为没有实参，而非一个空的initializer_list；如果需要使用空的{}作为实参调用initializer_list ctor版本，需要提供{{}}参数

```C++
class Widget {
public:
	Widget(); // default ctor
	Widget(std::initializer_list<int> il); // std::initializer
};

Widget w1; // calls default ctor
Widget w2{}; // calls default ctor
Widget w3(); // most vexing parse! declares a function!
Widget w4({}); // calls std::initializer_list ctor with empty list
Widget w5{{}}; // ditto
```

对于std::vector需要注意

```C++
std::vector<int> v1(10, 20); // use non-std::initializer_list ctor，10个元素的vector，每个都是20
std::vector<int> v2{10, 20}; // use std::initializer_list ctor，两个元素，10和20
```

作为类的设计者，需要使得使用{}或者()初始化效果一致，在对一个类添加initializer_list ctor时，注意原有的对普通ctor的调用会转而使用对initializer_list ctor的调用

有时模板的作者也不知道具体应当使用()还是{}，只有调用者知道（例如std::make_unique和std::make_shared，这两个函数模板内部使用()，并在文档中详细说明）

```C++
template<typename T, typename... Ts>
void doSomeWork(Ts&&... params) {
	// create local T object from params… 可选择()或者{}
	T localObject(std::forward<Ts>(params)...); // using parens
	T localObject{std::forward<Ts>(params)...}; // using braces
}

doSomeWork<std::vector<int>>(10, 20); // doSomeWork模板的作者也不知道调用者的具体意图
```

### Item 8: Prefer nullptr to 0 and NULL

0是一个int类型，NULL可以是一个整数类型（例如int或者long），这两个都不是指针类型，这会在重载函数决议时调用非指针类型的函数

Nullptr的真实类型是std::nullptr_t，std::nullptr_t被隐式转换为所有裸指针类型，因此nullptr表现出任意类型的指针

std::nullptr_t也定义为nullptr类型

nullptr使程序可读

```C++
auto result = findRecord( /* arguments */ );
if (result == 0) { … } // 不清楚result是否是指针类型
if (result == nullptr) { … } // result是指针类型
```

使用0或者NULL在模板中进行类型推导时会产生错误


### Item 9: Prefer alias declarations to typedefs

对于typedef和alias declaration，声明一个函数（接受一个int和const std::string &，返回值void）

```C++
typedef void (*FP)(int, const std::string&); // typedef
using FP = void (*)(int, const std::string&); // alias declaration
```

对于模板，使用alias declaration可以被模板化（alias templates），typedef不可以。例如对std::list<T, MyAlloc<T>>的模板，alias declaration版本为

```C++
template<typename T> 
using MyAllocList = std::list<T, MyAlloc<T>>; 

MyAllocList<Widget> lw;
```

typedef版本为

```C++
template<typename T> 
struct MyAllocList {
	typedef std::list<T, MyAlloc<T>> type; 
};
MyAllocList<Widget>::type lw;
```

依赖于模板类型参数（T）的类型（MyAllocList<T>::type而非MyAllocList<T>）是dependent type，C++要求dependent type需要前置typename

```C++
template<typename T> 
struct MyAllocList {
	typedef std::list<T, MyAlloc<T>> type; 
};

template<typename T>
class Widget { 
private: 
	typename MyAllocList<T>::type list; // as a data member
};
```

使用alias template变得简单

```C++
template<typename T>
using MyAllocList = std::list<T, MyAlloc<T>>; 

template<typename T>
class Widget {
private:
	MyAllocList<T> list; // no "typename", no "::type"
};
```

上述例子中，编译器在处理使用alias template定义的MyAllocList<T>时已经知道这是一个类型（因为这是一个alias template，一个alias template只能表示一个类型），因此是一个non-dependent type；但是使用嵌套的typedef定义的MyAllocList<T>::type时，不能知道这表示一个类型（因为只有MyAllocList被实例化了才能看到MyAllocList内部的MyAllocList<T>::type是一个名字或者一个成员变量），因此是一个dependent type

例如

```C++
class Wine { … };

template<>
class MyAllocList<Wine> { 
private:
	enum class WineType { White, Red, Rose }; 
	WineType type; // in this class, type is a data member!
};
```

C++11的头文件<type_traits>中定义了许多traits，例如

```C++
std::remove_const<T>::type // yields T from const T
std::remove_reference<T>::type // yields T from T& and T&&
std::add_lvalue_reference<T>::type // yields T& from T
```

出于历史原因，这些都是用结构体模板中嵌套的typedef而非alias template实现的（注意那些::type），使用时都要在前面添上typename

C++14使用alias template提供了上述这些功能

```C++
std::remove_const<T>::type // C++11: const T → T
std::remove_const_t<T> // C++14 equivalent

std::remove_reference<T>::type // C++11: T&/T&& → T
std::remove_reference_t<T> // C++14 equivalent

std::add_lvalue_reference<T>::type // C++11: T → T&
std::add_lvalue_reference_t<T> // C++14 equivalent
```

在C++11中自行定义这些alias template也很简单

```C++
template <class T>
using remove_const_t = typename remove_const<T>::type;

template <class T>
using remove_reference_t = typename remove_reference<T>::type;

template <class T>
using add_lvalue_reference_t = typename add_lvalue_reference<T>::type;
```

### Item 10: Prefer scoped enums to unscoped enums

通常大括号内声明的name仅在大括号内可见，但是C++98风格的enum属于包含这个enum的作用域，会造成名称泄露，称为unscoped enum

```C++
enum Color { black, white, red }; 
auto white = false; // 错误，white已声明
```

C++11的scoped enum不会造成这种问题

```C++
enum class Color { black, white, red };
auto white = false;
Color c = white; // 错误，white未声明
Color c = Color::white; // 可行
auto c = Color::white; // 可行
```

unscoped enum会被隐式转化为整数类型（并可能接着转为浮点类型），scoped enum不会发生隐式类型转换

```C++
enum Color { black, white, red }; 
Color c = red;
if (c < 14.5) { … } // 可行，Color和double进行比较
```

如果scoped enum进行这类类型转化，可以使用static_cast

```C++
enum class Color { black, white, red }; 
Color c = Color::red; 
if (c < 14.5) { … } // 错误
if (static_cast<double>(c) < 14.5) { // odd code, but it's valid
	auto factors = primeFactors(static_cast<std::size_t>(c)); // suspect, but it compiles
}
```

Scoped enum可以前向声明

```C++
enum class Color; // 可行
```

但是在C++11对unscoped enum做前向声明必须做额外工作（C++98仅允许unscoped enum的定义，不允许前向声明）

对于每个unscoped enum，编译器需要使用其的underlying type，不同的unscoped enum的underlying type大小不同，（编译器通常选择最小的能够覆盖所有enum的类型，但有时为了优化速度而非大小选择其他更大的类型）

C++98不支持前向声明确保了先对unscoped enum选择underlying type，再使用这个类型。但是unscoped enum类型的往往被include在一个头文件中，使得类型的一个修改会导致大量的重新编译（修改会使得underlying type发生变动？？）

对于C++11中的scoped enum的默认underlying type为int，也可以自行声明大小

```C++
enum class Status: std::uint32_t { 
	good = 0,
	failed = 1,
	incomplete = 100,
	corrupt = 200,
	audited = 500,
	indeterminate = 0xFFFFFFFF
};
```

C++11中unscoped enum的前向声明要指定underlying type

```C++
enum Color: std::uint8_t;
```

仍然有一个unscoped enum有用处的例子，（可以使用scoped enum解决），即当访问std::tuple时

```C++
using UserInfo = std::tuple<std::string, std::string, std::size_t> ; // 一个用户的name，email，reputation
enum UserInfoFields { uiName, uiEmail, uiReputation };
UserInfo uInfo; 
auto val = std::get<uiEmail>(uInfo); // 通过unscoped enum访问UserInfo这个tuple（std::get模板参数要求编译期确定）
```

如果使用scoped enum，可以使用

```C++
enum class UserInfoFields { uiName, uiEmail, uiReputation };
UserInfo uInfo; 
auto val = std::get<static_cast<std::size_t>(UserInfoFields::uiEmail)>(uInfo);
```

或者写一个编译期函数toUType，利用std::underlying_type这个type trait

```C++
// C++11
template<typename E>
constexpr 
typename std::underlying_type<E>::type // 代替std::size_t
toUType(E enumerator) noexcept {
	return static_cast<typename std::underlying_type<E>::type>(enumerator);
}

// C++14
template<typename E> // C++14
constexpr 
std::underlying_type_t<E> // 对应C++11中的typename std::underlying_type<E>::type
toUType(E enumerator) noexcept {
	return static_cast<std::underlying_type_t<E>>(enumerator);
}

// C++14
template<typename E> // C++14
constexpr 
auto // 使用auto返回类型
toUType(E enumerator) noexcept {
return static_cast<std::underlying_type_t<E>>(enumerator);
}
```

调用方式为

```C++
auto val = std::get<toUType(UserInfoFields::uiEmail)>(uInfo);
```

### Item 11: Prefer deleted functions to private undefined ones

所有istream和ostream的基类basic_ios不允许进行copy operation

C++98中定义为

```C++
template <class charT, class traits = char_traits<charT> >
class basic_ios : public ios_base {
public:
	…
private:
	basic_ios(const basic_ios& ); // not defined
	basic_ios& operator=(const basic_ios&); // not defined
};
```

C++11中定义为

```C++
template <class charT, class traits = char_traits<charT> >
class basic_ios : public ios_base {
public:
	…
	basic_ios(const basic_ios& ) = delete;
	basic_ios& operator=(const basic_ios&) = delete;
};
```

如果使用了这些函数，C++98的做法会在link-time报错，C++11会在编译期报错；C++11中将deleted function声明为public，如果为private报错信息中可能只包含访问private而不包含delete

deleted function可以用于任意函数，而只有成员函数才能为private。例如一个判断是否幸运数字的程序

```C++
bool isLucky(int number); // original function
bool isLucky(char) = delete; // reject chars
bool isLucky(bool) = delete; // reject bools
bool isLucky(double) = delete; // reject doubles and floats
```

Deleted function会参与重载决议，即上述例子中float参数会调用重载函数isLucky(double)而非isLucky(int)

Deleted function还可以用于防止模板实例化

```C++
template<typename T>
void processPointer(T* ptr);

template<>
void processPointer<void>(void*) = delete; // 防止对void*参数的实例化

template<>
void processPointer<const void>(const void*) = delete; 

template<>
void processPointer<const void>(const volatile void*) = delete;
```

成员函数模板不能被指定不同的access level（模板实例化需要写在名空间作用域中，而非类作用域中），因此不能通过对模板实例化声明private来防止特定的实例化

```C++
class Widget {
public:
	template<typename T>
	void processPointer(T* ptr) { … }
private:
	template<> // error!
	void processPointer<void>(void*);
};
```

正确做法是

```C++
class Widget {
public:
	template<typename T>
	void processPointer(T* ptr) { … }
};
template<>
void Widget::processPointer<void>(void*) = delete;
```

### Item 12: Declare overriding functions override


C++98中override发生条件：

	1. 基类函数为虚函数
	2. 基函数和派生函数（除了dtor）的名称完全相同
	3. 基函数和派生函数的形参类型完全相同
	4. 基函数和派生函数的constness完全相同
	5. 基函数和派生函数的返回值类型和exception specification必须兼容

C++11中添加了一个：函数的reference qualifier必须相同

函数的reference qualifier用于根据*this是lvalue还是rvalue约束函数的调用条件

```C++
class Widget {
public:
	void doWork() &; // 仅当*this为lvalue时可用
	void doWork() &&; // 仅当*this为rvalue时可用
};

Widget makeWidget(); // factory function (returns rvalue)
Widget w; // normal object (an lvalue)

w.doWork(); // calls Widget::doWork for lvalues (i.e., Widget::doWork &)
makeWidget().doWork(); // calls Widget::doWork for rvalues (i.e., Widget::doWork &&)
```

如果显式写出override，会避免因写错导致的本因override的函数没有被override

C++11引入两个contextual keywords，override和final，这两个keyword只在特定场景中产生作用。只有override出现在成员函数声明的最后才有含义，历史代码中已经使用的override并不需要为了适应C++11而更改；只有final使用在虚函数和类型上才有特定作用，使用在虚函数上防止该虚函数被派生类的函数override，使用在类上防止这个类被作为他与类型的基类

```C++
class Warning { // potential legacy class from C++98
public:
	void override(); // legal in both C++98 and C++11
};
```

#### 补充Reference qualifier的使用场景

给出一个Widget类，内含成员变量std::vector<double> values，需要使用data()成员函数获取内部values

```C++
class Widget {
public:
	using DataType = std::vector<double>;
	DataType& data() { return values; }
private:
	DataType values;
};
```

调用情形为

```C++
Widget w; // lvalue
auto vals1 = w.data(); // 将w.values拷贝至vals1

Widget makeWidget(); // 工厂类，返回rvalue
auto vals2 = makeWidget().data(); // 将values拷贝值vals2
```

上述最后一行，调用rvalue的data()成员函数，由于data()成员函数返回一个lvalue reference，C++要求编译器产生拷贝操作。而对于rvalue，获取其values成员变量只需要move即可，因此根据*this是lvalue还是rvalue区分对待

```C++
class Widget {
public:
	using DataType = std::vector<double>;
	DataType& data() & // for lvalue Widgets,
		{ return values; } // return lvalue
	DataType data() && // for rvalue Widgets,
		{ return std::move(values); } // return rvalue
private:
	DataType values;
};
```

这样调用情形为

```C++
auto vals1 = w.data(); // 调用lvalue重载版本，拷贝构造vals1
auto vals2 = makeWidget().data(); // 调用rvalue重载版本，移动构造vals2
```

### Item 13: Prefer const_iterators to iterators

C++11下使用const_iterator很简单

```C++
std::vector<int> values; 
auto it = std::find(values.cbegin(),values.cend(), 1983);
values.insert(it, 1998);
```

如果要实现最大程度的泛型，需要考虑到一些库或是类似容器的数据结构提供non-member function的begin/end等函数（C++11为数组提供了non-member begin，返回指向数组的首个元素的指针）。C++11只包含non-member function的begin和end；C++14包含non-member function的cbegin, cend, rbegin, rend, crbegin, crend，使用这些函数实现泛型的findAndInsert

```C++
// C++14
template<typename C, typename V>
void findAndInsert(C& container, const V& targetVal, const V& insertVal) { 
	using std::cbegin; 
	using std::cend;
	auto it = std::find(cbegin(container), // non-member cbegin
				cend(container), // non-member cend
				targetVal);
	container.insert(it, insertVal);
}
```

在C++11中模拟non-member cbegin，如果容器是const的，那么元素也是const的，使用begin就可以获得const_iterator

```C++
template <class C>
auto cbegin(const C& container)->decltype(std::begin(container)) {
	return std::begin(container); 
}
```

### Item 14: Declare functions noexcept if they won’t emit exceptions.

C++98中，需要制定函数可能抛出异常的类型，对函数的修改可能会使得抛出异常的类型不同，导致调用方的失败

C++11中要求函数声明可能会或者不会抛出异常，无条件的noexcept表面函数保证不抛出异常

函数的noexcept是函数接口的一部分

给函数添加noexcept可以允许编译器生成更好的目标代码

```C++
int f(int x) throw(); // no exceptions from f: C++98 style
int f(int x) noexcept; // no exceptions from f: C++11 style
```

对于上述函数f，如果运行期间抛出异常，则违反函数的异常指定。在C++98的异常指定下，对于f的调用者，函数调用栈unwound；在C++11下，只是可能unwound

在noexcept函数中，如果一个异常propagate到函数外，在unwindable状态下优化器不需要维护运行期的调用栈，也不需要在异常离开函数时，保证这个noexcept函数中的对象按照构造的逆序销毁。C++98的throw()函数缺乏这一优化弹性，总结为

```C++
RetType function(params) noexcept; // most optimizable
RetType function(params) throw(); // less optimizable
RetType function(params); // less optimizable
```

对于某些函数（例如move和swap），是否声明noexcept差别巨大

std::vector<Widget>，通过push_back添加元素。C++98下分配空间，并将旧元素拷贝至新的位置，并销毁旧的内存，push_back可以提供强异常安全保证，即拷贝元素期间抛出任何异常，旧的内存空间没有被破坏；C++11下如果将拷贝改为移动，就不能提供强异常安全保证了（移动期间抛出异常时旧内存空间已经被修改）

已有代码可能会依赖于C++98中push_back的强异常安全保证，因此C++11不能直接将内部操作由拷贝替换成移动，而是使用“move if you can, but copy if you must”策略，这一策略也同时应用于许多标准库中（std::vector::reverse和std::deque::insert），当移动操作声明为noexcept时，使用移动替换拷贝操作

swap通常使用copy assignment实现，标准库中的swap的noexcept性质通常依赖于用户定义的swap的noexcept性质

```C++
template <class T, size_t N>
void swap(T (&a)[N], T (&b)[N]) noexcept(noexcept(swap(*a, *b)));

template <class T1, class T2>
struct pair {
    void swap(pair& p) noexcept(noexcept(swap(first, p.first)) &&
                                 noexcept(swap(second, p.second)));
    ...
};
```

大部分函数是exception-neutral的，他们本身并不抛出异常，但是会调用一些可能抛出异常的函数

程序的正确性比优化更重要，不能为了noexcept带来的优化而影响功能的实现，例如不能为了声明noexcept而在函数内部处理本该抛出到函数外的异常

对于某些函数，noexcept非常重要，默认即是隐式声明了noexcept的。在C++98中允许内存释放函数（operator delete和operator delete[]）和析构函数抛出异常时很不好的行为；在C++11中这一点直接升级为语言规则，即无论是编译器产生或是自定义的所有内存释放函数和所有析构函数都隐式声明了noexcept，除非类的一个成员（包括继承成员以及包含在其他成员中的成员）可能抛出异常（声明为“noexcept(false)”）。标准库中没有可能抛出异常的内存释放函数和析构函数，并且标准库使用的析构函数（在容器内或者被传给algorithm）抛出异常为未定义行为

函数可以分为wide contracts和narrow contracts。wide constract函数没有preconditions，这类函数不会产生未定义行为；narrow contract函数有preconditions，如果这些precondition不能满足则会产生未定义行为

假设函数f接受std::string，并有一个precondition为string的长度不超过32，那么当string的长度超过32则产生未定义行为，函数f没有检查这个precondition的义务，可以声明为

```C++
void f(const std::string& s) noexcept; // precondition: s.length() <= 32
```

但是f的实现也可以检查这个precondition并在不满足时抛出异常“precondition was violated”，这在进行测试的时会易于debug，因此对于函数声明的noexcept，通常只对wide contract函数保留这个noexcept


编译器并不会识别函数实现和函数exception specification的统一性

```C++
void doWork() noexcept {
  setup();
  cleanup();
}
```

如果setup和cleanup没有声明noexcept，但是文档中表示他们不会抛出异常，doWork也可以声明noexcept。因为setup和cleanup有可能通过C实现（从C标准库中加入std名空间中的函数也缺少exception specification，例如std::strlen没有被声明为noexcept），或者是没有为C++11重写的来自于C++98的没有使用exception specification的库函数

因此noexcept函数依赖于没有声明noexcept函数的行为是合法的，编译器也不会产生警告

### Item 15: Use constexpr whenever possible

constexpr表示一个变量在编译期可知。
Lest I
ruin the surprise ending, for now I’ll just say that you can’t assume that the results of
constexpr functions are const, nor can you take for granted that their values are
known during compilation. Perhaps most intriguingly, these things are features. It’s
good that constexpr functions need not produce results that are const or known
during compilation!

constexpr对象是const的，并且他的值在编译期可知（其实值是在translation阶段确定，translation阶段包括编译和链接两个步骤）

编译期可知的值可以被放入只读内存

const类型的编译期可知的整形变量可以使用在integral contant expression场景中，包括数组大小指定，整形模板参数（std::array的长度），枚举值，alignment specifier等等

所有constexpr对象都是const的，反之不成立


使用compile-time constants参数调用constexpr函数时，产生compile-time constants

	1. constexpr函数可以用在需要compile-time constants的场景中，如果实参是compile-time constants，则结果是compile-time constants
	2. 如果constexpr函数的参数为非编译期可知的值，则其表现成一个普通函数

对于3的n次方的实现，std::pow进应用于浮点类型，并且不是constexpr。自定义constexpr的pow

```C++
constexpr int pow(int base, int exp) noexcept;
```

自定义pow的返回值不是const的，因此当参数编译期可知时，返回值是compile-time constant；如果参数编译期不可知，则变为普通函数

C++11中constexpr函数不能包含多于一个可执行语句，自定义pow为

```C++
constexpr int pow(int base, int exp) noexcept {
	return (exp == 0 ? 1 : base * pow(base, exp - 1));
}
```

C++14版本为

```C++
constexpr int pow(int base, int exp) noexcept {
	auto result = 1;
	for (int i = 0; i < exp; ++i) result *= base;
	return result;
}
```

constexpr函数被限制为只能接受和返回literal type

C++11中除了void所有内建类型都是literal类型，并且自定义类型也可以是literal type（如果类的ctor和其他成员函数都是constexpr函数，则这个类也是literal type）

C++11的Point版本

```C++
class Point {
public:
	constexpr Point(double xVal = 0, double yVal = 0) noexcept
		: x(xVal), y(yVal) {}
	
	constexpr double xValue() const noexcept { return x; }
	constexpr double yValue() const noexcept { return y; }
	
	void setX(double newX) noexcept { x = newX; } // 不是constexpr
	void setY(double newY) noexcept { y = newY; } // 不是constexpr

private:
	double x, y;
};

constexpr Point p1(9.4, 27.7); // fine
constexpr Point p2(28.8, 5.3); // also fine

constexpr
Point midpoint(const Point& p1, const Point& p2) noexcept {
	return { (p1.xValue() + p2.xValue()) / 2, (p1.yValue() + p2.yValue()) / 2 }; 
}
constexpr auto mid = midpoint(p1, p2);
```

上面C++11的Point例子中setter函数不是constexpr，因为C++11中void不是literal type，并且C++11中constexpr成员函数隐式const

C++14中没有这些限制，Point版本为

```C++
class Point {
public:
	…
	constexpr void setX(double newX) noexcept { x = newX; } // C++14
	constexpr void setY(double newY) noexcept { y = newY; } // C++14
	…
};

constexpr Point reflection(const Point& p) noexcept { // C++14
	Point result; // create non-const Point
	result.setX(-p.xValue()); // set its x and y values
	result.setY(-p.yValue());
	return result; // return copy of it
}

constexpr Point p1(9.4, 27.7); 
constexpr Point p2(28.8, 5.3);
constexpr auto mid = midpoint(p1, p2);
constexpr auto reflectedMid = reflection(mid);
```

constexpr应该是一个对象或者函数的接口的一部分，constexpr表明可以被用在需要一个constant expression的场景中，如果将原来应用constexpr的程序中的constexpr删去，很多程序将会不能编译（I/O通常不被允许出现在constexpr函数中，因此为了调试等原因将I/O加入函数往往会出现问题）

### Item 16: make const member functions thread safe

Const 成员函数可以对mutable成员变量作修改，需要保护mutable变量的线程安全

```C++
class Polynomial {
public:
	using RootsType = std::vector<double>;
	RootsType roots() const { // 需要保证rootsAreValid和rootVals两个mutable变量的线程安全
		if (!rootsAreValid) { // if cache not valid
			… 
			rootsAreValid = true;
		}
		return rootVals;
	}
private:
	mutable bool rootsAreValid{ false }; 
	mutable RootsType rootVals{};
};
```

使用std::lock_guard和std::mutex，std::mutex只可以被move不能被copy，会导致Polynomial只能被move不能被copy

```C++
class Polynomial {
public:
	using RootsType = std::vector<double>;
	RootsType roots() const {
		std::lock_guard<std::mutex> g(m); // lock mutex
		if (!rootsAreValid) { // if cache not valid
			… 
			rootsAreValid = true;
		}
		return rootVals;
	} // unlock mutex
private:
	mutable std::mutex m;
	mutable bool rootsAreValid{ false };
	mutable RootsType rootVals{};
};
```


对于一个需要synchronization的变量或内存地址，std::atomic够用；对于需要整体看做一个单元的多个变量或者内存地址，需要使用mutex

### item 17: understand special member function generation

类的自动生成函数

C++98中包含：默认构造函数，默认析构函数，拷贝构造函数，拷贝赋值函数
C++11中包含：移动构造函数，移动赋值函数

move con执行member wise move（不强制要求，为了兼容 not move-enable c++98）

copy con 和 copy assign 独立生成（没有显式声明）
move con 和 move assign 同时生成（没有任何一个显式move con或move assign声明）

显式声明copy operation会阻止move operation生成；显式声明move operation会阻止copy operation生成；这两点都意味着普通member wise op并不能满足要求

> rule of three：声明copy con/copy assign/destructor中的任何一个，就需要声明全部三个（参与资源管理）

显式声明destructor不对copy op产生影响（98 11），应该加以注意

move op自动生成条件：

	1. no copy op
	2. no move op
	3. no destructor

成员模板函数不会影响这些特殊成员函数的生成







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

若在一个树中，父节点拥有子节点的所有权，那么父节点通过unique_ptr指向子节点，子节点通过raw ptr指向父节点即可；如果所有权并不是如此严格，那么需要考虑weak_ptr

Weak_ptr和shared_ptr占用同样大小空间，复用shared_ptr的control block

### Item 21: Prefer std::make_unique and std::make_shared to direct use of new.

Make_shared是c++11，make_unique是c++14，自定义make_unique为

```C++
template<typename T, typename... Ts>
std::unique_ptr<T> make_unique(Ts&&... params)
{
	return std::unique_ptr<T>(new T(std::forward<Ts>(params)...));
}
```
	
make_unique只是将参数完美转发给构造函数（不支持数组和自定义deleter），不要将其放入std命名空间中

三个make函数（传入参数集合，完美转发给动态创建的对象，并返回智能指针）为make_unique，make_shared，allocate_shared（该函数接收allocator object用于动态内存分配）

#### 多数情况下使用make函数而非智能指针创建函数

	1. 少打一个类型名称
	2. 类型安全
	3. 效率（对make_shared和allocate_shared而言）

少打一个类型名称的情况

```C++
auto upw1(std::make_unique<Widget>()); // with make func
std::unique_ptr<Widget> upw2(new Widget); // without make func 
auto spw1(std::make_shared<Widget>()); // with make func
std::shared_ptr<Widget> spw2(new Widget); // without make func
```

类型安全的情况

```C++
processWidget(std::shared_ptr<Widget>(new Widget), computePriority());
```

编译器按照 new Widget，computePriority()，std::shared_ptr<>()的顺序创建，则computePriority产生异常则会使new Widget内存泄露
	
效率（对make_shared和allocate_shared而言）
	
Shared_ptr<W> spw(new W)分配两次内存（W类型和control block）
	
make_shared<W>()允许编译器优化为分配一次内存（W类型和control block分配在一起）

#### 某些情况下不使用make函数（对于make_unique和make_shared而言）

	1. 不允许自定义deleter
	2. 句法细节
	
句法细节

（item7）通过大括号创建的对象倾向于使用std::initializer_list参数的ctor，小括号倾向于non-std::initializer_list的ctor。在make函数中使用完美转发处理这些参数，倾向于使用小括号的ctor；即需要braced initializer的地方需要使用new+braced initializer；（item30）braced initializer不能被完美转发，但是可以通过auto类型推导获得initializer_list对象，并传给make函数

```C++
auto initList = { 10, 20 };
auto spv = std::make_shared<std::vector<int>>(initList);
```

#### 某些情况下对于make_shared而言还存在更多的问题

	1. 对于重载了op new和op delete的类，不要使用make函数（这些往往被设计为处理sizeof(W)大小的内存，但是make函数会分配sizeof(W)+sizeof(control block)大小的内存）
	2. 使用make_shared函数，当引用计数为0，调用对象的dtor，但是需要等到control block的内存被释放时才能释放整个占用的内存
	3. Control block上含有ref count和weak count，weak_ptr通过检查ref count（而非weak count）判断是否expired，如果ref count为0则调用dtor，weak_ptr已经expired。只要weak_ptr联系了control block（weak count > 0），那么control block需要存在，即make函数分配的内存（包括已经销毁的对象和需要存在的control block）仍然不能被释放（直到所有shared_ptr和weak_ptr都销毁）。即当对象很大并且销毁最后一个shared_ptr和weak_ptr之间的时间很长，使用make_shared会出现内存问题；而直接使用new语句并构造shared_ptr不会出现这种问题

在不能使用make函数时，最好做法是

> new语句创建指针并在一个语句中立即将其传给smart point的构造函数，并且这个语句中不做任何别的事情。

```C++
processWidget(std::shared_ptr<Widget>(new Widget, customDeleter), computePriority());
```

需要修改为

```C++
std::shared_ptr<Widget> spw(new Widget, customDeleter);
processWidget(spw, computePriority()); // lvalue
```

一定条件下可以优化为

```C++
std::shared_ptr<Widget> spw(new Widget, customDeleter);
processWidget(std::move(spw), computePriority()); // rvalue
```

### Item 22: When using the Pimpl Idiom, define special member functions in the implementation file.

Pimpl Idiom一种在对类进行修改后加速编译的方法

```C++
#include <string>
#include <vector>
Class Widget {
private:
	std::string name;
	std::vector<double> data;
};
```

修改为

```C++
#include "Impl"
Class Widget {
Public:
	Widget() : pImpl(std::make_unique<Impl>()) {}
private:
	Struct Impl;
	Impl * pImpl;
};
```

声明未定义的类型称为incomplete type

????



## Chapter 5: Rvalue References, Move Semantics, and Perfect Forwarding

move语义：使用廉价的move代替昂贵的copy，并且允许创造move-only类型（例如std::unique_ptr，std::future,std::thread）

Perfect forwarding：可以写出接受抽象参数类型的函数模板，并将这些抽象类型参数forward给其他函数，使得这些函数接收的参数类型和我们传给forward函数的类型完全一致

参数永远是lvalue，即使他的类型是rvalue reference

```C++
void f(Widget&& w);
```

w是lvalue，即使他的类型是rvalue-reference-to-Widget

### Item 23: Understand std::move and std::forward

move和forward并不产生任何可执行代码，均为cast函数

move无条件将参数转为rvalue，forward仅当确定条件满足执行这个转换

c++14下的move的类似实现

```C++
template<typename T>
decltype(auto) move(T&& param)
{
	using ReturnType = remove_reference_t<T>&&;
	return static_cast<ReturnType>(param);
}
```

如果要move一个对象，不要将他声明为const（const对象的move操作会转换为拷贝操作）

move不保证move操作cast的对象是可以move的

```C++
class Annotation {
public:
	explicit Annotation(const std::string text) // lvalue const string
		: value(std::move(text)) {} // 应用move之后是rvalue const string
}
```

string的构造函数

```C++
string(const string& rhs); // copy ctor
string(string&& rhs); // move ctor
```

Move ctor接受non-const类型，上述构造函数使用了copy ctor

```C++
void process(const Widget& lvalArg); // lvalue
void process(Widget&& rvalArg); // rvalue
```

如果形参通过rvalue初始化（形参是由lvalue或rvalue初始化记录在T中，item28），则forward将其cast为rvalue

```C++
template<typename T> 
void logAndProcess(T&& param) 
{
	// 所有函数的实参均为lvalue
	process(std::forward<T>(param)); // 调用process(Widget&& rvalArg)
}

logAndProcess(w); // call with lvalue
logAndProcess(std::move(w)); // call with rvalue
```

纯技术角度上说forward可以干一切需要move的活

forward需要加上type模板参数，move不需要

forward参数必须为non-reference，如果传入string&会发生拷贝构造而非移动构造（because that’s the convention for encoding that the argument being passed is an rvalue (see Item 28).？？？）

### Item 24: Distinguish universal references from rvalue references.

Universal reference出现的两个例子，都出现了type deduction

```C++
template<typename T>
void f(T&& param); // param为universal reference
auto&& var2 = var1; // var2为universal reference
```

Universal reference需要被初始化，lvalue进行初始化则为lvalue reference，rvalue初始化则为rvalue reference

```C++
template<typename T>
void f(T&& param); // param is a universal reference
Widget w;
f(w); // 左值传给f，param类型为左值引用Widget& (i.e., an lvalue reference)
f(std::move(w)); // 右值传给f，param类型为右值引用Widget&& (i.e., an rvalue reference)
```

作为universal reference，不仅需要涉及类型推导，还需要精确出现T&&（即f被调用时T会被类型推导，Corner case为除非调用者显式指定T）

```C++
template<typename T>
void f(std::vector<T>&& param); // 右值引用而非universal reference
template<typename T>
void f(const T&& param); // 右值引用而非universal reference
```

Corner case：

```C++
template<class T, class Allocator = allocator<T>> // from C++
class vector { // Standards
public:
	void push_back(T&& x); // x为右值引用而非universal reference
};
```

Push_back函数的存在依赖vector的实例化，即声明了vector<Widget>导致push_back实例化为push_back(Widget&& x);

与之相反

```C++
template<class T, class Allocator = allocator<T>> // still from
class vector { // C++
public: // Standards
	template <class... Args>
	void emplace_back(Args&&... args); //不依赖于vector，调用时进行类型推导，为universal reference
};
```

个人总结为，“调用时需要进行类型推导”的“形如T&&”的类型为universal reference

Auto universal reference，auto&&可以绑定到任何抽象类型

```C++
auto timeFuncInvocation = [](auto&& func, auto&&... params) // C++14
{
	start timer;
	std::forward<decltype(func)>(func)( // invoke func
		std::forward<decltype(params)>(params)... // on params
	);
	stop timer and record elapsed time;
};
```

func可以绑定到任何callable object，lvalue，rvalue，args可以绑定到任何抽象类型

### Item 25: Use std::move on rvalue references, std::forward on universal references.

move用于rvalue reference（右值引用被move无条件cast为右值）

```C++
class Widget {
public:
	Widget(Widget&& rhs) // rhs is rvalue reference
		: name(std::move(rhs.name)), p(std::move(rhs.p)) { … }
};
```
			
forward用于universal reference（universal reference被forward有条件的cast为右值）

```C++
class Widget {
public:
	template<typename T>
	void setName(T&& newName) // newName is universal reference
	{ name = std::forward<T>(newName); } 
};
```

move用于universal reference会带来灾难

```C++
class Widget {
public:
	template<typename T>
	void setName(T&& newName) // universal reference
	{ name = std::move(newName); }
};
Auto n = …;
w.setName(n);
// 现在n的值不确定
```

内部需要将参数传给其他函数的函数，不要为其重载参数为const string&或者string&&这两个版本，而是使用universal reference并在内部使用forward将参数传递给其他函数

对于最后一次使用的rvalue/universal reference，应用move（右值引用）或者forward（universal reference）

Return by value函数，并且返回右值引用或者universal reference，则使用move或者forward返回
Move

```C++
Matrix operator+(Matrix&& lhs, const Matrix& rhs) // lhs为右值，重用其存储空间从而保留矩阵和
{
	lhs += rhs;
	return std::move(lhs); // 使用move将lhs cast为右值
	Return lhs; // 拷贝lhs至return value
}
```

若matrix不支持move操作，return std::move(lhs)将调用拷贝构造函数

Forward

```C++
template<typename T>
Fraction reduceAndCopy(T&& frac) // universal reference param
{
	frac.reduce();
	return std::forward<T>(frac);
}
```

frac为右值则move，左值则拷贝

Return by value函数，但是返回函数内的局部变量，不要应用move或者forward（会使用C++的返回值优化RVO）

```C++
Widget makeWidget() // Moving version of makeWidget
{
	Widget w;
	return std::move(w); // move w into return value，(don't do this!)
	return w; // 编译器可能使用RVO
} 
```

满足以下两个条件，编译器会忽略对函数return by value的局部变量的拷贝或者移动操作：

	1. 局部变量类型和返回值类型一样
	2. 返回这个局部变量

RVO根据局部变量的是不是named（临时变量）又可看做RVO和NRVO（unamed）

若上述例子使用了return std::move(w)；则返回的不是这个局部变量，使得编译器无法进行RVO

如果编译器满足上述两个条件但并没有使用RVO（满足RVO条件但不应用 RVO），那么被返回的局部变量必须被视作右值，因此在RVO可以应用的情况下，编译器：

	1. 应用RVO
	2. 视返回值为rvalue，隐式应用move：

```C++
Widget makeWidget()
{
	Widget w;
	return std::move(w);
}
```

同样的，对于by-value函数参数，They’re not eligible for copy elision with respect to their function’s return value（返回值不能使用copy取消RVO？？？）, but compilers must treat them as rvalues if they’re returned（如果返回编译器要将他们视作右值？？？）. 

```C++
Widget makeWidget(Widget w) // by-value参数与返回值类型一样
{ return w; }
```
	
视作被写为

```C++
Widget makeWidget(Widget w) // w视作右值
{ return std::move(w); }
```

### Item 26: Avoid overloading on universal references 

```C++
std::multiset<std::string> names;     // global data structure
void logAndAdd(const std::string& name) {
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
```

如果给logAndAdd添加一个参数为string id的重载函数

```C++
void logAndAdd(int idx)
	names.emplace(nameFromIdx(idx));
}
logAndAdd(22); // 调用此重载版本
short id;
logAndAdd(id); // 精确匹配universal reference参数版本
```

使用universal reference的函数是C++中最贪婪的函数，几乎可以精确匹配任何类型（item 30几个例外），因此不应该使用含有universal reference参数的重载函数

```C++
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
```

T 实例化为Person&，而编译器生成的copy ctor含有一个const，匹配完美转发ctor

```C++
const Person p("Nancy");
auto cloneOfP(p);
```

T 实例化为 const Person&，在重载决议时若模板函数和普通函数均同等匹配，优先选择普通函数，匹配copy ctor

涉及到继承时

```C++
class SpecialPerson: public Person {
public:
	SpecialPerson(const SpecialPerson& rhs)  // copy ctor，调用基类的转发构造函数
		: Person(rhs) {... }
	SpecialPerson(SpecialPerson&& rhs) // move ctor，调用基类的转发构造函数
		: Person(std::move(rhs))  {... }
};
```

子类总是将类型为SpecialPerson的参数传给父类


### Item 27: Familiarize yourself with alternatives to overloading on universal references

避免重载universal reference

#### Pass by lvalue-reference-to-const(const T &)

（item26）但是效率有所下降

#### Pass by value

```C++
class Person {
public:
	explicit Person(std::string n) // replaces T&& ctor; see
		: name(std::move(n)) {} // Item 41 for use of std::move
	explicit Person(int idx) // as before
		: name(nameFromIdx(idx)) {}
Private:
	std::string name;
};
```

#### Tag dispatch

如果继续使用universal reference并且使用重载，则使用tag dispatch（将需要重载的函数写成logAndAddImpl，使用一个带有tag参数的函数根据tag将调用分发给某个对应的impl）
原有错误重载形式

```C++
template<typename T> 
void logAndAdd(T&& name) { … }
void logAndAdd(int idx) { … }
```

Tag dispatcher为

```C++
template<typename T>
void logAndAdd(T&& name) {
	logAndAddImpl(std::forward<T>(name), 
		std::is_integral<typename std::remove_reference<T>::type>()); 
}
```
	
原有universal reference non-integral函数为

```C++
template<typename T> // non-integral
void logAndAddImpl(T&& name, std::false_type) { // compile time value（true和false是runtime values）
	names.emplace(std::forward<T>(name));
}
void logAndAddImpl(int idx, std::true_type) { // integral
	logAndAdd(nameFromIdx(idx)); 
}
```

logAndAdd(stringId)会被正确分发至logAndAddImpl(int idx)函数中


#### 约束使用universal reference的模板

即使使用tag dispatch写了一个ctor处理重载问题，对于编译器自动生成的copy和move ctor，还是会有ctor调用绕过tag dispatch机制的情况：提供一个universal reference的ctor会使得拷贝non-const lvalue时调用universal reference版本（此时期望调用copy ctor），或者调用子类拷贝或移动ctor时调用了父类的universal reference ctor版本

使用std::enable_if

如果希望ctor的参数不是Person类型时调用universal reference ctor版本（Person类型期望调用copy/move ctor）

```C++
class Person {
public:
	template<typename T, 
			typename = typename std::enable_if<condition>::type> // 参考SFINAE
	explicit Person(T&& n);
};
```

condition为!std::is_same<Person, T>::value // 仍然错误

给出

```C++
Person p("Nancy");
auto cloneOfP(p); 
```

这里构造cloneOfP的过程中uniref中的T被推导为Person&，跟Person不是一类，condition为true就仍然调用uniref，错误！因此准确地说应该忽略（使用std::decay<T>::type）

	1. 类型是非为ref，Person Person& Person&&均视作同样类型
	2. 类型为const或volatile，const Person/volatile Person等视作同类型
	
并且，如果由子类的copy/move ctor调用父类的对应ctor，父类ctor的传入参数是子类类型，因此需要当类型为Person或者Person的子类时，不调用uniref版本（使用std::is_base_of<T1, T2>::value，is_base_of<T, T>::value为真）

condition应该修改为

```C++
!std::is_base_of<Person, typename std::decay<T>::type>::value
```

正确的Person类如下

```C++
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
```

Tag dispatch和constrained template都使用了perfect forward技术，有两个缺点：

	1. 有些参数不能perfect forward
	2. 错误信息复杂，只有在真正不匹配的位置（经过层层函数调用）产生错误信息

### Item 28: Understand reference collapsing.

template<typename T> void func(T&& param);会将参数是lvalue还是rvalue编码进uniref的T的类型推导中， 即如果是lvalue，T为lvalue reference，rvalue则T为non-reference

```C++
Widget widgetFactory(); // 返回rvalue
Widget w; // lvalue
func(w); // T为Widget&
func(widgetFactory()); // T为Widget
```

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

```C++
template<typename T> 
T&& forward(typename remove_reference<T>::type& param) {
	return static_cast<T&&>(param);
}
```

Lvalue reference传给forward，T为Widget&

```C++
Widget& && forward(typename remove_reference<Widget&>::type& param)
{ return static_cast<Widget& &&>(param); }
```

结果为

```C++
Widget& forward(Widget& param)
{ return static_cast<Widget&>(param); }
```

Rvalue reference传给forward，T为Widget&&

```C++
Widget&& && forward(typename remove_reference<Widget&&>::type& param)
{ return static_cast<Widget&& &&>(param); }
```

结果为

```C++
Widget&& forward(Widget& param)
{ return static_cast<Widget&&>(param); }
```
	
#### auto类型生成

```C++
Widget widgetFactory(); // 返回rvalue
Widget w; // lvalue
Auto && w1 = w; // lvalue初始化w1，auto为Widget&，auto &&为Widget&
Auto && w2 = widgetFactory(); //  rvalue初始化w2，auto为Widget&&，auto&&为Widget&&
```

因此uniref并不是新的ref类型，而是

	1. 类型推导，区分lvalue和rvalue（lvalue的T推导为T&，rvalue的T推导为T）
	2. 发生Reference collapse


#### typedef和alias声明

一个typedef

```C++
template<typename T>
class Widget {
public:
	typedef T&& RvalueRefToT;
};
```

调用

```C++
Widget<int&> w;
```

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

```C++
template<typename T>
void fwd(T&& param) { // accept any argument
	f(std::forward<T>(param)); // forward it to f
}
template<typename... Ts>
void fwd(Ts&&... params) { // accept any arguments
	f(std::forward<Ts>(params)...); // forward them to f
}
```

有几种参数类型会导致perfect forward失败（调用目标函数f和转发函数fwd产生不同表现）：

	1. Braced initializers
	2. 0 or NULL as null pointers
	3. Declaration-only integral static const data members
	4. Overloaded function names and template names
	5. Bitfields

#### Braces initializers

```C++
Void f(const vector<int&> v);
```

调用f函数，编译器比较传入参数和f函数的参数声明是否匹配，如需要会发生隐式转换

```C++
f({ 1, 2, 3}); // 可行，隐式转换为vector<int>
```

对fwd的调用不比较传入参数和fwd函数参数声明，而是推导传给fwd的参数的类型，并将推导类型与f的参数声明进行比较

```C++
fwd({ 1, 2, 3}); // 编译错误
```

没有声明为std::initializer_list的braced initializer是一个non-deduced context，编译器无法进行类型推导

```C++
auto il = { 1, 2, 3 }; // auto推导为std::initializer_list<int>
fwd(il); // 可行，fwd将il完美转发给f
```

Perfect forward失败：

	1. 编译器无法推导出类型
	2. 编译器推导出错误类型


#### 0 or NULL as null pointers

0和NULL会被错误推导为整形而不是null pointer，请使用nullptr

#### Declaration-only integral static const data members

对于static const整形成员，声明即可（不需要定义它，因为对于这些成员的值编译器执行const propagation，不需要额外设置这些成员的内存，编译器将所有使用这些成员的地方直接用值替换）。但是只有定义了这些变量，才能够获取他们的地址。

```C++
class Widget {
public:
	static const std::size_t MinVals = 28;
};
f(Widget::MinVals); // fine, treated as "f(28)"
fwd(Widget::MinVals); // error! shouldn't link
```

没有获取MinVals的地址，但是使用了uniref，在编译器生成代码中往往将reference视为指针，并且在程序的二进制码和硬件上，指针和引用是相同的。但也有编译器可能执行正确

	
#### Overloaded function names and template names

重载函数名情况
定义f为

```C++
void f(int (*pf)(int)); // 等同于void f(int pf(int));
```

两个重载函数
```C++
int processVal(int value);
int processVal(int value, int priority);
```

调用

```C++
f(processVal) // 可行，选择第一个processVal函数
Fwd(processVal) // 错误，fwd不包含任何信息指明需要哪种类型
```

模板名情况
函数模板代表许多函数

```C++
template<typename T>
T workOnVal(T param) { … }
fwd(workOnVal); // 错误，不知道使用哪一个实例化函数
```

正确做法为

```C++
using ProcessFuncType = int (*)(int); 
ProcessFuncType processValPtr = processVal;
fwd(processValPtr); // 可行
fwd(static_cast<ProcessFuncType>(workOnVal)); // 可行
```

#### Bitfields

（bitfield表示机器字中的一部分，例如32位int的第3-5位，不可能直接获取这个bitfield的地址，通常引用和指针在硬件层面上等同）

```C++
struct IPv4Header {
	std::uint32_t version:4,
			IHL:4,
			DSCP:6,
			ECN:2,
			totalLength:16;
…
};
```

定义

```C++
void f(std::size_t sz); // function to call
IPv4Header h;
f(h.totalLength); // fine
fwd(h.totalLength); // 错误
```

h.totalLength是non-const bitfield，fwd的参数是reference，c++标准要求

> non-const reference不能绑定到bitfield上

没有函数能将引用绑定到bitfield上，或者获取bitfield的地址，因此bitfield参数只能是by-value或者reference-to-const（标准要求在一个标准整形的对象上存储这个bitfield的拷贝，并将reference绑定到这个标准整形的对象上？？？）

正确做法为

```C++
auto length = static_cast<std::uint16_t>(h.totalLength); // 拷贝bitfield值
fwd(length); // forward这个拷贝
```



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

```C++
using FilterContainer = std::vector<std::function<bool(int)>>;
FilterContainer filters;
filters.emplace_back([](int value) { return value % 5 == 0; } ); // filter函数容器添加一个函数
```

使用

```C++
void addDivisorFilter()
{
	auto divisor = …;
	filters.emplace_back([&](int value) { return value % divisor == 0; }); // divisor会变为悬空指针
	filters.emplace_back([&divisor](int value) { return value % divisor == 0; }); // divisor会变为悬空指针
}
```

相比默认方式[&]而言，显式列出lambda中的局部变量[&divisor]会容易发现这个lambda依赖于这个引用并可能悬空

将divisor修改为by-value capture防止悬空引用

```C++
filters.emplace_back([=](int value) { return value % divisor == 0; }); // divisor为by-value capture
```

lambda只能使用于在定义该lambda的作用域内可见的non-static local variable（包含参数）。类的non-static成员变量在访问时通过隐式添加this指针，因此lambda中的[=]只能捕获this指针，使用non-static成员变量的方式为

```C++
void Widget::addFilter() const
{
	auto currentObjectPtr = this; // 获得this指针
	filters.emplace_back(
		[currentObjectPtr](int value) // 捕获this指针
		{ return value % currentObjectPtr->divisor == 0; } );
}
```

this变量生存期

```C++
using FilterContainer = std::vector<std::function<bool(int)>>;
FilterContainer filters;
void doSomeWork() {
	auto pw = std::make_unique<Widget>(); 
	pw->addFilter(); // addFilter中的lambda捕获了pw的this指针存入filters
}
// 这里pw失效，filters中的捕获pw的this的lambda产生悬空指针
```

使用by-value捕获pw->divisor，将addFilter改为

```C++
void Widget::addFilter() const {
	auto divisorCopy = divisor; // copy data member
	filters.emplace_back(
		[divisorCopy](int value) // capture the copy
		{ return value % divisorCopy == 0; } // use the copy
	);
}
```

使用默认by-value capture同样正确（不推荐）A default capture mode is what made it possible to accidentally capture this when you thought you were capturing divisor in the first place.？？？

```C++
void Widget::addFilter() const {
	auto divisorCopy = divisor; // copy data member
	filters.emplace_back(
		[=](int value) { return value % divisorCopy == 0; } // use the copy
	);
}
```

C++14中generalized lambda capture可以捕获成员变量

```C++
void Widget::addFilter() const
{
	filters.emplace_back( // C++14:
		[divisor = divisor](int value) // copy divisor to closure
		{ return value % divisor == 0; } // use the copy
	);
}
```

lambda不仅依赖于（可捕获的）局部变量和参数，还依赖于（不可捕获的）objects with static storage duration（定义在全局或名空间作用域下的对象，或者在类、函数、文件中声明为static的对象），因此by-value capture的lambda并不一定self-contained

```C++
void addDivisorFilter()
{
	static auto divisor = …;
	filters.emplace_back(
		[=](int value) // captures nothing!，并没有对任何对象作出by-value capture
		{ return value % divisor == 0; } // refers to above static
	);
	++divisor;
}
```

### item 32: Use init capture to move objects into closures

C++14能够将move-only对象move到closure中，C++11无法做到

C++11中的新的捕获机制init capture能够做C++11的几乎所有捕获形式（除了默认捕获模式等），并且还能够可以实现capture-by-move，又称作generalized lambda capture

Init capture可以指定：

	1. Closure class中一个data member的名字
	2. 一个用来初始化这个data member的表达式

使用init capture将unique_ptr移动至closure中

```C++
auto pw = std::make_unique<Widget>(); 
auto func = [pw = std::move(pw)] // init capture，（少了个()参数列表？？？）
	{ return pw->isValidated() && pw->isArchived(); }; 
```

在init capture [pw = std::move(pw)]中，=左边是指定的closure的data member名称，作用域是closure class；=右边是初始化表达式，作用域和该lambda的定义位置一样

局部变量pw并不是必须的，因为closure class的data member可以直接被make_unique初始化

```C++
auto func = [pw = std::make_unique<Widget>()]
	{ return pw->isValidated() && pw->isArchived(); };
```

可以通过C++11实现上述含有move的lambda效果

```C++
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
```

或者使用C++11的lambda实现上述效果：

	1. 将对象move到std::bind产生的function对象中（被函数对象“捕获”）
	2. 使用by-reference capture捕获上述被“捕获”的对象

例如

```C++
std::vector<double> data; 
auto func = [data = std::move(data)] // C++14 init capture
		{ /* uses of data */ };
auto func = std::bind( // C++11 实现版本
	[](const std::vector<double>& data) { /* uses of data */ },
	std::move(data)
);
```

Std::bind返回bing object，第一个参数为callable object

一个bing object包含所有传入bind函数参数的拷贝，包括第一个lambda产生的closure参数的拷贝（这个closure的生存期和bind object一样），如果参数为lvalue使用copy ctor，rvalue使用move ctor

上述bind版本跟C++14版本区别在于bind版本的lambda中的data是一个指向bing object中data的lvalue reference（std::move(data)是rvalue，但是bind object中的data是lvalue）

默认closure class中operator()成员函数是const的，在C++14中lambda体内部所有data member都是const的，为了C++11版本bind object内的move-ctor的data不被修改，将参数声明为reference-to-const。若lambda声明为mutable，则无需使用const

```C++
auto func = std::bind(
	[](std::vector<double>& data) mutable { /* uses of data */ },
	std::move(data)
);
```

注意的几点：

	1. C++11中，不可能move-ctor一个对象到closure中，但是可以到bind object中
	2. C++11中，模拟move-capture的方法是将对象move ctor到一个bind object中，并将这个对象by reference传给lambda
	3. Closure和bind object生存期相同，可以将bind object中的对象视作closure中

对make_unique被捕获，C++14为

```C++
auto func = [pw = std::make_unique<Widget>()] 
		{ return pw->isValidated() && pw->isArchived(); };
```

C++11的模拟为

```C++
auto func = std::bind(
	[](const std::unique_ptr<Widget>& pw)
		{ return pw->isValidated() && pw->isArchived(); },
	std::make_unique<Widget>()
);
```

Item34指出倾向于使用lambda代替bind，除了几个例外情况（例如C++11下bind模拟C++14特性）

### item 33: Use decltype on auto&& parameters to std::forward them.


C++14中引入了generic lambdas（lambda中可以使用auto进行参数指定）

```C++
auto f = [](auto x){ return func(normalize(x)); };
```

这个lambda对应的closure class类似如下

```C++
class SomeCompilerGeneratedClassName {
public:
	template<typename T> 
	auto operator()(T x) const // auto return type item3
		{ return func(normalize(x)); }
	… // other closure class
};
```

这里即使lambda的传入参数为rvalue，lambda仍然将x作为lvalue传给normalize函数。需要将x修改为uniref，并将其通过forward传给normalize函数，即

```C++
auto f = [](auto&& x) { return func(normalize(std::forward<???>(x))); };
```

如果作为模板函数，使用std::forward<T>即可，在generic lambda中，需要使用decltype（传入lvalue时产生lvalue reference，传入rvalue时产生rvalue reference）

> 表达式e是一个局部空间或者名空间中的变量，或是一个静态成员变量或函数param，decltype是这个变量或参数的声明类型
> 否则（表达式e不是上述情况），e is lvalue -> decltype(e) is T&；e is xvalue -> decltype(e) is T&&；e is prvalue -> decltype(e) is T
> (wiki: decltype)

Item28中C++14的forward实现

```C++
template<typename T> 
T&& forward(remove_reference_t<T>& param) {
	return static_cast<T&&>(param);
}
```

如果T被设置为Widget（T=Widget），那么为

```C++
Widget&& forward(Widget& param) { // T = Widget
	return static_cast<Widget&&>(param); 
}
```

如果T为Widget的右值引用

```C++
Widget&& && forward(Widget& param) { // T = Widget&&
	return static_cast<Widget&& &&>(param); 
}
```

即

```C++
Widget&& && forward(Widget& param) { // T = Widget&&
	return static_cast<Widget&&>(param); 
}
```

T=Widget和T=Widget&&完全一样，即使用rvalue reference或者non-reference类型实例化forward，结果一样

如果lvalue传给lambda，decltype(x)将customary type传给forward，如果rvalue传给lambda，decltype(x)使用与否结果一样，因此decltype(x)总能得到想要的结果。

因此正确的lambda为

```C++
auto f = [](auto&& param) { return func(normalize(std::forward<decltype(param)>(param))); };
```

C++14的lambda可以是可变参数，即

```C++
auto f = [](auto&&... params) { return func(normalize(std::forward<decltype(params)>(params)...)); };
```

### item 34: Prefer lambdas to std::bind

Bind最早在2005的TR1中引入，C++11进入std

倾向于使用lambda而不是bind

lambda比bind更可读

设置闹钟的lambda版本，从调用setSoundL内部的setAlarm函数开始记时

```C++
using Time = std::chrono::steady_clock::time_point;
enum class Sound { Beep, Siren, Whistle };
using Duration = std::chrono::steady_clock::duration;
void setAlarm(Time t, Sound s, Duration d);

auto setSoundL = [](Sound s) {
	using namespace std::chrono;
	setAlarm(steady_clock::now() + hours(1), s, seconds(30));
};
```

对应的bind版本

```C++
using namespace std::chrono; // as above
using namespace std::placeholders;

auto setSoundB = std::bind(
	setAlarm,
	steady_clock::now() + 1h, // 错误，从调用bind函数开始记时
	_1,
	seconds(30)
);
```

正确做法为（从调用setSoundB函数内部的setAlarm函数开始记时？？？）

```C++
auto setSoundB = std::bind(
	setAlarm,
	std::bind( std::plus<steady_clock::time_point>(), steady_clock::now(), hours(1) ), // 正确
	_1,
	seconds(30)
);
```

当setAlarm函数重载时，lambda仍然正确

```C++
enum class Volume { Normal, Loud, LoudPlusPlus };
void setAlarm(Time t, Sound s, Duration d);
void setAlarm(Time t, Sound s, Duration d, Volume v);

auto setSoundL = [](Sound s) {
	using namespace std::chrono;
	setAlarm(steady_clock::now() + 1h, s, 30s); // 调用三个参数版本
};
```

bind错误

```C++
auto setSoundB = std::bind(
	setAlarm,  // 错误，无法确定调用哪一个重载函数
	std::bind(std::plus<steady_clock::time_point>(), steady_clock::now(), 1h),
	_1,
	30s
);
```

需要显式指定函数指针类型

```C++
using SetAlarm3ParamType = void(*)(Time t, Sound s, Duration d);
auto setSoundB = std::bind(
	static_cast<SetAlarm3ParamType>(setAlarm), 
	std::bind(std::plus<steady_clock::time_point>(), steady_clock::now(), 1h),
	_1,
	30s
);
```

上述函数指针访问方法使得编译器更难将其inline处理，即setSoundL(Sound::Siren); 相比setSoundB(Sound::Siren);  更容易被inline处理

对于更加复杂的lambda

```C++
auto betweenL = [lowVal, highVal] (int val) { return lowVal <= val && val <= highVal; }; //  C++11 版本
auto betweenL = [lowVal, highVal] (const auto& val) { return lowVal <= val && val <= highVal; }; // C++14版本
```

bind版本为

```C++
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
```

Placeholder（例如_1，_2）加重理解困难

lambda对于by-value或是by-reference显式标出

```C++
auto compressRateL = [w](CompLevel lev) { return compress(w, lev); }; // w以by-value形式存储
```

bind总是拷贝他的参数（如果是rvalue？？？）

bind将w参数以by-value形式存储在compressRateB中（w是可变的，以by-value还是by-reference存储会影响compressRateB的调用结果）

```C++
using namespace std::placeholders;
enum class CompLevel { Low, Normal, High }; 
Widget compress(const Widget& w, CompLevel lev);
Widget w;
auto compressRateB = std::bind(compress, w, _1); 
```

如果想要以by-reference方式存储则使用

```C++
auto compressRateB = std::bind(compress, std::ref(w), _1);
```

在C++14中，没有任何理由使用bind而不非lambda，但是在C++11中，仍然有两个例外需要使用bind：

	1. Move capture（item32）
	2. Polymorphic function objects.

Polymorphic function objects，bindobject的函数调用操作使用perfect forwarding，它可以接收任何类型（一些限制见item30），可用于将对象绑定至模板化函数调用

```C++
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
```

C++14中对应做法为

```C++
auto boundPW = [pw](const auto& param) { pw(param); };
```



## Chapter 7: The Concurrency API

C++11提供了标准的多线程编程行为

标准库有两个future模板，std::future，std::shared_future，下文统称为future

### Item 35: Prefer task-based programming to threadbased

异步的执行函数doAsyncWork，thread-based方式

```C++
int doAsyncWork();
std::thread t(doAsyncWork);
```

Task-based方式

```C++
auto fut = std::async(doAsyncWork); // "fut" for "future"
```

想要获得doAsyncWork的返回值，thread-based方式没有直接获取结果的方法，task-based方式可以通过fut的get函数获得

如果doAsyncWork抛出异常，thread-based方式内部调用std::termunate线程die，get函数可以访问到这个异常

C++中task比thread更加高层，thread涉及到线程管理的细节

并行C++中的线程

> 硬件线程：真正进行计算的线程，现代硬件架构支持在每个CPU核心上提供多个硬件线程
> 软件线程（OS线程或系统线程）：操作系统基于硬件线程对所有进程和调度的执行做的线程管理，通常多于硬件线程
> Std::thread：为软件线程的handle，可以被创建、移动和detach


即使doAsyncWork声明为noexcept，仍然会因为创建的线程超过系统的线程数限制而抛出std::system_error

即使线程数没有超过上限，也会因为oversubscription而导致麻烦（unblocked线程多于硬件线程），频繁的上下文切换回带来巨大开销。(1) the CPU caches are typically cold for that software thread (i.e., they contain little data and few instructions useful to it) and (2) the running of the “new” software thread on that core “pollutes” the CPU caches for “old” threads that had been running on that core and are likely to be scheduled to run there again.

通过使用async将线程管理问题交给标准库实现

```C++
auto fut = std::async(doAsyncWork);
```

async不保证创建一个新的软件线程，相反，他允许doAsyncWork在想要获取doAsyncWork结果的线程（fut的get或者wait线程）上允许

对于负载均衡问题，现在由调度器来面对

使用async并不能避免GUI线程的负载均衡问题，因为不知道哪一个线程对响应需求要求更高，因此要使用std::launch::async传入launch policy（item36），这能够保证想要执行的函数在不同的线程上执行

最新技术的线程调度器使用了系统范围下的线程池来避免oversubscription问题，并使用work-stealing算法改进了硬件核心的负载均衡。C++的标准并没有对这些算法有要求，但是供应商们会应用这些算法，如果使用task-based方式，会自动利用上这些优势，thread-based方式需要自行处理各种问题

Task-based方式能够更自然的访问函数的异步执行的结果（包括返回值和异常）

一些需要直接使用thread的情形

	1. 需要获取内部线程实现的API（例如pthread和windows thread的相关实现），这些实现往往功能更多。Std::thread对象提供native_handle成员函数用以获取内部实现的API，而std::async返回的std::future没有对应功能
	2. 需要对特定应用进行线程使用的优化
	3. 需要基于C++并行API实现线程技术
	

### Item 36: Specify std::launch::async if asynchronicity is essential

通过std::async(f)进行异步执行有两种标准策略

	1. Std::launch::async launch policy，f要在不同线程上异步执行
	2. Std::launch::deferred launch policy，只有当调用了std::async返回的get或者wait函数是，运行f，即当get或者wait被调用时，同步执行f（阻塞调用方）直到f结束执行，如果没有调用get或者wait则永远不执行f

默认launch policy为

```C++
auto fut1 = std::async(f); // run f using default launch policy
auto fut2 = std::async(std::launch::async | 
				std::launch::deferred, 
				f); // run f either async or deferred
```

对于async的默认policy，线程t运行以下程序

```C++
auto fut = std::async(f); // run f using default launch policy
```

无法预测

	1. f是否会和t并行运行
	2. f是否会运行在不同于调用get或者wait函数的线程
	3. f是否运行

这个策略对thread_local变量来说很糟糕，如果f读写了thread-local storage（TLS），不能够预测究竟是哪一个线程的变量被access（上述例子中运行f的独立线程或是执行get或wait函数的线程）

这个策略也会影响使用timeout的wait-based 循环，调用wait_for或者wait_until的任务会被deferred，即看上去一定会终止的循环会永远运行

```C++
using namespace std::literals; 

void f() {
	std::this_thread::sleep_for(1s);
}

auto fut = std::async(f); // run f asynchronously (conceptually)
while (fut.wait_for(100ms) != std::future_status::ready) { // 可能永远无法结束循环
	…
}
```

如果调用async(f)使用的std::launch::async，则不会出现问题；如果使用std::launch::deferred，则wait_for永远返回std::future_status::deferred（永远不会为std::future_status::ready），循环永远不会终止

没有直接方法向future判断一个任务是否deferred，只能通过一个超时函数例如wait_for的返回值来判断，而我们并不应该等待而是要获得wait_for的返回值，因此做法为

```C++
auto fut = std::async(f); 

if (fut.wait_for(0s) == std::future_status::deferred) { // 如果任务被deferred
	// ...use wait or get on fut to call f synchronously
} 
else { // task isn't deferred
	while (fut.wait_for(100ms) != std::future_status::ready) { // f终会结束
		… // task is neither deferred nor ready, so do concurrent work until it's ready
	}
	… // fut is ready
}
```

因此默认策略正常工作的条件是

	1. 任务不要求和调用get或者wait的线程并行执行
	2. 无所谓哪一个线程的thread_local变量被读写
	3. get或者wait一定会被调用，或者任务也可以从不被执行
	4. 对wait_for或者wait_until的调用将人也可能会被延迟的情况考虑在内

任何一个条件不满足，都需要给出策略

```C++
auto fut = std::async(std::launch::async, f); // 异步launch函数f
```

自动使用std::launch::async的自定义std::async的C++11版本，使用std::result_of模板获得返回类型

```C++
template<typename F, typename... Ts>
inline
std::future<typename std::result_of<F(Ts...)>::type>
reallyAsync(F&& f, Ts&&... params) { // 返回future
	return std::async(std::launch::async, 
				std::forward<F>(f),
				std::forward<Ts>(params)...);
}

auto fut = reallyAsync(f);
```

C++14版本为

```C++
template<typename F, typename... Ts>
inline
auto
reallyAsync(F&& f, Ts&&... params) {
	return std::async(std::launch::async,
				std::forward<F>(f),
				std::forward<Ts>(params)...);
}
```
