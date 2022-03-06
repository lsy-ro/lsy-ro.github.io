# C++11

## 原始字面量
[jump](#test)
在 C++11 中添加了定义原始字符串的字面量，定义方式为：R “xxx(原始字符串)xxx” 其中（）两边的字符串可以省略。原始字面量 R 可以直接表示字符串的实际含义，而不需要额外对字符串做转义或连接等操作。

## C++11 标准的类成员初始化

1. 初始化类的非静态成员

在进行类成员变量初始化的时候，C++11 标准对于 C++98 做了补充，允许在定义类的时候在类内部直接对非静态成员变量进行初始化，在初始化的时候可以使用等号 = 也可以 使用花括号 {} 。

```C
class Test
{
private:
    int a = 9;
    int b = {5};
    int c{12};
    double array[4] = { 3.14, 3.15, 3.16, 3.17};
    double array1[4] { 3.14, 3.15, 3.16, 3.17 };
    string s1("hello");     // error
    string s2{ "hello, world" };
};
```

可以看到如果使用花括号 {} 的方式对类的非静态成员进行初始化，等号是可以省略不写的。

s1：错误，不能使用小括号 () 初始化对象，应该使用 花括号 {}

2. 类内部赋值和初始化列表

初始化列表总是看起来后作用于构造函数内部的非静态成员初始化。也就是说，通过初始化列表指定的值会覆盖就地初始化时指定的值

## final 和 override
<div id="test"></div>
C++ 中增加了 final 关键字来限制某个类不能被继承，或者某个虚函数不能被重写，和 Jave 的 final 关键字的功能是类似的。如果使用 final 修饰函数，只能修饰虚函数，并且要把final关键字放到类或者函数的后面。

使用了 override 关键字之后，假设在重写过程中因为误操作，写错了函数名或者函数参数或者返回值编译器都会提示语法错误，提高了程序的正确性，降低了出错的概率。

## 模板的优化

两个右尖括之间需要添加空格，这样写起来就非常的麻烦，C++11改进了编译器的解析规则，尽可能地将多个右尖括号（>）解析成模板参数结束符，方便我们编写模板相关的代码。

在 C++98/03 标准中，类模板可以有默认的模板参数，但是不支持函数的默认模板参数，在C++11中添加了对函数模板默认参数的支持:

```c
#include <iostream>
using namespace std;

template <typename T=int>   // C++98/03不支持这种写法, C++11中支持这种写法
void func(T t)
{
    cout << "current value: " << t << endl;
}

int main()
{
    func(100);
    return 0;
}

```

当所有模板参数都有默认参数时，函数模板的调用如同一个普通函数。但对于**类模板**而言，哪怕所有参数都有默认参数，在使用时也必须在模板名后跟随 <> 来实例化


另外：函数模板的默认模板参数在使用规则上和其他的默认参数也有一些不同，它没有必须写在参数表最后的限制。这样当默认模板参数和模板参数自动推导结合起来时，书写就显得非常灵活了。我们可以指定函数模板中的一部分模板参数使用默认参数，另一部分使用自动推导

```c
#include <iostream>
#include <string>
using namespace std;

template <typename R = int, typename N>
R func(N arg)
{
    return arg;
}

int main()
{
    auto ret1 = func(520);
    cout << "return value-1: " << ret1 << endl;

    auto ret2 = func<double>(52.134);
    cout << "return value-2: " << ret2 << endl;

    auto ret3 = func<int>(52.134);
    cout << "return value-3: " << ret3 << endl;

    auto ret4 = func<char, int>(100);
    cout << "return value-4: " << ret4 << endl;

    return 0;
}
```

测试代码输出的结果为:
```
return value-1: 520
return value-2: 52.134
return value-3: 52
return value-4: d
```

当默认模板参数和模板参数自动推导同时使用时（优先级从高到低）：

1. 如果可以推导出参数类型则使用推导出的类型
2. 如果函数模板无法推导出参数类型，那么编译器会使用默认模板参数
3. 如果无法推导出模板参数类型并且没有设置默认模板参数，编译器就会报错。

## 自动类型推导

1. **auto**

C++11 中 auto 并不代表一种实际的数据类型，只是一个类型声明的 “占位符”，auto 并不是万能的在任意场景下都能够推导出变量的实际类型，使用auto声明的变量**必须要进行初始化**，以让编译器推导出它的实际类型，**在编译时**将auto占位符替换为真正的类型。

使用语法如下
```
auto 变量名 = 变量值;
```
auto 还可以和指针、引用结合起来使用也可以带上 const、volatile 限定符，在不同的场景下有对应的推导规则，规则内容如下：

- 当变量不是指针或者引用类型时，推导的结果中不会保留 const、volatile 关键字
- 当变量是指针或者引用类型时，推导的结果中会保留 const、volatile 关键字
    
### auto的应用
1. 用于STL的容器遍历。
```c
#include <map>
int main()
{
    map<int, string> person;
    // 代码简化
    for (auto it = person.begin(); it != person.end(); ++it)
    {
        // do something
    }
    return 0;
}
```

2. 用于泛型编程，在使用模板的时候，很多情况下我们不知道变量应该定义为什么类型，比如下面的代码：
```c
#include <iostream>
#include <string>
using namespace std;

class T1
{
public:
    static int get()
    {
        return 10;
    }
};

class T2
{
public:
    static string get()
    {
        return "hello, world";
    }
};

template <class A>
void func(void)
{
    auto val = A::get();
    cout << "val: " << val << endl;
}

int main()
{
    func<T1>();
    func<T2>();
    return 0;
}
```
2. **decltype**

在某些情况下，不需要或者不能定义变量，但是希望得到某种类型，这时候就可以使用 C++11 提供的 decltype 关键字了，它的作用是在编译器编译的时候推导出一个表达式的类型。

语法格式如下：
```
decltype (表达式)
```

decltype 是 “declare type” 的缩写，意思是 “声明类型”。decltype 的推导是在编译期完成的，它只是用于表达式类型的推导，并不会计算表达式的值。来看一组简单的例子：

```
int a = 10;
decltype(a) b = 99;                 // b -> int
decltype(a+3.14) c = 52.13;         // c -> double
decltype(a+b*c) d = 520.1314;       // d -> double`
```

### 推导规则

- 表达式为普通变量或者普通表达式或者类表达式，在这种情况下，使用 decltype 推导出的类型和表达式的类型是一致的

```c
#include <iostream>
#include <string>
using namespace std;

class Test
{
public:
    string text;
    static const int value = 110;
};

int main()
{
    int x = 99;
    const int &y = x;
    decltype(x) a = x;
    decltype(y) b = x;
    decltype(Test::value) c = 0;

    Test t;
    decltype(t.text) d = "hello, world";

    return 0;
}
```

变量 a 被推导为 int 类型<br/>
变量 b 被推导为 const int & 类型<br/>
变量 c 被推导为 const int 类型<br/>
变量 d 被推导为 string 类型<br/>

- 表达式是函数调用，使用 decltype 推导出的类型和函数返回值一致。
```c
class Test{...};
//函数声明
int func_int();                 // 返回值为 int
int& func_int_r();              // 返回值为 int&
int&& func_int_rr();            // 返回值为 int&&

const int func_cint();          // 返回值为 const int
const int& func_cint_r();       // 返回值为 const int&
const int&& func_cint_rr();     // 返回值为 const int&&

const Test func_ctest();        // 返回值为 const Test

//decltype类型推导
int n = 100;
decltype(func_int()) a = 0;     
decltype(func_int_r()) b = n;   
decltype(func_int_rr()) c = 0;  
decltype(func_cint())  d = 0;   
decltype(func_cint_r())  e = n; 
decltype(func_cint_rr()) f = 0; 
decltype(func_ctest()) g = Test();  
```

变量 a 被推导为 int 类型<br/>
变量 b 被推导为 int& 类型<br/>
变量 c 被推导为 int&& 类型<br/>
变量 d 被推导为 int 类型<br/>
变量 e 被推导为 const int & 类型<br/>
变量 f 被推导为 const int && 类型<br/>
变量 g 被推导为 const Test 类型<br/>

函数 func_cint () 返回的是一个纯右值（在表达式执行结束后不再存在的数据，也就是临时性的数据），对于纯右值而言，**只有类类型可以携带const、volatile限定符，除此之外需要忽略掉这两个限定符**，因此推导出的变量 d 的类型为 int 而不是 const int。

- 表达式是一个左值，或者被括号 ( ) 包围，使用 decltype 推导出的是表达式类型的引用（如果有 const、volatile 限定符不能忽略）

```c
#include <iostream>
#include <vector>
using namespace std;

class Test
{
public:
    int num;
};

int main() {
    const Test obj;
    //带有括号的表达式
    decltype(obj.num) a = 0;
    decltype((obj.num)) b = a;
    //加法表达式
    int n = 0, m = 0;
    decltype(n + m) c = 0;
    decltype(n = n + m) d = n;
    return 0;
}
```

obj.num 为类的成员访问表达式，符合场景 1，因此 a 的类型为 int<br/>
obj.num 带有括号，符合场景 3，因此 b 的类型为 const int&。<br/>
n+m 得到一个右值，符合场景 1，因此 c 的类型为 int<br/>
n=n+m 得到一个左值 n，符合场景 3，因此 d 的类型为 int&<br/>

### decltype的应用
关于 decltype 的应用多出现在泛型编程中。比如我们编写一个类模板，在里边添加遍历容器的函数，操作如下：

```c
#include <list>
using namespace std;

template <class T>
class Container
{
public:
    void func(T& c)
    {
        for (m_it = c.begin(); m_it != c.end(); ++m_it)
        {
            cout << *m_it << " ";
        }
        cout << endl;
    }
private:
    ??? m_it;  // 这里不能确定迭代器类型
};

int main()
{
    const list<int> lst;
    Container<const list<int>> obj;
    obj.func(lst);
    return 0;
}
```


在程序的第 17 行出了问题，关于迭代器变量一共有两种类型：只读（T::const_iterator）和读写（T::iterator），有了 decltype 就可以完美的解决这个问题了，当 T 是一个 非 const 容器得到一个 T::iterator，当 T 是一个 const 容器时就会得到一个 T::const_iterator。

```c
private:
    decltype(T().begin()) m_it;  // 这里不能确定迭代器类型, 要依赖具体的T类型
```

- 返回值类型后置

```c
template <typename T, typename U>
decltype(t+u) add(T t, U u)
{
    return t + u;
}
```

当我们在编译器中将这几行代码改出来后就直接报错了，因此 decltype 中的 t 和 u 都是函数参数，直接这样写相当于变量还没有定义就直接用上了，这时候变量还不存在，有点心急了。

C++11中增加了返回类型后置语法，说明白一点就是将decltype和auto结合起来完成返回类型的推导。其语法格式如下:
```c
// 符号 -> 后边跟随的是函数返回值的类型
auto func(参数1, 参数2, ...) -> decltype(参数表达式)
```

通过对上述返回类型后置语法代码的分析，得到结论：auto 会追踪 decltype() 推导出的类型，因此上边的 add() 函数可以做如下的修改：

```c

#include <iostream>
using namespace std;

template <typename T, typename U>
// 返回类型后置语法
auto add(T t, U u) -> decltype(t+u) 
{
    return t + u;
}

int main()
{
    int x = 520;
    double y = 13.14;
    // auto z = add<int, double>(x, y);
    auto z = add(x, y);     // 简化之后的写法
    cout << "z: " << z << endl;
    return 0;
}
```

## Lambda表达式

- 声明式的编程风格：就地匿名定义目标函数或函数对象，不需要额外写一个命名函数或函数对象。
- 简洁：避免了代码膨胀和功能分散，让开发更加高效。
- 在需要的时间和地点实现功能闭包，使程序更加灵活。

lambda 表达式定义了一个匿名函数，并且可以捕获一定范围内的变量。lambda 表达式的语法形式简单归纳如下：
```c
[capture](params) opt -> ret {body;};
```
- 捕获列表 []: 捕获一定范围内的变量
- 参数列表 (): 和普通函数的参数列表一样，如果没有参数参数列表可以省略不写

- opt 选项， 不需要可以省略

    mutable: 可以修改按值传递进来的拷贝（注意是能修改拷贝，而不是值本身）
    exception: 指定函数抛出的异常，如抛出整数类型的异常，可以使用 throw ();

[] - 不捕捉任何变量
[&] - 捕获外部作用域中所有变量，并作为引用在函数体内使用 (按引用捕获)
[=] - 捕获外部作用域中所有变量，并作为副本在函数体内使用 (按值捕获)

使用 lambda 表达式捕获列表捕获外部变量，如果希望去修改按值捕获的外部变量，那么应该如何处理呢？这就需要使用 mutable 选项，被mutable修改是lambda表达式就算没有参数也要写明参数列表，并且可以去掉按值捕获的外部变量的只读（const）属性。

```c
int a = 0;
auto f1 = [=] {return a++; };              // error, 按值捕获外部变量, a是只读的
auto f2 = [=]()mutable {return a++; };     // ok
```
最后再剖析一下为什么通过值拷贝的方式捕获的外部变量是只读的:

    lambda表达式的类型在C++11中会被看做是一个带operator()的类，即仿函数。
    按照C++标准，lambda表达式的operator()默认是const的，一个const成员函数是无法修改成员变量值的。

mutable 选项的作用就在于取消 operator () 的 const 属性。

因为 lambda 表达式在 C++ 中会被看做是一个仿函数，因此可以使用std::function和std::bind来存储和操作lambda表达式：
```c
#include <iostream>
#include <functional>
using namespace std;

int main(void)
{
    // 包装可调用函数
    std::function<int(int)> f1 = [](int a) {return a; };
    // 绑定可调用函数
    std::function<int(int)> f2 = bind([](int a) {return a; }, placeholders::_1);

    // 函数调用
    cout << f1(100) << endl;
    cout << f2(200) << endl;
    return 0;
}
```
对于没有捕获任何变量的 lambda 表达式，还可以转换成一个普通的函数指针：
```c
using func_ptr = int(*)(int);
// 没有捕获任何外部变量的匿名函数
func_ptr f = [](int a)
{
    return a;  
};
// 函数调用
f(1314);
```


























>[C++11](https://subingwen.cn/cplusplus/)
