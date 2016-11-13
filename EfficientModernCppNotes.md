

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

forward参数必须为non-reference，如果传入string&会发生拷贝构造而非移动构造
because that’s the convention for encoding that the argument being passed is an rvalue (see Item 28).？？？
