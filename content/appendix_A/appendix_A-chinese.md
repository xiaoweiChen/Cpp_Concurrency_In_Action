#附录A 对C++11语言特性的简要介绍

新的C++标准，不仅带来了对并发的支持，以及其他语言的特性和一些新的标准库。在本附录中，会给出对这些新特性进行简要介绍（这些特性被用在了线程库中，以及这本书的其余部分）。除了thread_local(详见A.8部分)以外，就没有与并发直接相关的了，但是对于多线程代码来说，它们都是很重要。我已经对列表进行的设定，只列出有必要的部分(例如，右值引用)，或能够使代码更容易理解。由于对新特性不熟，对用到某些特性的代码理解起来可能有一些困难；但是，当对这些特性渐渐熟知后，就能很容易的理解代码。由于C++11的应用越来越广泛，这些特性在代码中的使用也将会变越来越普遍。

话不多说，让我们从线程库中的右值引用开始，来熟悉对象之间所有权(线程，锁等等)的转移。

##A.1 右值引用

如果你从事过C++编程，你会对引用比较熟悉，C++的引用允许你为已经存在的对象创建一个新的名字。所有对新引用所做的访问和修改操作，都会影响它的原型。

例如：
```c++
int var=42;
int& ref=var;  // 创建一个var的引用
ref=99;
assert(var==99);  // 原型的值被改变了，因为引用被赋值了
```

目前为止，我们用过的所有引用都是左值引用——对左值的引用。lvalue这个词来自于C语言，指的是可以放在赋值表达式左边的事物——命名的对象，在栈上或者堆上分配的对象，或者其他对象的成员——有明确的内存地址。rvalue这个词也来源于C语言，指的是，可以出现在赋值表达式右侧的对象——例如，文字常量和临时变量。因此，左值引用只能被绑定在左值上而不是右值。不能这样写：

```c++
int& i=42;  // 编译失败
```

例如，因为42是一个右值。好吧，这有些假；你可能通常使用下面的方式讲一个右值绑定到一个const左值引用上：

```c++
int const& i = 42;
```

这算是钻了标准的一个空子吧。不过，这种情况我们之前也介绍过，我们通过对左值的const引用创建临时性对象，作为参数传递给函数。其允许隐式转换，所以你可这样写：

```c++
void print(std::string const& s);
print("hello");  //创建了临时std::string对象
```

无论如何，C++11标准介绍了右值引用(*rvalue reference*)，这种方式只能绑定右值，不能绑定左值，其通过两个`&&`来进行声明：

```c++
int&& i=42;
int j=42;
int&& k=j;  // 编译失败
```

因此，可以使用函数重载的方式来确定：函数有左值或右值为参数的时候，看其能否被同名且对应参数为左值或有值引用的函数所重载。

其基础就是C++11新添的语义——移动语义(*move semantics*)。

###A.1.1 移动语义

右值通常都是临时的，所以可以随意修改；如果你知道函数的某个参数是一个右值，你可以将其看作为一个临时存储，或就算“窃取”其内容，也不会不影响程序的正确性。这就意味着，比起拷贝右值参数的内容，不如移动其内容。动态数组比较大的时候，这样做能节省很多内存分配，也提供了更多的优化空间。试想，一个函数以`std::vector<int>`作为一个参数，就需要将其拷贝进来，而不对原始的数据做任何操作。C++03/98的办法是，将这个参数作为一个左值的const引用传入，然后做内部拷贝：

```c++
void process_copy(std::vector<int> const& vec_)
{
  std::vector<int> vec(vec_);
  vec.push_back(42);
}
```

这就允许函数能以左值或右值的形式进行传递，不过任何情况下都是通过拷贝来完成的。如果使用右值引用的函数版本来重载这个函数，就能避免在传入右值的时候，函数会进行内部拷贝的过程，因为你知道你可以任意的对原始值进行修改：

```c++
void process_copy(std::vector<int> && vec)
{
  vec.push_back(42);
}
```

现在，如果这个问题存在于类的构造函数中，你可以窃取内部右值，在新的实例中使用。可以参考一下清单中的例子(默认构造函数会分配很大一块内存，其会在析构函数中释放)。

清单A.1 使用移动构造函数的类
```c++
class X
{
private:
  int* data;

public:
  X():
    data(new int[1000000])
  {}

  ~X()
  {
    delete [] data;
  }

  X(const X& other):  // 1
   data(new int[1000000])
  {
    std::copy(other.data,other.data+1000000,data);
  }
  
  X(X&& other):  // 2
    data(other.data)
  {
    other.data=nullptr;
  }
};
```

一般情况下，拷贝构造函数①都是这么定义：分配一块新内存，然后将数据拷贝进去。不过，你现在有了一个新的构造函数，可以接受右值引用来获取老数据②。这就是移动构造函数。在这个例子中，你只是将指针拷贝到数据中，并且将other以空指针的形式，实例留在了新实例中；使用右值里创建变量，就能避免了空间和时间上的多余消耗。

X类(清单A.1)中的移动构造函数，仅作为一次优化；不过在其他例子中，有些类型的构造函数只支持移动构造函数，而不知道拷贝构造函数。例如，智能指针`std::unique_ptr<>`的非空实例中，只有这个指针指向其对象，所以拷贝函数在这里就不能用了(如果使用拷贝函数，就会有两个`std::unique_ptr<>`指向该对象，这是不满足`std::unique_ptr<>`定义的)。不过，移动构造函数允许对指针的所有权，在实例之间进行传递，并且允许`std::unique_ptr<>`像一个带有返回值的函数一样使用——指针的转移是通过移动，而非拷贝。

如果你已经知道，某个变量在之后就不会在用到了，这时候可以选择显示的移动，你可以使用`static_cast<X&&>`将对应变量转换为右值，或者通过调用`std::move()`函数来做这件事：

```c++
X x1;
X x2=std::move(x1);
X x3=static_cast<X&&>(x2);
```

当你想要将参数值，不通过拷贝，转化为本地变量，或成员变量时，就可以使用这个办法；虽然右值引用参数绑定了右值，不过在函数内部，其会当做左值来处理：

```c++
void do_stuff(X&& x_)
{
  X a(x_);  // 拷贝
  X b(std::move(x_));  // 移动
}
do_stuff(X());  // ok，右值绑定到右值引用上
X x;
do_stuff(x);  // 错误，左值不能绑定到右值引用上
```

移动语义在线程库中用的比较广泛，对于无拷贝操作的时候，对数据进行转移，还有作为一种优化方式，避免对将要被销毁的变量，进行额外的拷贝。你已经在2.2节中看到，使用`std::move()`来转移`std::unique_ptr<>`实例的到一个新创建的线程中；在2.3节中，我们也了解了使用移动语义在`std:thread`的实例间，转移线程的所有权。

`std::thread`、`std::unique_lock<>`、`std::future<>`、 `std::promise<>`和`std::packaged_task<>`都不能拷贝，不过这些类都有移动构造函数，能让相关资源在实例中进行传递，并且其也支持用一个函数将值进行返回。`std::string`和`std::vector<>`都是可以拷贝的，不过它们也有移动构造函数和移动赋值操作符，就是为了避免拷贝从右值拷贝大量的数据。

C++标准库不会将一个对象显示的转移到另一个对象中，除非要将其销毁的时候，或对其赋值的时候(拷贝和移动的操作很相似)。不过，实践中表现良好，移动能保证类中的所有状态保持不变。一个`std::thread`实例可以作为移动源，转移到新创建(以默认构造方式)的`std::thread`实例中。还有，`std::string`可以通过移动原始数据进行构造，并且保留原始数据正确的状态，不过不能保证的是原始数据中该状态是否正确(这样更具字符串的长度或字符的数量决定)。

###A.1.2 右值引用和函数模板

不过在使用右值引用作为函数模板的参数时，还是之前的用法有些不同的：如果模板函数参数以右值引用作为一个模板参数，当在对应位置提供左值的时候，模板会自动将其类型认定为左值引用；当提供右值的时候，其会当做普通数据使用。这可能有些口语化，那就来看几个例子吧。

考虑一下下面的函数模板：

```c++
template<typename T>
void foo(T&& t)
{}
```

当你随后传入一个右值，T的类型将被推导为：

```c++
foo(42);  // foo<int>(42)
foo(3.14159);  // foo<double><3.14159>
foo(std::string());  // foo<std::string>(std::string())
```

不过，在向foo传入左值的时候，T会比推导为一个左值引用：

```c++
int i = 42;
foo(i);  // foo<int&>(i)
```

因为函数参数声明为T&&，所以就是引用的引用，可以视为是原始的引用类型。那么foo<int&>()就相当于：

```c++
foo<int&>(); // void foo<int&>(int& t);
```

这就允许一个函数模板可以即接受左值，又可以接受右值参数；这种方式已经被`std::thread`的构造函数所使用(2.1节和2.2节)，所以能够将可调用对象移动到内部存储，而非当参数是右值的时候进行拷贝。

##A.2 删除函数

有时让类去做拷贝是没有意义的。`std::mutex`就是一个最好的例子——拷贝一个互斥量，有什么意义呢？`std::unique_lock<>`是另一个例子——一个实例只能拥有一个锁。如果要复制，那拷贝的那个实例也能获取相同的锁，这样`std::unique_lock<>`就没有存在的意义了。在实例中转移所有权(A.1.2节)，是有意义的，其并不是使用的拷贝。当然其他例子就不一一列举了。

通常为了避免进行拷贝操作，会将拷贝构造函数和拷贝赋值操作符声明为私有成员，并且不进行实现。如果对实例进行拷贝，将会引起编译错误；如果有其他成员函数或友元函数想要拷贝一个实例，那将会引起链接错误(因为缺少实现)：

```c++
class no_copies
{
public:
  no_copies(){}
private:
  no_copies(no_copies const&);  // 无实现
  no_copies& operator=(no_copies const&);  // 无实现
};

no_copies a;
no_copies b(a);  // 编译错误
```

在C++11中，委员会意识到这种情况，但是没有意识到其会带来攻击性。因此，委员会提供了更多的通用机制，以用于其他情况：你可以通过添加`= delete`将一个函数声明为删除函数。no_copise类就可以写为：

```c++
class no_copies
{
public:
  no_copies(){}
  no_copies(no_copies const&) = delete;
  no_copies& operator=(no_copies const&) = delete;
};
```

这样的描述要比之前的代码更加清晰。这也会允许编译器提供更多的错误信息描述，当有成员函数想要执行拷贝操作的时候，可将连接错误转移到编译时。

当将拷贝构造和拷贝赋值操作删除后，你还需要显式写一个移动构造函数和移动赋值操作符，与`std::thread`和`std::unique_lock<>`一样，你的类就是只移动的。下面清单中的例子，就展示了一个只移动的类。

清单A.2 一个简单的只移动类型
```c++
class move_only
{
  std::unique_ptr<my_class> data;
public:
  move_only(const move_only&) = delete;
  move_only(move_only&& other):
    data(std::move(other.data))
  {}
  move_only& operator=(const move_only&) = delete;
  move_only& operator=(move_only&& other)
  {
    data=std::move(other.data);
    return *this;
  }
};

move_only m1;
move_only m2(m1);  // 错误，拷贝构造声明为“已删除”
move_only m3(std::move(m1));  // OK，找到移动构造函数
```

只移动对象可以作为函数的参数进行传递，并且从函数中返回，不过当你想要移动左值，你通常需要显式的使用`std::move()`或`static_cast<T&&>`。

你可以为任意函数添加`= delete`说明符，添加后就说明这些函数是不能使用的。当然，还可以用于很多的地方；删除函数可以以正常的方式参与重载解析，并且如果被使用，也只会引起编译错误。这种方式可以用来删除特定的重载。比如，当你的函数以short作为参数，为了避免扩展为int类型，你可以写出重载函数(以int为参数)的声明，然后添加删除说明符：

```c++
void foo(short);
void foo(int) = delete;
```

现在，任何向foo函数传递int类型参数都会残生一个编译错误，不过调用者可以显式的将其他类型转化为short：

```c++
foo(42);  // 错误，int重载声明已经删除
foo((short)42);  // OK
```

##A.3 默认函数

删除函数可以让你对一个显示声明的函数不进行实现，默认函数就完全相反：其会让编译器为你创建函数实现，通常都是“默认”实现。当然，这些函数你可以直接使用，它们都会自动生成：默认构造函数，析构函数，拷贝构造函数，移动构造函数，拷贝赋值操作符和移动赋值操作符。

为什么要这样做呢？这里列出一些原因：

- 改变函数的可访问性——编译器生成的默认函数通常都是声明为public。如果你想让其为protected或private成员，你必须自己去实现它们。通过将其声明为默认，你可以让编译器来帮助你实现函数和改变访问级别。

- 作为文档——当编译器生成版本已经足够使用，那么将其进行显式声明就利于你或其他人在之后对这段代码的阅读，这会让代码看起来很清晰。

- 让编译器自动生成函数，当这些函数没有单独实现的时候——通常会由默认构造函数来做这件事，如果用户没有定义构造函数，编译器将会生成一个。当你需要自定一个拷贝构造函数时(假设)，如果将其声明为默认，你也可以获得编译器为你实现的拷贝构造函数。

- 让编译器生成虚析构函数。

- 声明一个特殊版本的拷贝构造函数，比如：参数类型是非const引用，而不是const引用。

- 利用编译生成函数的特殊性质，如果你提供了对应的函数，其将会失效——会在后面具体讲解。

就像删除函数是在函数后面添加`= delete`一样，默认函数需要在函数后面添加`= default`，例如：

```c++
class Y
{
private:
  Y() = default;  // 改变访问级别
public:
  Y(Y&) = default;  // 以非const引用作为参数
  T& operator=(const Y&) = default; // 作为文档的形式，声明为默认函数
protected:
  virtual ~Y() = default;  // 改变访问级别，以及添加虚函数标签
};
```

之前编译器生成函数都有独特的特性，这是用户定义版本所不具备的。最大的区别就是编译器生成的函数都是简单的。这里列出了几点重要的特性：

- 对象具有简单的拷贝构造函数，拷贝赋值操作符和析构函数，其都能通过memcpy或memmove进行拷贝。

- 字面类型用于constexpr函数(可见A.4节)，必须有简单的构造，拷贝构造和析构函数。

- 类的默认构造，拷贝，拷贝赋值操作符合析构函数也可以用在，一个已有构造和析构函数的用户定义的联合体内。

- 类的简单拷贝赋值操作符可以使用`std::atomic<>`类型模板(见5.2.6节)，就是为了某种类型的值提供原子操作。

仅添加`= default`不会让函数变得简单——如果类还支持其他相关标准的函数，那这个函数就是简单的——不过用户显式的实现就不会让这些函数变简单。

第二个区别在于，编译器生成函数和用户提供的函数等价，也就是类中无用户提供的构造函数可以看作为一个aggregate，并且可以通过聚合初始化函数进行初始化：

```c++
struct aggregate
{
  aggregate() = default;
  aggregate(aggregate const&) = default;
  int a;
  double b;
};
aggregate x={42,3.141};
```

在这个例子中，x.a被42初始化，x.b被3.141初始化。

第三个不同点，编译器生成的函数只适用于构造函数，换句话说，只适用于符合某些标准的默认构造函数。

```c++
struct X
{
  int a;
};
```

如果你创建了一个X的实例，而未初始化，其中int(a)将会被默认初始化。如果对象有静态存储过程，那么a将会被初始化为0；另外，当a没有赋值的时候，其不定值可能会触发未定义行为：

```c++
X x1;  // x1.a的值不明确
```

另外，当你使用显示调用构造函数的方式对X进行初始化，那么a就会被初始化为0：

```c++
X x2 = X();  // x2.a == 0
```

这种奇怪的属性会扩展到基础类和成员函数中。当类的默认构造函数是由编译器提供，并且一些数据成员和基类都是有编译器提供默认构造函数时，基类的数据成员和该类中的数据成员都是内置类型的时候，其值要不就是不确定的，要不就是被初始化为0，这与默认构造函数是否能被显式调用有关。

虽然这条规则令人困惑，并且容易造成错误，不过其也很有用，并且当你编写构造函数的时候，就不会用到这个特性；数据成员，通常都可以被初始化(因为你为其指定了一个值，或调用了显式构造函数)，或不会被初始化(因为不需要)：

```c++
X::X():a(){}  // a == 0
X::X():a(42){}  // a == 2
X::X(){}  // 1
```

在第三个例子中①，你省略了对a的初始化，X中a就是一个未被初始化的非静态实例，被初始化的X实例都会有静态存储过程。

在通常的情况下，如果手动写了其他构造函数，编译器就不会在为你生成默认构造函数了。所以，当你想要自己写一个的时候，就意味着你放弃了这种奇怪的初始化属性。不过，将构造函数显示声明成默认的，你就能强制编译器为你生成一个默认构造函数了，并且刚才说的那种特性将会保留：

```c++
X::X() = default;  // 应用默认初始化规则
```

这种特性用于原子变量(见5.2节)，默认构造函数显式为默认的。他们的初始值通常都是没有定义的，除非他们具有(a)一个静态存储的过程(静态初始化为0)，或(b)显式调用默认构造函数，将成员初始化为0，亦或(c)指定一个特殊的值。注意，在这种情况下的原子变量，为了允许静态初始化过程，构造函数会通过一个声明为constexpr(见A.4节)的值为原子变量进行初始化。

##A.4 常量表达式函数

整型字面值，例如42，就是常量表达式。所以，简单的数学表达式，例如，23x2-4。可以使用其来初始化const整型变量，然后将const整型变量作为新表达的一部分：

```c++
const int i=23;
const int two_i=i*2;
const int four=4;
const int forty_two=two_i-four;
```

处理使用常量表达式创建变量也可用在其他常量表达式中，有些事情只能用常量表达式去做：

- 指定数组边界：
```c++
int bounds=99;
int array[bounds];  // 错误，bounds不是一个常量表达式
const int bounds2=99;
int array2[bounds2];  // 正确，bounds2是一个常量表达式
```

- 指定非类型模板参数的值：
```c++
template<unsigned size>
struct test
{};
test<bounds> ia;  // 错误，bounds不是一个常量表达式
test<bounds2> ia2;  // 正确，bounds2是一个常量表达式
```

- 可以对类中static const整型成员变量进行初始化：
```c++
class X
{
  static const int the_answer=forty_two;
};
```

- 可以对内置类型进行初始化，或可用于静态初始化集合：
```c++
struct my_aggregate
{
  int a;
  int b;
};
static my_aggregate ma1={forty_two,123};  // 静态初始化
int dummy=257;
static my_aggregate ma2={dummy,dummy};  // 动态初始化
```

- 静态初始化可以避免初始化顺序和条件变量的问题。

这些都不是新添加的——你可以在1998版本的C++标准中找到对应上面实例的条款。不过，在新标准中常量表达式进行了扩展，并添加了新的关键字`constexpr`。

关键字`constexpr`会对功能进行修改。当参数和函数返回类型符合要求，并且实现很简单，那么这样的函数就能够被声明为`constexpr`，这样函数可以当做常数表达式来使用：

```c++
constexpr int square(int x)
{
  return x*x;
}
int array[square(5)];
```

在这个例子中，array有25个元素，因为square函数的声明为`constexpr`。当然，这种方式可以被当做常数表达式来使用，不以为着什么情况下都是能够自动转换为常数表达式的：

```c++
int dummy=4;
int array[square(dummy)];  // 错误，dummy不是常数表达式
```

例子中，dummy不是常数表达式，所以square(dummy)也不是——其就是一个普通函数调用——所以其不能用来指定array的边界。

###A.4.1 常量表达式和自定义类型

目前为止的例子都是以内置int型展开的。不过，在新C++标准库中，对于满足字面类型要求的任何类型，都可以用常量表达式来表示。要想划分到字面类型中，需要满足一下几点：

- 一般的拷贝构造函数。

- 一般的析构函数。

- 所有成员变量都是非静态的，且基类需要是一个一般类型。

- 必须具有一个一般的默认构造函数，或一个constexpr构造函数。

后面会来看一下constexpr构造函数。现在，我们先将注意力集中在默认构造函数上，就像下面清单中的CX类一样。

清单A.3（一般)默认构造函数的类
```c++
class CX
{
private:
  int a;
  int b;
public:
  CX() = default;  // 1
  CX(int a_, int b_):  // 2
    a(a_),b(b_)
  {}
  int get_a() const
  {
    return a;
  }
  int get_b() const
  {
    return b;
  }
  int foo() const
  {
    return a+b;
  }
};
```

注意，这里我们显式的声明了默认构造函数①(见A.3节)，是为了保存用户定义的构造函数②。因此，这种类型符合字面类型的要求，就可以将其用在常量表达式中了。你可以提供一个constexpr函数来创建一个实例，例如：

```c++
constexpr CX create_cx()
{
  return CX();
}
```

也可以创建一个简单的constexpr函数来拷贝参数：

```
constexpr CX clone(CX val)
{
  return val;
}
```

不过，constexpr函数只有其他constexpr函数可以进行调用。在CX类中声明成员函数和构造函数为constexpr：

```c++
class CX
{
private:
  int a;
  int b;
public:
  CX() = default;
  constexpr CX(int a_, int b_):
    a(a_),b(b_)
  {}
  constexpr int get_a() const  // 1
  {
    return a;
  }
  constexpr int get_b()  // 2
  {
    return b;
  }
  constexpr int foo()
  {
    return a+b;
  }
};
```

注意，现在const对于get_a()①来说就是多余的，因为其暗示在使用constexpr.get_b()就已经是是const了，所以const描述符在这里会被忽略。这样就允许更多复杂的constexpr函数存在：

```c++
constexpr CX make_cx(int a)
{
  return CX(a,1);
}
constexpr CX half_double(CX old)
{
  return CX(old.get_a()/2,old.get_b()*2);
}
constexpr int foo_squared(CX val)
{
  return square(val.foo());
}
int array[foo_squared(half_double(make_cx(10)))];  // 49个元素
```

这里的函数都很有趣，如果你想要计算数组的边界或是一个整型常量，你就需要使用这种方式。最大的好处是常量表达式和constexpr函数会设计到用户定义类型的对象，可以使用这些函数，对这些对象进行初始化。因为常量表达式的初始化过程是静态初始化，所以这种方式就能避免条件竞争和初始化顺序的问题：

```c++
CX si=half_double(CX(42,19));  // 静态初始化
```

这也包含构造函数。当构造函数被声明为constexpr，并且构造函数参数是常量表达式，那么初始化过程就是常数初始化，可能作为静态初始化的一部分。随着并发的发展，这也C++11标准中的一个重要改变：允许用户定义构造函数可以进行静态初始化，就可以在初始化的时候避免条件竞争，因为静态过程能保证初始化过程在代码运行前进行。

特别是关于`std::mutex`(见3.2.1节)，或`std::atomic<>`(见5.2.6节)，当你可能想要使用一个全局实例来同步其他变量的访问，这样同步访问就能避免条件竞争的发生。在构造函数中，互斥量产生条件竞争是不可能的，因此对于`std::mutex`的默认构造函数就应该被声明为constexpr，就是为了保证互斥量初始化过程是一个静态初始化过程的一部分。

###A.4.2 常量表达式对象

目前，我们已经了解了constexpr在函数上的应用。constexpr也可以用在对象上，主要是用来做判断的；其验证对象是否是使用常量表达式，constexpr构造函数，或组合常量表达式，进行初始化的。且这个对象需要被声明为const：

```c++
constexpr int i=45;  // ok
constexpr std::string s(“hello”);  // 错误，std::string不是字面类型

int foo();
constexpr int j=foo();  // 错误，foo()没有声明为constexpr
```

###A.4.3 常量表达式函数的要求

为了将一个函数声明为constexpr，也是有几点要求的；当不满足这些要求，constexpr声明将会报编译错误。

要求如下：

- 所有参数都必须是字面类型。

- 返回类型必须是字面类型。

- 函数体内必须有一个return。

- return的表达式需要满足常量表达式的要求。

- 用来构造返回值/表达式的任何构造函数或转换操作，都需要是constexpr。

看起来很简单；你必须要内联函数到常量表达式中，并且需要返回的还是个常量表达式，还有不能对任何东西进行改动。constexpr函数就是无害的纯洁的函数。

对于constexpr类成员函数，需要追加几点要求：

- constexpr成员函数不能是虚函数。

- 在对应类必须有字面类的成员。

对于constexpr构造函数的规则也有些不同：

- 构造函数体必须是空的。

- 每一个基类必须是可初始化的。

- 每个非静态数据成员都需要初始化。

- 使用于成员初始化列表的任何表达式，必须是常量表达式。

- 构造函数可选择要进行初始化的数据成员，并且基类必须有constexpr构造函数。

- 任何用于构建数据成员的构造函数和转换操作，以及和初始化表达式相关的基类必须为constexpr。

这些条件同样适用于成员函数，除非函数没有返回值，也就没有return语句。另外，构造函数对初始化列表中的所有基类和数据成员进行初始化。一般的拷贝构造函数会隐式的声明为constexpr。

###A.4.4 常量表达式和模板

当constexpr应用于函数模板，或一个类模板的成员函数，根据参数如果模板的返回类型不是字面类，那么编译器会忽略其常量表达式的声明。当模板参数类型合适，且是一般inline函数，就可以将类型写成constexpr类型的函数模板。下面就是个例子：

```c++
template<typename T>
constexpr T sum(T a,T b)
{
  return a+b;
}
constexpr int i=sum(3,42);  // ok，sum<int>是constexpr
std::string s=
  sum(std::string("hello"),
      std::string(" world"));  // 也行，不过sum<std::string>就不是constexpr了
```

函数需要满足所有constexpr函数所需的条件。不能用多个constexpr来声明一个函数，因为其是一个模板；这样也会带来一些编译错误。

##A.5 Lambda函数

Lambda函数在C++11中的加入是令人兴奋的，因为Lambda函数能够大大简化代码复杂度，避免实现调用对象。C++11的lambda函数语法允许在需要使用的时候进行定义。能为等待函数，例如`std::condition_variable`(如同4.1.1节中的例子)，提供很好谓词函数，因为其语义可以用来快速的表示可访问的变量，而非使用类中函数，来对成员变量进行捕获。

最简单的情况下，lambda表达式就一个自给自足的函数，不需要传入函数，仅依赖管局变量和函数。其甚至都可以不用返回一个值。这样的lambda表达式的一系列语义都需要封闭在括号中，还要以方括号作为前缀(lambda的介绍)：

```c++
[]{  // lambda表达式以[]开始
  do_stuff();
  do_more_stuff();
}();  // 表达式结束，可以直接调用
```

在这个例子中，lambda表达式通过后面的括号调用，不过这种方式不常用。一方面，如果想要直接调用，你可以在写完对应的语句后，就对函数进行调用。对于函数模板，传递一个参数进去时很常见的事情，甚至都可以将可调用对象作为其参数传入，可调用对象通常也需要一些参数，或返回一个值，亦或两者都有。如果你想给lambda函数传递参数，你可以参考下面的lambda函数，其使用起来就像是一个普通函数。例如，下面代码是将vector中的元素使用`std::cout`进行打印：

```c++
std::vector<int> data=make_data();
std::for_each(data.begin(),data.end(),[](int i){std::cout<<i<<"\n";});
```

返回值也是很简单的。当lambda函数体包括一个return语句，返回值的类型就作为lambda表达式的返回类型。例如，你可以使用一个简单的lambda函数，用来等待`std::condition_variable`(见4.1.1节)中的标志被设置。

清单A.4 lambda简单的推导返回类型
```c++
std::condition_variable cond;
bool data_ready;
std::mutex m;
void wait_for_data()
{
  std::unique_lock<std::mutex> lk(m);
  cond.wait(lk,[]{return data_ready;});  // 1
}
```

lambda的返回值传递给cond.wait()①，函数就呢个推断出data_ready的类型是bool。当条件变量从等待中苏醒后，在上锁阶段会调用lambda函数，并且当data_ready为true时，仅返回到wait()中。

当你的lambda函数体中有多个return语句，这种情况下，就需要显式的指定返回类型。在只有一个返回语句的时候，你也可以这样做，不过这样可能会让你的lambda函数体看起来更复杂。返回类型可以使用跟在参数列表后面的箭头(->)进行设置。如果你的lambda函数没有任何参数，你还需要包含(空)的参数列表，这样做是为了能显示的对返回类型进行指定。对条件变量的预测可以写成下面这种方式：

```c++
cond.wait(lk,[]()->bool{return data_ready;});
```

还可以对lambda函数进行扩展，比如加上log信息的打印，或做更加复杂的操作：

```c++
cond.wait(lk,[]()->bool{
  if(data_ready)
  {
    std::cout<<”Data ready”<<std::endl;
    return true;
  }
  else
  {
    std::cout<<”Data not ready, resuming wait”<<std::endl;
    return false;
  }
});
```

虽然简单的lambda函数很强大，并且能简化代码，不过其真正的强大的地方在于对本地变量的捕获。

###A.5.1 引用本地变量的Lambda函数

Lambda函数使用空的`[]`(lambda introducer)就不能引用当前范围内的本地变量；其只能使用全局变量，或将其以参数的形式进行传递。当你想要方位一个本地变量，你需要对其进行捕获。最简单的方式就是将范围内的所有本地变量都进行捕获，使用`[=]`就可以完成这样的功能。在函数被创建的时候，就能对本地变量的副本进行访问了。

实践一下，来考虑一下下面的例子：

```c++
std::function<int(int)> make_offseter(int offset)
{
  return [=](int j){return offset+j;};
}
```

每当调用make_offseter时，就会通过`std::function<>`函数包装，返回一个新的lambda函数体。这个带有返回的函数添加了对参数的偏移功能。例如：

```c++
int main()
{
  std::function<int(int)> offset_42=make_offseter(42);
  std::function<int(int)> offset_123=make_offseter(123);
  std::cout<<offset_42(12)<<”,“<<offset_123(12)<<std::endl;
  std::cout<<offset_42(12)<<”,“<<offset_123(12)<<std::endl;
}
```

屏幕上将打印出54,135两次，因为第一次从make_offseter中返回，都是对参数加42的；第二次调用后，make_offseter会对参数加上123。所以，会打印两次相同的值。

这种本地变量捕获的方式相当安全；所有的东西都进行了拷贝，所以你可以通过lambda函数对表达式的值进行返回，并且可以在原始函数之外的地方对其进行调用。这也不是唯一的选择；你也可以通过选择通过引用的方式捕获本地变量。在这种情况下，当本地变量被销毁的时候，lambda函数会出现未定义的行为。

下面的例子，就介绍一下怎么使用`[&]`对所有本地变量进行引用：

```c++
int main()
{
  int offset=42;  // 1
  std::function<int(int)> offset_a=[&](int j){return offset+j;};  // 2
  offset=123;  // 3
  std::function<int(int)> offset_b=[&](int j){return offset+j;};  // 4
  std::cout<<offset_a(12)<<”,”<<offset_b(12)<<std::endl;  // 5
  offset=99;  // 6
  std::cout<<offset_a(12)<<”,”<<offset_b(12)<<std::endl;  // 7
}
```

在之前的例子中，我们使用`[=]`来对要偏移的变量进行拷贝，offset_a函数就是个使用`[&]`捕获offset的引用的例子②。所以，offset初始化成42也没什么关系①；offset_a(12)的例子通常会依赖与当前offset的值。在③上，offset的值会变为123，offset_b④函数将会使用到这个值，同样第二个函数也是使用引用的方式。

现在，第一行打印信息⑤，offset为123，所以输出为135,135。不过，第二行打印信息⑦就有所不同，offset变成99⑥，所以这是的输出为111,111。offset_a和offset_b都对当前值进行了加12的操作。

尘归尘，土归土。C++还是C++，这些选项不会让你感觉到特别困惑；你可以选择以引用或拷贝的方式对变量进行捕获，并且你还可以通过调整中括号中的表达式，来对特定的变量进行显式捕获。如果你想要拷贝所有变量，而非一两个，你可以使用`[=]`，通过参考中括号中的符号，对变量进行捕获。下面的例子将会打印出1239，因为i是拷贝进lambda函数中的，而j和k是通过引用的方式进行捕获的：

```c++
int main()
{
  int i=1234,j=5678,k=9;
  std::function<int()> f=[=,&j,&k]{return i+j+k;};
  i=1;
  j=2;
  k=3;
  std::cout<<f()<<std::endl;
}
```

或者，你也可以通过默认引用方式对一些变量做引用，而对一些特别的变量进行拷贝。在这种情况下，就要使用`[&]`与拷贝符号相结合的方式对列表中的变量进行拷贝捕获。下面的例子将打印出5688，因为i通过引用捕获，但j和
通过拷贝捕获：

```c++
int main()
{
  int i=1234,j=5678,k=9;
  std::function<int()> f=[&,j,k]{return i+j+k;};
  i=1;
  j=2;
  k=3;
  std::cout<<f()<<std::endl;
}
```

如果你只想捕获某些变量，那么你可以忽略=或&，仅使用变量名进行捕获就行；加上&前缀，是将对应变量以引用的方式进行捕获，而非拷贝的方式。下面的例子将打印出5682，因为i和k是通过引用的范式获取的，而j是通过拷贝的方式：

```c++
int main()
{
  int i=1234,j=5678,k=9;
  std::function<int()> f=[&i,j,&k]{return i+j+k;};
  i=1;
  j=2;
  k=3;
  std::cout<<f()<<std::endl;
}
```

最后一种方式，是为了确保预期的变量能被捕获，在捕获列表中，引用任何不存在的变量都会引起编译错误。当你选择以这种方式，你就要小心类成员的访问方式，确定类中是否包含一个lambda函数的成员变量。类成员变量是不能直接捕获的；如果你想通过lambda方式访问类中的成员，你需要在捕获列表中添加this指针，以便捕获。在下面的例子章，lambda捕获this，就能访问到some_data类中的成员：

```c++
struct X
{
  int some_data;
  void foo(std::vector<int>& vec)
  {
    std::for_each(vec.begin(),vec.end(),
         [this](int& i){i+=some_data;});
  }
};
```

在并发的上下文中，lambda是很有用的，其可以作为谓词放在`std::condition_variable::wait()`(见4.1.1节)和`std::packaged_task<>`(见4.2.1节)中；或是用在线程池中，对小任务进行打包。其也可以线程函数的方式`std::thread`的构造函数(见2.1.1)，以及作为一个并行算法实现，在parallel_for_each()(见8.5.1节)中使用。

##A.6 变参模板

变参模板，就是可以使用不定数量的参数进行特化的模板。就像你接触到的变参函数一样，printf就接受可变参数。现在，你就可以给你的模板指定不定数量的参数了。变参模板在整个C++线程库中都有使用。例如，`std::thread`的构造函数就是一个变参类模板。从使用者的角度看，仅知道模板可以接受无限个参数就够了，不过当要写这么一个模板，或对其工作原理很感兴趣时，就需要了解一些细节。

和变参函数一样，变参部分可以在参数列表章使用省略号`...`代表，变参模板需要在参数列表中使用省略号：

```c++
template<typename ... ParameterPack>
class my_template
{};
```

即使主模板不是变参模板，模板进行部分特化的类中，也可以使用可变参数模板。例如，`std::packaged_task<>`(见4.2.1节)的主模板就是一个简单的模板，这个简单的模板只有一个参数：

```c++
template<typename FunctionType>
class packaged_task;
```

不过，并不是所有地方都这样定义；对于部分特化模板来说，其就像是一个“占位符”：

```c++
template<typename ReturnType,typename ... Args>
class packaged_task<ReturnType(Args...)>;
```

这个部分特化的类就包含实际定义的类；第4章，你可以写一个`std::packaged_task<int(std::string,double)>`来声明一个以`std::string`和double作为参数的任务，当执行这个任务后，结果会由`std::future<int>`进行保存。

这个声明展示了两个变参模板的附加特性。第一个比较简单：普通模板参数(例如ReturnType)和可变模板参数(Args)可以同时声明。第二个特性，展示了`Args...`在特化类的模板参数列表中是如何使用的，就为了展示实例化模板中的Args的组成类型。实际上，因为这是部分特化，所以其作为一种模式进行匹配；在列表中出现的类型(被Args捕获)都会进行实例化。参数包(parameter pack)调用可变参数Args，并且使用`Args...`作为包的扩展。

和可变参函数一样，变参部分可能什么都没有，也可能有很多类型项。例如，在`std::packaged_task<my_class()>`中ReturnType参数就是my_class，并且Args参数包是空的，不过在`std::packaged_task<void(int,double,my_class&,std::string*)>`中，ReturnType为void，并且Args列表中的类型就有：int, double, my_class&和std::string*。

###A.6.1 扩展参数包

变参模板主要依靠包括扩展功能，因为你不能限制有更多的类型添加到模板参数中。首先，在列表中的参数类型使用到的时候，可以字节使用包扩展，比如需要给其他模板提供类型参数：

```c++
template<typename ... Params>
struct dummy
{
  std::tuple<Params...> data;
};
```

成员变量data是一个`std::tuple<>`实例，包含所有指定类型，所以dummy<int, double, char>的成员变量就为`std::tuple<int, double, char>`。可以将包扩展和普通类型相结合：

```c++
template<typename ... Params>
struct dummy2
{
  std::tuple<std::string,Params...> data;
};
```

这次，元组中添加了额外的(第一个)成员类型`std::string`。其优雅指出在于，你可以通过包扩展的方式创建一种模式，这种模式会在之后将每个元素拷贝到扩展之中。你可以使用`...`来表示扩展模式的结束。例如，比起创建使用参数包来创建元组中所有的元素，不如在元组中创建指针，或使用`std::unique_ptr<>`指针，指向对应元素：

```c++
template<typename ... Params>
struct dummy3
{
  std::tuple<Params* ...> pointers;
  std::tuple<std::unique_ptr<Params> ...> unique_pointers;
};
```

类型表达式会比较复杂，提供的参数包是在类型表达式中产生的，并且表达式中使用`...`作为扩展。当参数包已经扩展 ，在包中的每一项都会代替对应的类型表达式，在结果里列表中产生相应的数据项。因此，当参数包Params包含int，int，char类型，那么`std::tuple<std::pair<std::unique_ptr<Params>,double> ... >`将扩展为`std::tuple<std::pair<std::unique_ptr<int>,double>`,`std::pair<std::unique_ptr<int>,double>`,`std::pair<std::unique_ptr<char>, double> >`。如果包扩展被当做模板参数列表使用，那么模板就不需要变长的参数了；如果不需要了，那么参数包就要对模板参数的要求进行准确的匹配：

```c++
template<typename ... Types>
struct dummy4
{
  std::pair<Types...> data;
};
dummy4<int,char> a;  // 1 ok，为std::pair<int, char>
dummy4<int> b;  // 2 错误，无第二个类型
dummy4<int,int,int> c;  // 3 错误，类型太多
```

你可以使用包扩展的方式，对函数的参数进行声明：

```c++
template<typename ... Args>
void foo(Args ... args);
```

这将会创建一个新参数包args，其是一组函数参数，而非一组类型，并且这里`...`也能像之前一样进行扩展。例如，你可以在`std::thread`的构造函数中使用，使用右值引用的方式获取函数所有的参数(见A.1节)：

```c++
template<typename CallableType,typename ... Args>
thread::thread(CallableType&& func,Args&& ... args);
```

函数参数包也可以用来调用其他函数，将制定包扩展成参数列表，可匹配调用的函数。如同类型扩展一样，你也可以使用某种模式对参数列表进行扩展。例如，使用`std::forward()`以右值引用的方式来保存提供给函数的参数：

```c++
template<typename ... ArgTypes>
void bar(ArgTypes&& ... args)
{
  foo(std::forward<ArgTypes>(args)...);
}
```

注意一下这个例子，包扩展包括对类型包ArgTypes和函数参数包args的扩展，并且省略了其余的表达式。当这样调用bar函数：

```c++
int i;
bar(i,3.141,std::string("hello "));
```

将会扩展为

```c++
template<>
void bar<int&,double,std::string>(
         int& args_1,
         double&& args_2,
         std::string&& args_3)
{
  foo(std::forward<int&>(args_1),
      std::forward<double>(args_2),
      std::forward<std::string>(args_3));
}
```

这样就将第一个参数以左值引用的形式，正确的传递给了foo函数，其他两个函数都是以右值引用的方式传入的。

最后一件事，在参数包中使用`sizeof...`操作可以获取类型参数类型的大小。`sizeof...(p)`就是p参数包中所包含元素的个数。不管是类型参数包或函数参数包，结果都是一样的。这可能是你唯一一次在使用参数包的时候，没有加省略号；这里的省略号是作为`sizeof...`操作的一部分，所以不算是用到省略号。下面的函数会返回参数的数量：

```c++
template<typename ... Args>
unsigned count_args(Args ... args)
{
  return sizeof... (Args);
}
```

就像普通的sizeof操作一样，`sizeof...`的结果为常量表达式，所以其可以用来指定定义数组长度，等等。

##A.7 自动推导变量类型

c++是静态语言：所有变量的类型，都会在编译时被准确指定。所以，作为程序员你需要为每个变量指定对应的类型。有些时候，就需要使用一些繁琐类型定义，比如：

```c++
std::map<std::string,std::unique_ptr<some_data>> m;
std::map<std::string,std::unique_ptr<some_data>>::iterator
      iter=m.find("my key");
```

常规的解决办法是使用typedef来缩短类型名的长度，并消除类型不一致的问题。这种方式在C++11中仍然可行，不过这里要介绍一种新的解决办法：如果一个变量需要通过一个已初始化的变量类型来为其做声明，那么你就可以直接使用`auto`关键字。这样，编译器就会通过已初始化的变量，自动的去推断变量的类型。

```c++
auto iter=m.find("my key");
```

当然，`auto`还有很多种用法：你可以使用它来声明const、指针或引用变量。这里使用`auto`对相关类型进行了声明：

```c++
auto i=42; // int
auto& j=i; // int&
auto const k=i; // int const
auto* const p=&i; // int * const
```

变量类型的推导规则是建立一些语言规则基础上：函数模板参数。其声明形式如下：

```c++
some-type-expression-involving-auto var=some-expression;
```

var变量的类型与声明函数模板的参数的类型相同。要想替换`auto`，需要使用完整的类型参数：

```c++
template<typename T>
void f(type-expression var);
f(some-expression);
```

在使用`auto`的时候，数组类型将衰变为指针，引用将会被删除(除非将类型进行显式为引用)，比如：

```c++
int some_array[45];
auto p=some_array; // int*
int& r=*p;
auto x=r; // int
auto& y=r; // int&
```

这样能大大简化变量的声明过程，特别是在类型标识符特别长，或不清楚具体类型的时候(例如，调用函数模板，等到的目标值类型就是不确定的)。

##A.8 线程本地变量

线程本地变量允许程序中国的每个线程都有一个独立的实例拷贝。可以使用`thread_local`关键字来对这样的变量进行声明。命名空间内的变量，静态成员变量，以及本地变量都可以声明成线程本地变量，就是为了在线程运行前，对这些数据进行存储操作：

```c++
thread_local int x;  // 命名空间内的线程本地变量

class X
{
  static thread_local std::string s;  // 线程本地的静态成员变量
};
static thread_local std::string X::s;  // 这里需要添加X::s

void foo()
{
  thread_local std::vector<int> v;  // 一般线程本地变量
}
```

由命名空间或静态数据成员构成的线程本地变量，需要在线程单元对其进行使用**前**进行构建。有些实现，会将对现场本地变量的初始化过程，放在线程中去做；还有一些可能会在其他时间点做初始化，在一些有依赖的组合中，根据具体情况来进行初始化。当将没有构造好的线程本地变量传递给线程单元使用，就不能保证它们会在线程中进行构造。这就允许动加载带有线程本地变量的模块——这些变量首先需要在一个给定的线程中进行构造，之后其他线程就可以通过动态加载模块对线程本地变量进行引用。

在函数中声明的线程本地变量，需要使用一个给定线程进行初始化(通过控制流将这些声明传递给指定线程)。如果函数没有被指定线程调用，那么这个函数中声明的线程本地变量就不会构造。本地静态变量也是同样的情况，除非其单独的应用于每一个线程。

静态变量与线程本地变量会共享一些属性——它们可以做进一步的初始化(比如，动态初始化)；如果在构造线程本地变量时抛出异常，`srd::terminate()`就会将程序终止。

析构函数会在构造线程本地变量的那个线程返回时调用，析构顺序是构造的逆顺序。当初始化顺序没有指定时，确定析构函数和这些变量是否有相互依存关系就尤为重要了。当线程本地变量的析构函数抛出异常时，`std::terminate()`会被调用，将程序终止。

当线程调用`std::exit()`或从main()函数返回(这等价于调用`std::exit()`作为main()的“返回值”)时，线程本地变量也会为了这个线程进行销毁。当应用退出时，还有线程在运行，那么对于这些线程来说，线程本地变量的析构函数就没有被调用。

虽然，线程本地变量在不同线程上有不同的地址，不过还是可以获取指向这些变量的一般指针。指针会在线程中，通过获取地址的方式，引用相应的对象。当引用被销毁的对象时，会出现未定义行为，所以在向其他线程传递线程本地变量指针时，就需要保证，当指向对象所在的线程结束后，就不能对相应的指针进行解引用。

##A.9 小结

本附录仅是摘录了部分C++11标准的新特性，因为这些特性和线程库之间有着良好的互动。其他的新特性，包括：静态断言(static_assert)，强类型枚举(enum class)，委托构造函数，Unicode码支持，模板别名，以及统一的初始化序列。对于新功能的详细描述已经超出了本书的范围；需要另外一本书来进行详细介绍。对标准改动的最好的概述可能就是由Bjarne Stroustrup编写的《C++11FAQ》[1], 其他C++的参考书籍也会在以后对C++11标准进行覆盖。

希望这里的简短介绍，能让你了解这些新功能和线程库之间的关系，并且你在写多线程代码的时候能用到这些新功能。虽然，本附录为了新特性提供了足够简单的例子，不过还是一个简单的介绍，并非新功能的一份完整的参考或教程。如果你想在你的代码中大量使用这些新功能，我建议你去找相关权威的参考书或教程，了解更加详细的情况。

----------

【1】 http://www.research.att.com/~bs/C++0xFAQ.html