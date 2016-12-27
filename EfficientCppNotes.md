# Effective C++ 摘要


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
