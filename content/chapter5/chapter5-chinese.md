# 第5章 C++内存模型和原子类型操作

**本章主要内容**

- C++11内存模型详解<br>
- 标准库提供的原子类型<br>
- 使用各种原子类型<br>
- 原子操作实现线程同步功能<br>

C++11标准中，有一个十分重要特性，常被程序员们所忽略。它不是一个新的语法特性，也不是新的工具，它就是新的多线程(感知)内存模型。内存模型没有明确的定义基本部件应该如何工作的话，之前介绍的那些工具就无法正常工作。那为什么大多数程序员都没有注意到它呢？当你使用互斥量保护你的数据和条件变量，或者是“期望”上的信号事件时，对于互斥量*为什么*能起到这样作用，大多数人不会去关心。只有当你试图去“接触硬件”，你才能详尽的了解到内存模型是如何起作用的。

C++是一个系统级别的编程语言，标准委员会的目标之一就是不需要比C++还要底层的高级语言。C++应该向程序员提供足够的灵活性，无障碍的去做他们想要做的事情；当需要的时候，可以让他们“接触硬件”。原子类型和原子操作就允许他们“接触硬件”，并提供底层级别的同步操作，通常会将常规指令数缩减到1~2个CPU指令。

在本章，我们将讨论内存模型的基本知识，而后再了解一下原子类型和操作，最后了解与原子类型操作相关的各种同步。这个过程可能会比较复杂：如果你已经打算使用原子操作(比如，第7章的无锁数据结构)同步你的代码，那么你就没有必要了解过多的细节。

现在让我们轻松愉快的来看一下有关内存模型的基本知识。

## 5.1 内存模型基础

这里从两方面来讲内存模型：一方面是基本结构，这个结构奠定了与内存相关的基础；另一方面就是并发。基本结构对于并发也是很重要的，特别是当你阅读到底层原子操作的时候，所以我将会从基本结构讲起。在C++中，它与所有的对象和内存位置有关。

### 5.1.1 对象和内存位置

在一个C++程序中的所有数据都是由对象(objects)构成。这不是说你可以创建一个int的衍生类，或者是基本类型中存在有成员函数，或是像在Smalltalk和Ruby语言下讨论程序那样——“一切都是对象”。“对象”仅仅是对C++数据构建块的一个声明。C++标准定义类对象为“存储区域”，但对象还是可以将自己的特性赋予其他对象，比如，其类型和生命周期。

像int或float这样的对象就是简单基本类型；当然，也有用户定义类的实例。一些对象(比如，数组，衍生类的实例，无静态数据成员的类的实现)拥有子对象，但是其他对象就没有。

无论对象是怎么样的一个类型，一个对象都会存储在一个或多个内存位置上。每一个内存位置不是一个标量类型的对象，就是一个标量类型的子对象，比如，unsigned short、my_class*或序列中的相邻位域。当你使用位域，就需要注意：虽然相邻位域中是不同的对象，但仍视其为相同的内存位置。如图5.1所示，将一个struct分解为多个对象，并且展示了每个对象的内存位置。

![](https://raw.githubusercontent.com/xiaoweiChen/Cpp_Concurrency_In_Action/master/images/chapter5/5-1.png)

图5.1 分解一个struct，展示不同对象的内存位置

首先，完整的struct是一个有多个子对象(每一个成员变量)组成的对象。位域bf1和bf2共享同一个内存位置(int是4字节、32位类型)，并且`std::string`类型的对象s由内部多个内存位置组成，但是其他的每个成员都拥有自己的内存位置。注意，位域宽度为0的bf3是如何与bf4分离，并拥有各自的内存位置的。(译者注：图中bf3是一个错误展示，在C++和C中规定，宽度为0的一个未命名位域强制下一位域对齐到其下一type边界，其中type是该成员的类型。这里使用命名变量为0的位域，可能只是想展示其与bf4是如何分离的。有关位域的更多可以参考[MSDN](https://msdn.microsoft.com/zh-cn/library/ewwyfdbe.aspx)、[wiki](http://en.wikipedia.org/wiki/Bit_field)的页面)。

这里有四个需要牢记的原则：<br>

1. 每一个变量都是一个对象，包括作为其成员变量的对象。<br>
2. 每个对象至少占有一个内存位置。<br>
3. 基本类型都有确定的内存位置(无论类型大小如何，即使他们是相邻的，或是数组的一部分)。<br>
4. 相邻位域是相同内存中的一部分。<br>

我确定你会好奇，这些在并发中有什么作用，那么下面就让我们来见识一下。

### 5.1.2 对象、内存位置和并发

这部分对于C++的多线程应用来说是至关重要的：所有东西都在内存中。当两个线程访问不同(*separate*)的内存位置时，不会存在任何问题，一切都工作顺利。而另一种情况下，当两个线程访问同一(*same*)个内存位置，你就要小心了。如果没有线程更新内存位置上的数据，那还好；只读数据不需要保护或同步。当有线程对内存位置上的数据进行修改，那就有可能会产生条件竞争，就如第3章所述的那样。

为了避免条件竞争，两个线程就需要一定的执行顺序。第一种方式，如第3章所述那样，使用互斥量来确定访问的顺序；当同一互斥量在两个线程同时访问前被锁住，那么在同一时间内就只有一个线程能够访问到对应的内存位置，所以后一个访问必须在前一个访问之后。另一种方式是使用原子操作(*atmic operations*)同步机制(详见5.2节中对于原子操作的定义)，决定两个线程的访问顺序。使用原子操作来规定顺序在5.3节中会有介绍。当多于两个线程访问同一个内存地址时，对每个访问这都需要定义一个顺序。

如果不去规定两个不同线程对同一内存地址访问的顺序，那么访问就不是原子的；并且，当两个线程都是“作者”时，就会产生数据竞争和未定义行为。

以下的声明由为重要：未定义的行为是C++中最黑暗的角落。根据语言的标准，一旦应用中有任何未定义的行为，就很难预料会发生什么事情；因为，未定义行为是难以预料的。我就知道一个未定义行为的特定实例，让某人的显示器起火的案例。虽然，这种事情应该不会发生在你身上，但是数据竞争绝对是一个严重的错误，并且需要不惜一切代价避免它。

另一个重点是：当程序中的对同一内存地址中的数据访问存在竞争，你可以使用原子操作来避免未定义行为。当然，这不会影响竞争的产生——原子操作并没有指定访问顺序——但原子操作把程序拉回了定义行为的区域内。

在我们了解原子操作前，还有一个有关对象和内存地址的概念需要重点了解：修改顺序。

### 5.1.3 修改顺序

每一个在C++程序中的对象，都有(由程序中的所有线程对象)确定好的修改顺序(*modification order*)，在的初始化开始阶段确定。在大多数情况下，这个顺序不同于执行中的顺序，但是在给定的执行程序中，所有线程都需要遵守这顺序。如果对象不是一个原子类型(将在5.2节详述)，你必要确保有足够的同步操作，来确定每个线程都遵守了变量的修改顺序。当不同线程在不同序列中访问同一个值时，你可能就会遇到数据竞争或未定义行为(详见5.1.2节)。如果你使用原子操作，编译器就有责任去替你做必要的同步。

这一要求意味着：投机执行是不允许的，因为当线程按修改顺序访问一个特殊的输入，之后的读操作，必须由线程返回较新的值，并且之后的写操作必须发生在修改顺序之后。同样的，在同一线程上允许读取对象的操作，要不返回一个已写入的值，要不在对象的修改顺序后(也就是在读取后)再写入另一个值。虽然，所有线程都需要遵守程序中每个独立对象的修改顺序，但它们没有必要遵守在独立对象上的相对操作顺序。在5.3.3节中会有更多关于不同线程间操作顺序的内容。

所以，什么是原子操作？它如何来规定顺序？接下来的一节中，会为你揭晓答案。

## 5.2 C++中的原子操作和原子类型

原子操作是一类不可分割的操作，当这样操作在任意线程中进行一半的时候，你是不能查看的；它的状态要不就是完成，要不就是未完成。如果从对象中读取一个值的操作是原子的，并且对对象的所有修改也都是原子的话，那么加载操作要不就会检索对象初始化的值，要不就将值存在某一次修改中。

另一方面，非原子操作可能会被视为由一个线程完成一半的操作。如果这种是一个存储操作，那么其他线程看到的，可能既不是存储前的值，也可能不是已存储的值。如果非原子操作是一个加载操作，那么它可能会去检索对象的部分成员，或是在另一个线程修改了对象的值后，对对象进行检索；所以，检索出来的值可能既不是第一个值，也不是第二个值，可能是某种两者结合的值。这就是一个简单的条件竞争(如第3章所描述)，但是这种级别的竞争会构成数据竞争(详见5.1节)，且会伴有有未定义行为。

在C++中(大多数情况下)你需要一个原子类型去执行一个原子操作，所以我们来看一下原子类型。

### 5.2.1 标准原子类型

标准原子类型(*atomic types*)可以在<atomic>头文件中找到。所有在这种类型上的操作都是原子的，虽然可以使用互斥量去达到原子操作的效果，但只有在这些类型上的操作是原子的(语言明确定义)。实际上，标准原子类型都很相似：它们(大多数)都有一个is_lock_free()成员函数，这个函数允许用户决定是否直接对一个给定类型使用原子指令(x.is_lock_free()返回true)，或对编译器和运行库使用内部锁(x.is_lock_free()返回false)。

只用`std::atomic_flag`类型不提供is_lock_free()成员函数。这个类型是一个简单的布尔标志，并且在这种类型上的操作都需要是无锁的(*lock-free*)；当你有一个简单无锁的布尔标志是，你可以使用其实现一个简单的锁，并且实现其他基础的原子类型。当你觉得“真的很简单”时，就说明：在`std::atomic_flag`对象明确初始化后，做查询和设置(使用test_and_set()成员函数)，或清除(使用clear()成员函数)都很容易。这就是：无赋值，无拷贝，没有测试和清除，没有其他任何操作。

剩下的原子类型都可以通过特化`std::atomic<>`类型模板而访问到，并且拥有更多的功能，但可能不都是无锁的(如之前解释的那样)。在最流行的平台上，期望原子变量都是无锁的内置类型(例如`std::atomic<int>`和`std::atomic<void*>`)，但这没有必要。你在后面将会看到，每个特化接口所反映出的类型特点；位操作(如&=)就没有为普通指针所定义，所以它也就不能为原子指针所定义。

除了直接使用`std::atomic<>`类型模板外，你可以使用在表5.1中所示的原子类型集。由于历史原因，原子类型已经添加入C++标准中，这些备选类型名可能参考相应的`std::atomic<>`特化类型，或是特化的基类。在同一程序中混合使用备选名与`std::atomic<>`特化类名，会使代码的移植大打折扣。

表5.1 标准原子类型的备选名和与其相关的`std::atomic<>`特化类

| 原子类型 | 相关特化类 |
| ------------ | -------------- |
| atomic_bool | std::atomic&lt;bool> |
| atomic_char | std::atomic&lt;char> |
| atomic_schar | std::atomic&lt;signed char> |
| atomic_uchar | std::atomic&lt;unsigned char> |
| atomic_int | std::atomic&lt;int> |
| atomic_uint | std::atomic&lt;unsigned> |
| atomic_short | std::atomic&lt;short> |
| atomic_ushort | std::atomic&lt;unsigned short> |
| atomic_long | std::atomic&lt;long> |
| atomic_ulong | std::atomic&lt;unsigned long> |
| atomic_llong | std::atomic&lt;long long> |
| atomic_ullong | std::atomic&lt;unsigned long long> |
| atomic_char16_t | std::atomic&lt;char16_t> |
| atomic_char32_t | std::atomic&lt;char32_t> |
| atomic_wchar_t | std::atomic&lt;wchar_t> |

C++标准库不仅提供基本原子类型，还定义了与原子类型对应的非原子类型，就如同标准库中的`std::size_t`。如表5.2所示这些类型:

表5.2 标准原子类型定义(typedefs)和对应的内置类型定义(typedefs)

| 原子类型定义 | 标准库中相关类型定义 |
| ------------ | -------------- |
| atomic_int_least8_t | int_least8_t |
| atomic_uint_least8_t | uint_least8_t |
| atomic_int_least16_t | int_least16_t |
| atomic_uint_least16_t | uint_least16_t |
| atomic_int_least32_t | int_least32_t |
| atomic_uint_least32_t | uint_least32_t |
| atomic_int_least64_t | int_least64_t |
| atomic_uint_least64_t | uint_least64_t |
| atomic_int_fast8_t | int_fast8_t |
| atomic_uint_fast8_t | uint_fast8_t |
| atomic_int_fast16_t | int_fast16_t |
| atomic_uint_fast16_t | uint_fast16_t |
| atomic_int_fast32_t | int_fast32_t |
| atomic_uint_fast32_t | uint_fast32_t |
| atomic_int_fast64_t | int_fast64_t |
| atomic_uint_fast64_t | uint_fast64_t |
| atomic_intptr_t | intptr_t |
| atomic_uintptr_t | uintptr_t |
| atomic_size_t | size_t |
| atomic_ptrdiff_t | ptrdiff_t |
| atomic_intmax_t | intmax_t |
| atomic_uintmax_t | uintmax_t |

好多种类型！不过，它们有一个相当简单的模式；对于标准类型进行typedef T，相关的原子类型就在原来的类型名前加上atomic_的前缀：atomic_T。除了singed类型的缩写是s，unsigned的缩写是u，和long long的缩写是llong之外，这种方式也同样适用于内置类型。对于`std::atomic<T>`模板，使用对应的T类型去特化模板的方式，要好于使用别名的方式。

通常，标准原子类型是不能拷贝和赋值，他们没有拷贝构造函数和拷贝赋值操作。但是，因为可以隐式转化成对应的内置类型，所以这些类型依旧支持赋值，可以使用load()和store()成员函数，exchange()、compare_exchange_weak()和compare_exchange_strong()。它们都支持复合赋值符：+=, -=, *=, |= 等等。并且使用整型和指针的特化类型还支持 ++ 和 --。当然，这些操作也有功能相同的成员函数所对应：fetch_add(), fetch_or() 等等。返回值通过赋值操作返回，并且成员函数不是对值进行存储(在有赋值符操作的情况下)，就是对值进行操作(在命名函数中)。这就能避免赋值操作符返回引用。为了获取存储在引用的的值，代码需要执行单独的读操作，从而允许另一个线程在赋值和读取进行的同时修改这个值，这也就为条件竞争打开了大门。

`std::atomic<>`类模板不仅仅一套特化的类型，其作为一个原发模板也可以使用用户定义类型创建对应的原子变量。因为，它是一个通用类模板，很多成员函数的操作在这种情况下有所限制：load(),store()(赋值和转换为用户类型), exchange(), compare_exchange_weak()和compare_exchange_strong()。

每种函数类型的操作都有一个可选内存排序参数，这个参数可以用来指定所需存储的顺序。在5.3节中，会对存储顺序选项进行详述。现在，只需要知道操作分为三类：

1. Store操作，可选如下顺序：memory_order_relaxed, memory_order_release, memory_order_seq_cst。<br>
2. Load操作，可选如下顺序：memory_order_relaxed, memory_order_consume, memory_order_acquire, memory_order_seq_cst。<br>
3. Read-modify-write(读-改-写)操作，可选如下顺序：memory_order_relaxed, memory_order_consume, memory_order_acquire, memory_order_release, memory_order_acq_rel, memory_order_seq_cst。<br>
所有操作的默认顺序都是memory_order_seq_cst。

现在，让我们来看一下每个标准原子类型进行的操作，就从`std::atomic_flag`开始吧。

### 5.2.2 std::atomic_flag的相关操作

`std::atomic_flag`是最简单的标准原子类型，它表示了一个布尔标志。这个类型的对象可以在两个状态间切换：设置和清除。它就是那么的简单，只作为一个构建块存在。我从未期待这个类型被使用，除非在十分特别的情况下。正因如此，它将作为讨论其他原子类型的起点，因为它会展示一些原子类型使用的通用策略。

`std::atomic_flag`类型的对象必须被ATOMIC_FLAG_INIT初始化。初始化标志位是“清除”状态。这里没得选择；这个标志总是初始化为“清除”：

```
std::atomic_flag f = ATOMIC_FLAG_INIT;
```

这适用于任何对象的声明，并且可在任意范围内。它是唯一需要以如此特殊的方式初始化的原子类型，但它也是唯一保证无锁的类型。如果`std::atomic_flag`是静态存储的，那么就的保证其是静态初始化的，也就意味着没有初始化顺序问题；在首次使用时，其都需要初始化。

当你的标志对象已初始化，那么你只能做三件事情：销毁，清除或设置(查询之前的值)。这些事情对应的函数分别是：clear()成员函数，和test_and_set()成员函数。clear()和test_and_set()成员函数可以指定好内存顺序。clear()是一个存储操作，所以不能有memory_order_acquire或memory_order_acq_rel语义，但是test_and_set()是一个“读-改-写”操作，所有可以应用于任何内存顺序标签。每一个原子操作，默认的内存顺序都是memory_order_seq_cst。例如：

```
f.clear(std::memory_order_release);  // 1
bool x=f.test_and_set();  // 2
```

这里，调用clear()①明确要求，使用释放语义清除标志，当调用test_and_set()②使用默认内存顺序设置表示，并且检索旧值。

你不能拷贝构造另一个`std::atomic_flag`对象；并且，你不能将一个对象赋予另一个`std::atomic_flag`对象。这并不是`std::atomic_flag`特有的，而是所有原子类型共有的。一个原子类型的所有操作都是原子的，因赋值和拷贝调用了两个对象，这就就破坏了操作的原子性。在这样的情况下，拷贝构造和拷贝赋值都会将第一个对象的值进行读取，然后再写入另外一个。对于两个独立的对象，这里就有两个独立的操作了，合并这两个操作必定是不原子的。因此，操作就不被允许。

有限的特性集使得`std::atomic_flag`非常适合于作自旋互斥锁。初始化标志是“清除”，并且互斥量处于解锁状态。为了锁上互斥量，循环运行test_and_set()直到旧值为false，就意味着这个线程已经被设置为true了。解锁互斥量是一件很简单的事情，将标志清除即可。实现如下面的程序清单所示：

清单5.1 使用`std::atomic_flag`实现自旋互斥锁

```
class spinlock_mutex
{
  std::atomic_flag flag;
public:
  spinlock_mutex():
    flag(ATOMIC_FLAG_INIT)
  {}
  void lock()
  {
    while(flag.test_and_set(std::memory_order_acquire));
  }
  void unlock()
  {
    flag.clear(std::memory_order_release);
  }
};
```

这样的互斥量是最最基本的，但是它已经足够`std::lock_guard<>`使用了(详见第3章)。其本质就是在lock()中等待，所以这里几乎不可能有竞争的存在，并且可以确保互斥。当我们看到内存顺序语义时，你将会看到它们是如何对一个互斥锁保证必要的强制顺序的。这个例子将在5.3.6节中展示。

由于`std::atomic_flag`局限性太强，因为它没有非修改查询操作，它甚至不能像普通的布尔标志那样使用。所以，你最好使用`std::atomic<bool>`，接下来让我们看看应该如何使用它。

### 5.2.3 std::atomic<bool>的相关操作

最基本的原子整型类型就是`std::atomic<bool>`。如你所料，它有着比`std::atomic_flag`更加齐全的布尔标志特性。虽然它依旧不能拷贝构造和拷贝赋值，但是你可以使用一个非原子的bool类型构造它，所以它可以被初始化为true或false，并且你也可以从一个非原子bool变量赋值给`std::atomic<bool>`的实例：

```
std::atomic<bool> b(true);
b=false;
```

另一件需要注意的事情时，非原子bool类型的赋值操作不同于通常的操作(转换成对应类型的引用，再赋给对应的对象)：它返回一个bool值来代替指定对象。这是在原子类型中，另一种常见的模式：赋值操作通过返回值(返回相关的非原子类型)完成，而非返回引用。如果一个原子变量的引用被返回了，任何依赖与这个赋值结果的代码都需要显式加载这个值，潜在的问题是，结果可能会被另外的线程所修改。通过使用返回非原子值进行赋值的方式，你可以避免这些多余的加载过程，并且得到的值就是实际存储的值。

虽然有内存顺序语义指定，但是使用store()去写入(true或false)还是好于`std::atomic_flag`中限制性很强的clear()。同样的，test_and_set()函数也可以被更加通用的exchange()成员函数所替换，exchange()成员函数允许你使用你新选的值替换已存储的值，并且自动的检索原始值。`std::atomic<bool>`也支持对值的普通(不可修改)查找，其会将对象隐式的转换为一个普通的bool值，或显示的调用load()来完成。如你预期，store()是一个存储操作，而load()是一个加载操作。exchange()是一个“读-改-写”操作：

```
std::atomic<bool> b;
bool x=b.load(std::memory_order_acquire);
b.store(true);
x=b.exchange(false, std::memory_order_acq_rel);
```

`std::atomic<bool>`提供的exchange()，不仅仅是一个“读-改-写”的操作；它还介绍了一种新的存储方式：当当前值与预期值一致时，存储新值的操作。

**存储一个新值(或旧值)取决于当前值**

这是一种新型操作，叫做“比较/交换”，它的形式表现为compare_exchange_weak()和compare_exchange_strong()成员函数。“比较/交换”操作是原子类型编程的基石；它比较原子变量的当前值和提供的预期值，当两值相等时，存储预期值。当两值不等，预期值就会被更新为原子变量中的值。“比较/交换”函数值是一个bool变量，当返回true时执行存储操作，当false则更新期望值。

对于compare_exchange_weak()函数，当原始值与预期值一致时，存储也可能会不成功；在这个例子中变量的值不会发生改变，并且compare_exchange_weak()的返回是false。这可能发生在缺少独立“比较-交换”指令的机器上，当处理器不能保证这个操作能够自动的完成——可能是因为线程的操作将指令队列从中间关闭，并且另一个线程安排的指令将会被操作系统所替换(这里线程数多于处理器数量)。这被称为“伪失败”(*spurious failure*)，因为造成这种情况的原因是时间，而不是变量值。

因为compare_exchange_weak()可以“伪失败”，所以这里通常使用一个循环：

```
bool expected=false;
extern atomic<bool> b; // 设置些什么
while(!b.compare_exchange_weak(expected,true) && !expected);
```

在这个例子中，循环中expected的值始终是false，表示compare_exchange_weak()会莫名的失败。

另一方面，如果实际值与期望值不符，compare_exchange_strong()就能保证值返回false。这就能消除对循环的需要，就可以知道是否成功的改变了一个变量，或已让另一个线程完成。

如果你想要改变变量值，且无论初始值是什么(可能是根据当前值更新了的值)，更新后的期望值将会变更有用；经历每次循环的时候，期望值都会重新加载，所以当没有其他线程同时修改期望时，循环中对compare_exchange_weak()或compare_exchange_strong()的调用都会在下一次(第二次)成功。如果值的计算很容易存储，那么使用compare_exchange_weak()能更好的避免一个双重循环的执行，即使compare_exchange_weak()可能会“伪失败”(因此compare_exchange_strong()包含一个循环)。另一方面，如果值计算的存储本身是耗时的，那么当期望值不变时，使用compare_exchange_strong()可以避免对值的重复计算。对于`std::atomic<bool>`这些都不重要——毕竟只可能有两种值——但是对于其他的原子类型就有较大的影响了。

 “比较/交换”函数很少对两个拥有内存顺序的参数进行操作，这就就允许内存顺序语义在成功和失败的例子中有所不同；其可能是对memory_order_acq_rel语义的一次成功调用，而对memory_order_relaxed语义的一次失败的调动。一次失败的“比较/交换”将不会进行存储，所以“比较/交换”操作不能拥有memeory_order_release或memory_order_acq_rel语义。因此，这里不保证提供的这些值能作为失败的顺序。你也不能提供严格的失败内存顺序；当你需要memory_order_acquire或memory_order_seq_cst作为失败语序，你必须要像“指定它们是成功语序”时那样做。

如果你没有指定失败的语序，那就假设和成功的顺序是一样的，除了release部分的顺序：memory_order_release变成memory_order_relaxed，并且memoyr_order_acq_rel变成memory_order_acquire。如果你都不指定，他们默认顺序将为memory_order_seq_cst，这个顺序提供了对成功和失败的全排序。下面对compare_exchange_weak()的两次调用等价于：

```
std::atomic<bool> b;
bool expected;
b.compare_exchange_weak(expected,true,
  memory_order_acq_rel,memory_order_acquire);
b.compare_exchange_weak(expected,true,memory_order_acq_rel);
```

我在5.3节中会详解对于不同内存顺序选择的结果。

`std::atomic<bool>`和`std::atomic_flag`的不同之处在于，`std::atomic<bool>`不是无锁的；为了保证操作的原子性，其实现中需要一个内置的互斥量。当处于特殊情况时，你可以使用is_lock_free()成员函数，去检查`std::atomic<bool>`上的操作是否无锁。这是另一个，除了`std::atomic_flag`之外，所有原子类型都拥有的特征。

第二简单的原子类型就是特化原子指针——`std::atomic<T*>`，接下来就看看它是如何工作的吧。

### 5.2.4 std::atomic<T*>:指针运算

原子指针类型，可以使用内置类型或自定义类型T，通过特化`std::atomic<T*>`进行定义，就如同使用bool类型定义`std::atomic<bool>`类型一样。虽然接口几乎一致，但是它的操作是对于相关的类型的指针，而非bool值本身。就像`std::atomic<bool>`，虽然它既不能拷贝构造，也不能拷贝赋值，但是他可以通过合适的类型指针进行构造和赋值。如同成员函数is_lock_free()一样，`std::atomic<T*>`也有load(), store(), exchange(), compare_exchange_weak()和compare_exchage_strong()成员函数，与`std::atomic<bool>`的语义相同，获取与返回的类型都是T*，而不是bool。

`std::atomic<T*>`为指针运算提供新的操作。基本操作有fetch_add()和fetch_sub()提供，它们在存储地址上做原子加法和减法，为+=, -=, ++和--提供简易的封装。对于内置类型的操作，如你所预期：如果x是`std::atomic<Foo*>`类型的数组的首地址，然后x+=3让其偏移到第四个元素的地址，并且返回一个普通的`Foo*`类型值，这个指针值是指向数组中第四个元素。fetch_add()和fetch_sub()的返回值略有不同(所以x.ftech_add(3)让x指向第四个元素，并且函数返回指向第一个元素的地址)。这种操作也被称为“交换-相加”，并且这是一个原子的“读-改-写”操作，如同exchange()和compare_exchange_weak()/compare_exchange_strong()一样。正像其他操作那样，返回值是一个普通的`T*`值，而非是`std::atomic<T*>`对象的引用，所以调用代码可以基于之前的值进行操作：

```
class Foo{};
Foo some_array[5];
std::atomic<Foo*> p(some_array);
Foo* x=p.fetch_add(2);  // p加2，并返回原始值
assert(x==some_array);
assert(p.load()==&some_array[2]);
x=(p-=1);  // p减1，并返回原始值
assert(x==&some_array[1]);
assert(p.load()==&some_array[1]);
```

函数也允许内存顺序语义作为给定函数的参数：

```
p.fetch_add(3,std::memory_order_release);
```

因为fetch_add()和fetch_sub()都是“读-改-写”操作，它们可以拥有任意的内存顺序标签，以及加入到一个释放序列中。指定的语序不可能是操作符的形式，因为没办法提供必要的信息：这些形式都具有memory_order_seq_cst语义。

剩下的原子类型基本上都差不多：它们都是整型原子类型，并且都拥有同样的接口(除了相关的内置类型不一样)。下面我们就看看这一类类型。

### 5.2.5 标准的原子整型的相关操作

如同普通的操作集合一样(load(), store(), exchange(), compare_exchange_weak(), 和compare_exchange_strong())，在`std::atomic<int>`和`std::atomic<unsigned long long>`也是有一套完整的操作可以供使用：fetch_add(), fetch_sub(), fetch_and(), fetch_or(), fetch_xor()，还有复合赋值方式((+=, -=, &=, |=和^=)，以及++和--(++x, x++, --x和x--)。虽然对于普通的整型来说，这些复合赋值方式还不完全，但也十分接近完整了：只有除法、乘法和移位操作不在其中。因为，整型原子值通常用来作计数器，或者是掩码，所以以上操作的缺失显得不是那么重要；如果需要，额外的操作可以将compare_exchange_weak()放入循环中完成。

对于`std::atomic<T*>`类型紧密相关的两个函数就是fetch_add()和fetch_sub()；函数原子化操作，并且返回旧值，而符合赋值运算会返回新值。前缀加减和后缀加减与普通用法一样：++x对变量进行自加，并且返回新值；而x++对变量自加，返回旧值。正如你预期的那样，在这两个例子中，结果都是相关整型的一个值。

我们已经看过所有基本原子类型；剩下的就是`std::atomic<>`类型模板，而非其特化类型。那么接下来让我们来了解一下`std::atomic<>`类型模板。

### 5.2.6 std::atomic<>主要类的模板

主模板的存在，在除了标准原子类型之外，允许用户使用自定义类型创建一个原子变量。不是任何自定义类型都可以使用`std::atomic<>`的：需要满足一定的标准才行。为了使用`std::atomic<UDT>`(UDT是用户定义类型)，这个类型必须有拷贝赋值运算符。这就意味着这个类型不能有任何虚函数或虚基类，以及必须使用编译器创建的拷贝赋值操作。不仅仅是这些，自定义类型中所有的基类和非静态数据成员也都需要支持拷贝赋值操作。这(基本上)就允许编译器使用memcpy()，或赋值操作的等价操作，因为它们的实现中没有用户代码。

最后，这个类型必须是“位可比的”(*bitwise equality comparable*)。这与对赋值的要求差不多；你不仅需要确定，一个UDT类型对象可以使用memcpy()进行拷贝，还要确定其对象可以使用memcmp()对位进行比较。之所以要求这么多，是为了保证“比较/交换”操作能正常的工作。

以上严格的限制都是依据第3章中的一个建议：不要将锁定区域内的数据，以引用或指针的形式，作为参数传递给用户提供的函数。通常情况下，编译器不会为`std::atomic<UDT>`类型生成无锁代码，所以它将对所有操作使用一个内部锁。如果用户提供的拷贝赋值或比较操作被允许，那么这就需要传递保护数据的引用作为一个参数，这就有悖于指导意见了。当原子操作需要时，运行库也可自由的使用单锁，并且运行库允许用户提供函数持有锁，这样就有可能产生死锁(或因为做一个比较操作，而组设了其他的线程)。最终，因为这些限制可以让编译器将用户定义的类型看作为一组原始字节，所以编译器可以对`std::atomic<UDT>`直接使用原子指令(因此实例化一个特殊无锁结构)。

注意，虽然使用`std::atomic<float>`或`std::atomic<double>`（内置浮点类型满足使用memcpy和memcmp的标准），但是它们在compare_exchange_strong函数中的表现可能会令人惊讶。当存储的值与当前值相等时，这个操作也可能失败，可能因为旧值是一个不同的表达式。这就不是对浮点数的原子计算操作了。在使用compare_exchange_strong函数的过程中，你可能会遇到相同的结果，如果你使用`std::atomic<>`特化一个用户自定义类型，且这个类型定义了比较操作，而这个比较操作与memcmp又有不同——操作可能会失败，因为两个相等的值用有不同的表达式。

如果你的UDT类型的大小如同(或小于)一个int或`void*`类型时，大多数平台将会对`std::atomic<UDT>`使用原子指令。有些平台可能会对用户自定义类型(两倍于int或`void*`的大小)特化的`std::atmic<>`使用原子指令。这些平台通常支持所谓的“双字节比较和交换”([double-word-compare-and-swap](http://en.wikipedia.org/wiki/Double_compare-and-swap)，*DWCAS*)指令，这个指令与compare_exchange_xxx相关联着。这种指令的支持，对于写无锁代码是有很大的帮助，具体的内容会在第7章讨论。

以上的限制也意味着有些事情你不能做，比如，创建一个`std::atomic<std::vector<int>>`类型。这里不能使用包含有计数器，标志指针和简单数组的类型，作为特化类型。虽然这不会导致任何问题，但是，越是复杂的数据结构，就有越多的操作要去做，而非只有赋值和比较。如果这种情况发生了，你最好使用`std::mutex`保证数据能被必要的操作所保护，就像第3章描述的。

当使用用户定义类型T进行实例化时，`std::atomic<T>`的可用接口就只有: load(), store(), exchange(), compare_exchange_weak(), compare_exchange_strong()和赋值操作，以及向类型T转换的操作。表5.3列举了每一个原子类型所能使用的操作。

![](https://raw.githubusercontent.com/xiaoweiChen/Cpp_Concurrency_In_Action/master/images/chapter5/5-3-table.png)

表5.3 每一个原子类型所能使用的操作

### 5.2.7 原子操作的释放函数

直到现在，我都还没有去描述成员函数对原子类型操作的形式。但是，在不同的原子类型中也有等价的非成员函数存在。大多数非成员函数的命名与对应成员函数有关，但是需要“atomic_”作为前缀(比如，`std::atomic_load()`)。这些函数都会被不同的原子类型所重载。在指定一个内存序列标签时，他们会分成两种：一种没有标签，另一种将“_explicit”作为后缀，并且需要一个额外的参数，或将内存顺序作为标签，亦或只有标签(例如，`std::atomic_store(&atomic_var,new_value)`与`std::atomic_store_explicit(&atomic_var,new_value,std::memory_order_release`)。不过，原子对象被成员函数隐式引用，所有释放函数都持有一个指向原子对象的指针(作为第一个参数)。

例如，`std::atomic_is_lock_free()`只有一种类型(虽然会被其他类型所重载)，并且对于同一个对象a，`std::atomic_is_lock_free(&a)`返回值与a.is_lock_free()相同。同样的，`std::atomic_load(&a)`和a.load()的作用一样，但需要注意的是，与a.load(std::memory_order_acquire)等价的操作是`std::atomic_load_explicit(&a, std::memory_order_acquire)`。

释放函数的设计是为了要与C语言兼容，在C中只能使用指针，而不能使用引用。例如，compare_exchange_weak()和compare_exchange_strong()成员函数的第一个参数(期望值)是一个引用，而`std::atomic_compare_exchange_weak()`(第一个参数是指向对象的指针)的第二个参数是一个指针。`std::atomic_compare_exchange_weak_explicit()`也需要指定成功和失败的内存序列，而“比较/交换”成员函数都有一个单内存序列形式(默认是`std::memory_order_seq_cst`)，重载函数可以分别获取成功和失败内存序列。

对`std::atomic_flag`的操作是“反潮流”的，在那些操作中它们“标志”的名称为：`std::atomic_flag_test_and_set()`和`std::atomic_flag_clear()`，但是以“_explicit”为后缀的额外操作也能够指定内存顺序：`std::atomic_flag_test_and_set_explicit()`和`std::atomic_flag_clear_explicit()`。

C++标准库也对在一个原子类型中的`std::shared_ptr<>`智能指针类型提供释放函数。这打破了“只有原子类型，才能提供原子操作”的原则，这里`std::shared_ptr<>`肯定不是原子类型。但是，C++标准委员会感觉对此提供额外的函数是很重要的。可使用的原子操作有：load, store, exchange和compare/exchange，这些操作重载了标准原子类型的操作，并且获取一个`std::shared_ptr<>*`作为第一个参数：

```
std::shared_ptr<my_data> p;
void process_global_data()
{
  std::shared_ptr<my_data> local=std::atomic_load(&p);
  process_data(local);
}
void update_global_data()
{
  std::shared_ptr<my_data> local(new my_data);
  std::atomic_store(&p,local);
}
```

作为和原子操作一同使用的其他类型，也提供“_explicit”变量，允许你指定所需的内存顺序，并且`std::atomic_is_lock_free()`函数可以用来确定实现是否使用锁，来保证原子性。

如之前的描述，标准原子类型不仅仅是为了避免数据竞争所造成的未定义操作，它们还允许用户对不同线程上的操作进行强制排序。这种强制排序是数据保护和同步操作的基础，例如，`std::mutex`和`std::future<>`。所以，让我继续了解本章的真实意义：内存模型在并发方面的细节，如何使用原子操作同步数据和强制排序。

## 5.3 同步操作和强制排序

假设你有两个线程，一个向数据结构中填充数据，另一个读取数据结构中的数据。为了避免恶性条件竞争，第一个线程设置一个标志，用来表明数据已经准备就绪，并且第二个线程在这个标志设置前不能读取数据。下面的程序清单就是这样的情况。

清单5.2 不同线程对数据的读写

```
#include <vector>
#include <atomic>
#include <iostream>

std::vector<int> data;
std::atomic<bool> data_ready(false);

void reader_thread()
{
  while(!data_ready.load())  // 1
  {
    std::this_thread::sleep(std::milliseconds(1));
  }
  std::cout<<"The answer="<<data[0]<<"\m";  // 2
}
void writer_thread()
{
  data.push_back(42);  // 3
  data_ready=true;  // 4
}
```

先把等待数据的低效循环①放在一边（你需要这个循环，否则想要在线程间共享数据就是不切实际的：数据的每一项都必须是原子的）。你已经知道，当非原子读②和写③对同一数据结构进行无序访问时，将会导致未定义行为的发生，因此这个循环就是确保访问循序被严格的遵守的。

强制访问顺序是由对`std::atomic<bool>`类型的data_ready变量进行操作完成的；这些操作通过“[先行发生](http://en.wikipedia.org/wiki/Happened-before)”(*happens-before*)和“同步发生”(*synchronizes-with*)确定必要的顺序。写入数据③的操作，在写入data_ready标志④的操作前发生，并且读取标志①发生在读取数据②之前。当data_ready①为true，写操作就会与读操作同步，建立一个“先行发生”关系。因为“先行发生”是可传递的，所以读取数据③先行与写入标志④，这两个行为有先行与读取标志的操作①，之前的操作都先行与读取数据②，这样你就拥有了强制顺序：写入数据先行与读取数据，其他没问题了。图5.2展示了先行发生在两线程间的重要性。我向读者线程的while循环中添加了一对迭代。

![](https://raw.githubusercontent.com/xiaoweiChen/Cpp_Concurrency_In_Action/master/images/chapter5/5-2.png)

图5.2 对非原子操作，使用原子操作对操作进行强制排序

所有事情看起来非常直观：对一个值来说，写操作必然先于读操作！在默认它们都是原子操作的时候，这无疑是正确的(这就是原子操作为默认属性的原因)，不过这里需要详细说明：原子操作对于排序要求，也有其他的选项，会在稍后进行详述。

现在，你已经了解了“先行发生”和“同步发生”操作，也是时候看看他们真正的意义了。我将从“同步发生”开始说起。

### 5.3.1 同步发生

“同步发生”关系是指：只能在原子类型之间进行的操作。例如对一个数据结构进行操作(对互斥量上锁)，如果数据结构包含有原子类型，并且操作内部执行了一定的原子操作，那么这些操作就是同步发生关系。从根本上说，这种关系只能来源于对原子类型的操作。

“同步发生”的基本想法是：在变量x进行适当标记的原子写操作W，同步与对x进行适当标记的原子读操作，读取的是W操作写入的内容；或是在W之后，同一线程上的原子写操作对x写入的值；亦或是任意线程对x的一系列原子读-改-写操作(例如，fetch_add()或compare_exchange_weak())。这里，第一个线程读取到的值是W操作写入的(详见5.3.4节)。

先将“适当的标记”放在一边，因为所有对原子类型的操作，默认都是适当标记的。这实际上就是：如果线程A存储了一个值，并且线程B读取了这个值，线程A的存储操作与线程B的载入操作就是同步发生的关系，如同清单5.2所示的那样。

我确信你假设过，所有细微的差别都在“适当的标记”中。C++内存模型允许为原子类型提供各种约束顺序，并且这个标记我们已经提过了。内存排序的各种选项和它们如何与同步发生的关系，将会在5.3.3节中讨论。

让我们先退一步，再来看一下“先行发生”关系。

### 5.3.2 先行发生

“先行发生”关系是一个程序中，基本构建块的操作顺序；它指定了某个操作去影响另一个操作。对于单线程来说，就简单了：当一个操作排在另一个之后，那么这个操作就是先行执行的。这意味着，如果源码中操作A发生在操作B之前，那么A就先行与B发生。你可以回看清单5.2：对data的写入③先于对data_ready④的写入。如果操作在同时发生，因为操作间无序执行，通常情况下，它们就没有先行关系了。这就是另一种排序未被指定的情况。下面的程序会输出“1，2”或“2，1”，因为两个get_num()的执行顺序未被指定。

清单5.3 对于参数中的函数调用顺序是未指定顺序的

```
#include <iostream>
void foo(int a,int b)
{
  std::cout<<a<<”,”<<b<<std::endl;
}
int get_num()
{
  static int i=0;
  return ++i;
}
int main()
{
  foo(get_num(),get_num());  // 无序调用get_num()
}
```

这种情况下，操作在单一声明中是可测序的，例如，逗号操作符的使用，或一个表达式的结果作为一个参数传给另一个表达式。但在通常情况下，操作在单一声明中是不可测序的，所以对其无法先行安排顺序(也就没有先行发生了)。当然，所有操作在一个声明中先行与在下一个声明中的操作。

这只是对之前单线程排序规则的重述，放在这里有什么新意吗？有新意的是线程间的互相作用：如果操作A在线程上，并且线程先行与另一线程上的操作B，那么A就先行于B。这也没什么：你只是添加了一个新关系(线程间的先行)，但当你正在编写多线程程序时，是就这是一个至关重要的关系了。

从基本层面上讲，线程间的先行比较简单，并且依赖与同步关系(详见5.3.1节):如果操作A在一个线程上，与另一个线程上的操作B同步，那么A就线程间先行与B。这同样是一个传递关系：如果A线程间先行与B，并且B线程间先行与C，那么A就线程间先行与C。你可以回看一下清单5.2。

线程间先行可以与排序先行关系相结合：如果操作A排序先行于操作B，并且操作B线程间先行与操作C，那么A线程间先行于C。同样的，如果A同步于B，并且B排序先于C，那么A线程间先行于C。两者的结合，意味着当你对数据进行一系列修改(单线程)时，为线程后续执行C，只需要对可见数据进行一次同步。

这些是线程间强制排序操作的关键规则，也是让清单5.2正常运行的因素。并在数据依赖上有一些细微的差别，你马上就会看到。为了让你理解这些差别，需要讲述一下原子操作使用的内存排序标签，以及这些标签和同步发生之间的联系。

### 5.3.3 原子操作的内存顺序

这里有六个内存序列选项可应用于对原子类型的操作：memory_order_relaxed, memory_order_consume, memory_order_acquire, memory_order_release, memory_order_acq_rel, 以及memory_order_seq_cst。除非你为特定的操作指定一个序列选项，要不内存序列选项对于所有原子类型默认都是memory_order_seq_cst。虽然有六个选项，但是它们仅代表三种内存模型：排序一致序列(*sequentially consistent*)，获取-释放序列(*memory_order_consume, memory_order_acquire, memory_order_release和memory_order_acq_rel*)，和自由序列(*memory_order_relaxed*)。

这些不同的内存序列模型，在不同的CPU架构下，功耗是不一样的。例如，基于处理器架构的可视化精细操作的系统，比起其他系统，添加的同步指令可被排序一致序列使用(在获取-释放序列和自由序列之前)，或被获取-释放序列调用(在自由序列之前)。如果这些系统有多个处理器，这些额外添加的同步指令可能会消耗大量的时间，从而降低系统整体的性能。另一方面，CPU使用的是x86或x86-64架构(例如，使用Intel或AMD处理器的台式电脑)，使用这种架构的CPU不需要任何对获取-释放序列添加额外的指令(没有保证原子性的必要了)，并且，即使是排序一致序列，对于加载操作也不需要任何特殊的处理，不过在进行存储时，有点额外的消耗。

不同种类的内存序列模型，允许专家利用其提升与更细粒度排序相关操作的性能。当默认使用排序一致序列(相较于其他序列，它是最简单的)时，对于在那些不大重要的情况下是有利的。

选择使用哪个模型，或为了了解与序列相关的代码，为什么选择不同的内存模型，是需要了解一个重要的前提，那就是不同模型是如何影响程序的行为。让我们来看一下选择每个操作序列和同步相关的结果。

**排序一致队列**

默认序列命名为“排序一致”(*sequentially cons*)，是因为它意味着，程序中的行为从任意角度去看，序列顺序都保持一致。如果原子类型实例上的所有操作都是序列一致的，那么一个多线程程序的行为，就以某种特殊的排序执行，好像单线程那样。这是目前来看，最容易理解的内存序列，这也就是将其设置为默认的原因：所有线程都必须了解，不同的操作也遵守相同的顺序。因为其简单的行为，可以使用原子变量进行编写。通过不同的线程，你可以写出所有序列上可能的操作，这样就可以消除那些不一致，以及验证你代码的行为是否与预期相符。这也就意味着，所有操作都不能重排序；如果你的代码，在一个线程中，将一个操作放在另一个操作前面，那么这个顺序就必须让其他所有的线程所了解。

从同步的角度看，对于同一变量，排序一致的存储操作同步相关于同步一致的载入操作。这就提供了一种对两个(以上)线程操作的排序约束，但是排序一致的功能要比排序约束大的多。所以，对于使用排序一致原子操作的系统上的任一排序一致的原子操作，都会在对值进行存储以后，再进行加载。清单5.4就是这种一致性约束的演示。这种约束不是线程在自由内存序列中使用原子操作；这些线程依旧可以知道，操作以不同顺序排列，所以你必须使用排序一致操作，去保证在多线的情况下有加速的效果。

不过，简单是要付出代价的。在一个多核若排序的机器上，它会加强对性能的惩罚，因为整个序列中的操作都必须在多个处理器上保持一致，可能需要对处理器间的同步操作进行扩展(代价很昂贵！)。即便如此，一些处理器架构(比如通用x86和x86-64架构)就提供了相对廉价的序列一致，所以你需要考虑使用序列一致对性能的影响，这就需要你去查阅你目标处理器的架构文档，进行更多的了解。

以下清单展示了序列一致的行为，对于x和y的加载和存储都显示标注为memory_order_seq_cst，不过在这段代码中，标签可能会忽略，因为其是默认项。

清单5.4 全序——序列一致

```
#include <atomic>
#include <thread>
#include <assert.h>

std::atomic<bool> x,y;
std::atomic<int> z;

void write_x()
{
  x.store(true,std::memory_order_seq_cst);  // 1
}

void write_y()
{
  y.store(true,std::memory_order_seq_cst);  // 2
}
void read_x_then_y()
{
  while(!x.load(std::memory_order_seq_cst));
  if(y.load(std::memory_order_seq_cst))  // 3
    ++z;
}
void read_y_then_x()
{
  while(!y.load(std::memory_order_seq_cst));
  if(x.load(std::memory_order_seq_cst))  // 4
    ++z;
}
int main()
{
  x=false;
  y=false;
  z=0;
  std::thread a(write_x);
  std::thread b(write_y);
  std::thread c(read_x_then_y);
  std::thread d(read_y_then_x);
  a.join();
  b.join();
  c.join();
  d.join();
  assert(z.load()!=0);  // 5
}
```

assert⑤语句是永远不会触发的，因为不是存储x的操作①发生，就是存储y的操作②发生。如果在read_x_then_y中加载y③返回false，那是因为存储x的操作肯定发生在存储y的操作之前，那么在这种情况下在read_y_then_x中加载x④必定会返回true，因为while循环能保证在某一时刻y是true。因为memory_order_seq_cst的语义需要一个单全序将所有操作都标记为memory_order_seq_cst，这就暗示着“加载y并返回false③”与“存储y①”的操作，有一个确定的顺序。只有一个全序时，如果一个线程看到x==true，随后又看到y==false，这就意味着在总序列中存储x的操作发生在存储y的操作之前。

当然，因为所有事情都是对称的，所以就有可能以其他方式发生，比如，加载x④的操作返回false，或强制加载y③的操作返回true。在这两种情况下，z都等于1。当两个加载操作都返回true，z就等于2，所以任何情况下，z都不能是0。

当read_x_then_y知道x为true，并且y为false，那么这些操作就有“先发执行”关系了，如图5.3所示。

![](https://raw.githubusercontent.com/xiaoweiChen/Cpp_Concurrency_In_Action/master/images/chapter5/5-3.png)

图5.3 序列一致与先发执行

虚线始于read_x_then_y中对y的加载操作，到达write_y中对y的存储，其暗示了排序关系需要保持序列一致：在操作的全局操作顺序memory_order_seq_cst中，加载操作必须在存储操作之前发生，就产生了图中的结果。

序列一致是最简单、直观的序列，但是他也是最昂贵的内存序列，因为它需要对所有线程进行全局同步。在一个多处理系统上，这就需要处理期间进行大量并且费时的信息交换。

为了避免这种同步消耗，你需要走出序列一致的世界，并且考虑使用其他内存序列。

**非排序一致内存模型**

当你踏出序列一致的世界，所有事情就开始变的复杂。可能最需要处理的问题就是：再也不会有全局的序列了(*there’s no longer a single global order of events*)。这就意味着不同线程看到相同操作，不一定有着相同的顺序，还有对于不同线程的操作，都会整齐的，一个接着另一个执行的想法是需要摒弃的。不仅是你有没有考虑事情真的同时发生的问题，还有线程没必要去保证一致性(*threads don’t have to agree on the order of events*)。为了写出(或仅是了解)任何一段使用非默认内存序列的代码，要想做这件事情，那么之前的那句话就是至关重要的。你要知道，这不仅仅是编译器可以重新排列指令的问题。即使线程运行相同的代码，它们都能拒绝遵循事件发生的顺序，因为操作在其他线程上没有明确的顺序限制；因为不同的CPU缓存和内部缓冲区，在同样的存储空间中可以存储不同的值。这非常重要，这里我再重申一遍：线程没必要去保证一致性。

不仅是要摒弃交错执行操作的想法，你还要放弃使用编译器或处理器重排指令的想法。在没有明确的顺序限制下，唯一的要求就是，所有线程都要统一对每一个独立变量的修改顺序。对不同变量的操作可以体现在不同线程的不同序列上，提供的值要与任意附加顺序限制保持一致。

踏出排序一致世界后，最好的示范就是使用memory_order_relaxed对所有操作进行约束。如果你已经对其有所了解，那么你可以跳到获取-释放序列继续阅读，获取-释放序列允许你选择在操作间引入顺序关系(并且收回你的理智)。

**自由序列**

在原子类型上的操作以自由序列执行，没有任何同步关系。在同一线程中对于同一变量的操作还是服从先发执行的关系，但是这里不同线程几乎不需要相对的顺序。唯一的要求是，在访问同一线程中的单个原子变量不能重排序；当一个给定线程已经看到一个原子变量的特定值，线程随后的读操作就不会去检索变量较早的那个值。当使用memory_order_relaxed，就不需要任何额外的同步，对于每个变量的修改顺序只是线程间共享的事情。

为了演示如何不去限制你的非限制操作，你只需要两个线程，就如同下面代码清单那样。

清单5.5 非限制操作只有非常少的顺序要求

```
#include <atomic>
#include <thread>
#include <assert.h>

std::atomic<bool> x,y;
std::atomic<int> z;

void write_x_then_y()
{
  x.store(true,std::memory_order_relaxed);  // 1
  y.store(true,std::memory_order_relaxed);  // 2
}
void read_y_then_x()
{
  while(!y.load(std::memory_order_relaxed));  // 3
  if(x.load(std::memory_order_relaxed))  // 4
    ++z;
}
int main()
{
  x=false;
  y=false;
  z=0;
  std::thread a(write_x_then_y);
  std::thread b(read_y_then_x);
  a.join();
  b.join();
  assert(z.load()!=0);  // 5
}
```

这次assert⑤可能会触发，因为加载x的操作④可能读取到false，即使加载y的操作③读取到true，并且存储x的操作①先发与存储y的操作②。x和y是两个不同的变量，所以这里没有顺序去保证每个操作产生相关值的可见性。

非限制操作对于不同变量可以自由重排序，只要它们服从任意的先发执行关系即可(比如，在同一线程中)。它们不会引入同步相关的顺序。清单5.5中的先发执行关系如图5.4所示(只是其中一个可能的结果)。尽管，在不同的存储/加载操作间有着先发执行关系，这里不是在一对存储于载入之间了，所以载入操作可以看到“违反”顺序的存储操作。

![](https://raw.githubusercontent.com/xiaoweiChen/Cpp_Concurrency_In_Action/master/images/chapter5/5-4.png)

图5.4 非限制原子操作与先发执行

让我们来看一个略微复杂的例子，其有三个变量和五个线程。

清单5.6 非限制操作——多线程版

```
#include <thread>
#include <atomic>
#include <iostream>

std::atomic<int> x(0),y(0),z(0);  // 1
std::atomic<bool> go(false);  // 2

unsigned const loop_count=10;

struct read_values
{
  int x,y,z;
};

read_values values1[loop_count];
read_values values2[loop_count];
read_values values3[loop_count];
read_values values4[loop_count];
read_values values5[loop_count];

void increment(std::atomic<int>* var_to_inc,read_values* values)
{
  while(!go)
    std::this_thread::yield();  // 3 自旋，等待信号
  for(unsigned i=0;i<loop_count;++i)
  {
    values[i].x=x.load(std::memory_order_relaxed);
    values[i].y=y.load(std::memory_order_relaxed);
    values[i].z=z.load(std::memory_order_relaxed);
    var_to_inc->store(i+1,std::memory_order_relaxed);  // 4
    std::this_thread::yield();
  }
}

void read_vals(read_values* values)
{
  while(!go)
    std::this_thread::yield(); // 5 自旋，等待信号
  for(unsigned i=0;i<loop_count;++i)
  {
    values[i].x=x.load(std::memory_order_relaxed);
    values[i].y=y.load(std::memory_order_relaxed);
    values[i].z=z.load(std::memory_order_relaxed);
    std::this_thread::yield();
  }
}

void print(read_values* v)
{
  for(unsigned i=0;i<loop_count;++i)
  {
    if(i)
      std::cout<<",";
    std::cout<<"("<<v[i].x<<","<<v[i].y<<","<<v[i].z<<")";
  }
  std::cout<<std::endl;
}

int main()
{
  std::thread t1(increment,&x,values1);
  std::thread t2(increment,&y,values2);
  std::thread t3(increment,&z,values3);
  std::thread t4(read_vals,values4);
  std::thread t5(read_vals,values5);

  go=true;  // 6 开始执行主循环的信号

  t5.join();
  t4.join();
  t3.join();
  t2.join();
  t1.join();

  print(values1);  // 7 打印最终结果
  print(values2);
  print(values3);
  print(values4);
  print(values5);
}
```

这段代码本质上很简单。你拥有三个全局原子变量①和五个线程。每一个线程循环10次，使用memory_order_relaxed读取三个原子变量的值，并且将它们存储在一个数组上。其中三个线程每次通过循环④来更新其中一个原子变量，这时剩下的两个线程就只负责读取。当所有线程都“加入”，就能打印出来每个线程存到数组上的值了。

原子变量go②用来确保循环在同时退出。启动线程是昂贵的操作，并且没有明确的延迟，第一个线程可能在最后一个线程开始前结束。每个线程都在等待go变为true前都在进行循环③⑤，并且一旦go设置为true所有线程都会开始运行⑥。

程序一种可能的输出为：

```
(0,0,0),(1,0,0),(2,0,0),(3,0,0),(4,0,0),(5,7,0),(6,7,8),(7,9,8),(8,9,8),(9,9,10)
(0,0,0),(0,1,0),(0,2,0),(1,3,5),(8,4,5),(8,5,5),(8,6,6),(8,7,9),(10,8,9),(10,9,10)
(0,0,0),(0,0,1),(0,0,2),(0,0,3),(0,0,4),(0,0,5),(0,0,6),(0,0,7),(0,0,8),(0,0,9)
(1,3,0),(2,3,0),(2,4,1),(3,6,4),(3,9,5),(5,10,6),(5,10,8),(5,10,10),(9,10,10),(10,10,10)
(0,0,0),(0,0,0),(0,0,0),(6,3,7),(6,5,7),(7,7,7),(7,8,7),(8,8,7),(8,8,9),(8,8,9)
```

前三行中线程都做了更新，后两行线程只是做读取。每三个值都是一组x，y和z，并按照这样的顺序依次循环。对于输出，需要注意的一些事是：

1. 第一组值中x增1，第二组值中y增1，并且第三组中z增1。<br>
2. x元素只在给定集中增加，y和z也一样，但是增加是不均匀的，并且相对顺序在所有线程中都不同。<br>
3. 线程3看不到x或y的任何更新；他能看到的只有z的更新。这并不妨碍别的线程观察z的更新，并同时观察x和y的更新。<br>

对于非限制操作，这个结果是合法的，但是不是唯一合法的输出。任意组值都用三个变量保持一致，值从0到10依次递增，并且线程递增给定变量，所以打印出来的值在0到10的范围内都是合法的。

**了解自由排序**

为了了解自由序列是如何工作的，先将每一个变量想象成一个在独立房间中拿着记事本的人。他的记事本上是一组值的列表。你可以通过打电话的方式让他给你一个值，或让他写下一个新值。如果你告诉他写下一个新值，他会将这个新值写在表的最后。如果你让他给你一个值，他会从列表中读取一个值给你。

在你第一次与这个人交谈时，如果你问他要一个值，他可能会给你现在列表中的任意值。如果之后你再问他要一个值，它可能会再给你同一个值，或将列表后面的值给你，他不会给你列表上端的值。如果你让他写一个值，并且随后再问他要一个值，他要不就给你你刚告诉他的那个值，要不就是一个列表下端的值。

试想当他的笔记本上开始有5，10，23，3，1，2这几个数。如果你问他索要一个值，你可能获取这几个数中的任意一个。如果他给你10，那么下次再问他要值的时候可能会再给你10，或者10后面的数，但绝对不会是5。如果那你问他要了五次，他就可能回答“10，10，1，2，2”。如果你让他写下42，他将会把这个值添加在列表的最后。如果你再问他要值，他可能会告诉你“42”，直到有其他值写在了后面并且他认为他愿意将那个数告诉你。

现在，想象你有个朋友叫Carl，他也有那个计数员的电话。Carl也可以打电话给计算员，让他写下一个值或获取一个值，他对Carl回应的规则和你是一样的。他只有一部电话，所以他一次只能处理一个人的请求，所以他记事本上的列表是一个简单的列表。但是，你让他写下一个新值的时候，不意味着他会将这个消息告诉Carl，反之亦然。如果Carl从他那里获取一个值“23”，之后因为你告诉他写下42，这不意味着下次他会将这件事告诉Carl。他可能会告诉Carl任意一个值，23，3，1，2，42亦或是67(是Fred在你之后告诉他的)。他会很高兴的告诉Carl“23，3，3，1，67”，与你告诉他的值完全不一致。这就像它在使用便签跟踪告诉每个人的数，就像图5.5那样。

![](https://raw.githubusercontent.com/xiaoweiChen/Cpp_Concurrency_In_Action/master/images/chapter5/5-5.png)

图5.5 计数员的笔记

现在，想象一下，不仅仅只有一个人在房间里，而是在一个小农场里，每个人都有一部电话和一个笔记本。这就是我们的原子变量。每一个变量拥有他们自己的修改顺序(笔记上的简单数值列表)，但是每个原子变量之间没有任何关系。如果每一个调用者(你，Carl，Anne，Dave和Fred)是一个线程，那么对每个操作使用memory_order_relaxed你就会得到上面的结果。这里还有些事情你可以告诉在小房子的人，例如，“写下这个值，并且告诉我现在列表中的最后一个值”(exchange)，或“写下这个值，当列表的最后一个值为某值；如果不是，告诉我看我是不是猜对了”(compare_exchange_strong)，但是这都不影响一般性原则。

如果你仔细想想清单5.5的逻辑，那么write_x_then_y就像某人打电话给房子x里的人，并且告诉他写下true，之后打电话给在y房间的另一个人，告诉他写下true。线程反复执行调用read_y_then_x，就像打电话给房间y的人问他要值，直到要到true，然后打电话给房间x的，继续问他要值。在x房间中的人有义务告诉你在他列表中任意指定的值，他也是有权利所false的。

这就让自由的原子操作变得难以处理。他们必须与原子操作结合使用，这些原子操作必须有较强的排序语义，为了让内部线程同步变得更有用。我强烈建议避免自由的原子操作，除非它们是硬性要求的，并且在使用它们的时候需要十二分的谨慎。给出的不直观的结果，就像是清单5.5中使用双线程和双变量的结果一样，不难想象在有更多线程和更多变量时，其会变的更加复杂。

要想获取额外的同步，且不使用全局排序一致，可以使用获取-释放序列(*acquire-release ordering*)。

**获取-释放序列**

这个序列是自由序列(*relaxed ordering*)的加强版；虽然操作依旧没有统一的顺序，但是在这个序列引入了同步。在这种序列模型中，原子加载就是“获取”(*acquire*)操作(memory_order_acquire)，原子存储就是“释放”操作(memory_order_release)，原子读-改-写操作(例如fetch_add()或exchange())在这里，不是“获取”，就是“释放”，或者两者兼有的操作(memory_order_acq_rel)。这里，同步在线程释放和获取间，是成对的(*pairwise*)。释放操作与获取操作同步，这样就能读取已写入的值。这意味着不同线程看到的序列虽还是不同，但这些序列都是受限的。下面列表中是使用获取-释放序列(而非序列一致方式)，对清单5.4的一次重写。

清单5.7 获取-释放不意味着统一操作顺序

```
#include <atomic>
#include <thread>
#include <assert.h>

std::atomic<bool> x,y;
std::atomic<int> z;
void write_x()
{
  x.store(true,std::memory_order_release);
}
void write_y()
{
  y.store(true,std::memory_order_release);
}
void read_x_then_y()
{
  while(!x.load(std::memory_order_acquire));
  if(y.load(std::memory_order_acquire))  // 1
    ++z;
}
void read_y_then_x()
{
  while(!y.load(std::memory_order_acquire));
  if(x.load(std::memory_order_acquire))
    ++z;
}
int main()
{
  x=false;
  y=false;
  z=0;
  std::thread a(write_x);
  std::thread b(write_y);
  std::thread c(read_x_then_y);
  std::thread d(read_y_then_x);
  a.join();
  b.join();
  c.join();
  d.join();
  assert(z.load()!=0); // 3
}
```

在这个例子中断言③可能会触发(就如同自由排序那样)，因为可能在加载x②和y③的时候，读取到的是false。因为x和y是由不同线程写入，所以序列中的每一次释放到获取都不会影响到其他线程的操作。

图5.6展示了清单5.7的先行关系，对于读取的结果，两个(读取)线程看到的是两个完全不同的世界。如前所述，这可能是因为这里没有对先行顺序进行强制规定导致的。

![](https://raw.githubusercontent.com/xiaoweiChen/Cpp_Concurrency_In_Action/master/images/chapter5/5-6.png)

图5.6 获取-释放，以及先行过程

为了了解获取-释放序列有什么优点，你需要考虑将两次存储由一个线程来完成，就像清单5.5那样。当你需要使用memory_order_release改变y中的存储，并且使用memory_order_acquire来加载y中的值，就像下面程序清单所做的那样，而后，就会影响到序列中对x的操作。

清单5.8 获取-释放操作会影响序列中的释放操作

```
#include <atomic>
#include <thread>
#include <assert.h>

std::atomic<bool> x,y;
std::atomic<int> z;

void write_x_then_y()
{
  x.store(true,std::memory_order_relaxed);  // 1 自旋，等待y被设置为true
  y.store(true,std::memory_order_release);  // 2
}
void read_y_then_x()
{
  while(!y.load(std::memory_order_acquire));  // 3
  if(x.load(std::memory_order_relaxed))  // 4
    ++z;
}
int main()
{
  x=false;
  y=false;
  z=0;
  std::thread a(write_x_then_y);
  std::thread b(read_y_then_x);
  a.join();
  b.join();
  assert(z.load()!=0);  // 5
}
```

最后，读取y③时会得到true，和存储时写入的一样②。因为存储使用的是memory_order_release，读取使用的是memory_order_acquire，存储就与读取就同步了。因为这两个操作是由同一个线程完成的，所以存储x①先行与加载y②。对y的存储同步与对y的加载，存储x也就先行于对y的加载，并且扩展先行与x的读取。因此，加载x的值必为true，并且断言⑤不会触发。如果对于y的加载不是在while循环中，那么情况可能就会有所不同；加载y的时候可能会读取到false，在这种情况下对于读取到的x是什么值，就没有要求了。为了保证同步，加载和释放操作必须成对。所以，无论有何影响，释放操作存储的值，必须要让获取操作看到。当存储如②或加载如③，都是一个释放操作时，对x的访问就无序了，也就无法保证④处读到的是true，并且还会触发断言。

你也可以将获取-释放序列与之前提到记录员和他的小隔间相关联，不过你可能需要添加很多东西到这个模型中。首先，试想每个存储操作做一部分更新，那么当你电话给一个人，让他写下一个数字，你也需要告诉他更新哪一部分：“请在423组中写下99”。对于某一组的最后一个值的存储，你也需要告诉那个人：“请写下147，这是最后存储在423组的值”。小隔间中的人会即使写下这一信息，以及告诉他的值。这个就是存储-释放操作的模型。下一次，你告诉另外一个人写下一组值时，你需要改变组号：“请在组424中写入41”

当你询问一个时，你就要做出一个选择：你要不就仅仅询问一个值(这就是一次自由加载，这种情况下，小隔间中的人会给你的)，要不询问一个值以及其关于组的信息(是否是某组中的最后一个，这就是加载-获取模型)。当你询问组信息，且值不是组中的最后一个，隔间中的人会这样告诉你，“这个值是987，它是一个‘普通’值”，但当这个值是最后一个时，他会告诉你：“数字为987，这个值是956组的最后一个，来源于Anne”。现在，获取-释放的语义就很明确了：当你查询一个值，你告诉他你知道到所有组后，她会低头查看他的列表，看你知道的这些数，是不是在对应组的最后，并且告诉你那个值的属性，或继续在列表中查询。

如何理解这个模型中获取-释放的语义？让我们看一下我们的例子。首先，线程a运行write_x_then_y函数，然后告诉在x屋的记录员，“请写下true作为组1的一部分，信息来源于线程a”，之后记录员工整的写下了这些信息。而后，线程a告诉在y屋的记录员，“请写下true作为组1的一部分，信息来源于线程a”。在此期间，线程b运行read_y_then_x。线程b持续向y屋的记录员询问值与组的信息，知道它听到记录员说“true”。记录员可能不得不告诉他很多遍，不过最终记录员还是说了“true”。y屋的记录员不仅仅是说“true”，他还要说“组1最后是由线程a写入”。

现在，线程b会持续询问x屋的记录员，但这次他会说“请给我一个值，我知道这个值是组1的值，并且是由线程a写入的”。所以现在，x屋中的记录员就开始查找组1中由线程a写入的值。这里他注意到，他写入的值是true，同样也是他列表中的最后一个值，所以它必须读出这个值；否则，他讲打破这个游戏的规则。

当你回看5.3.2节中对“线程间先行”的定义，一个很重要的特性就是它的传递：当A线程间先行于B，并且B线程间先行于C，那么A就线程间先行于C。这就意味着，获取-释放序列可以在若干线程间使用同步数据，甚至可以在“中间”线程接触到这些数据前，使用这些数据。

**与同步传递相关的获取-释放序列**

为了考虑传递顺序，你至少需要三个线程。第一个线程用来修改共享变量，并且对其中一个做“存储-释放”处理。然后第二个线程使用“加载-获取”读取由“存储-释放”操作过的变量，并且再对第二个变量进行“存储-释放”操作。最后，由第三个线程通过“加载-获取”读取第二个共享变量。提供“加载-获取”操作，来读取被“存储-释放”操作写入的值，是为了保证同步关系，这里即便是中间线程没有对共享变量做任何操作，第三个线程也可以读取被第一个线程操作过的变量。下面的代码可以用来描述这样的场景。

清单5.9 使用获取和释放顺序进行同步传递

```
std::atomic<int> data[5];
std::atomic<bool> sync1(false),sync2(false);

void thread_1()
{
  data[0].store(42,std::memory_order_relaxed);
  data[1].store(97,std::memory_order_relaxed);
  data[2].store(17,std::memory_order_relaxed);
  data[3].store(-141,std::memory_order_relaxed);
  data[4].store(2003,std::memory_order_relaxed);
  sync1.store(true,std::memory_order_release);  // 1.设置sync1
}

void thread_2()
{
  while(!sync1.load(std::memory_order_acquire));  // 2.直到sync1设置后，循环结束
  sync2.store(true,std::memory_order_release);  // 3.设置sync2
}
void thread_3()
{
  while(!sync2.load(std::memory_order_acquire));   // 4.直到sync1设置后，循环结束
  assert(data[0].load(std::memory_order_relaxed)==42);
  assert(data[1].load(std::memory_order_relaxed)==97);
  assert(data[2].load(std::memory_order_relaxed)==17);
  assert(data[3].load(std::memory_order_relaxed)==-141);
  assert(data[4].load(std::memory_order_relaxed)==2003);
}
```

尽管thread_2只接触到变量syn1②和sync2③，不过这对于thread_1和thread_3的同步就足够了，这就能保证断言不会触发。首先，thread_1将数据存储到data中先行与存储sync1①（它们在同一个线程内）。因为加载sync1①的是一个while循环，它最终会看到thread_1存储的值(是从“释放-获取”对的后半对获取)。因此，对于sync1的存储先行与最终对于sync1的加载(在while循环中)。thread_3的加载操作④，位于存储sync2③操作的前面(也就是先行)。存储sync2③因此先行于thread_3的加载④，加载又先行与存储sync2③，存储sync2又先行与加载sync2④，加载syn2又先行与加载data。因此，thread_1存储数据到data的操作先行于thread_3中对data的加载，并且保证断言都不会触发。

在这个例子中，你可以将sync1和sync2，通过在thread_2中使用“读-改-写”操作(memory_order_acq_rel)，将其合并成一个独立的变量。其中会使用compare_exchange_strong()来保证thread_1对变量只进行一次更新：

```
std::atomic<int> sync(0);
void thread_1()
{
  // ...
  sync.store(1,std::memory_order_release);
}

void thread_2()
{
  int expected=1;
  while(!sync.compare_exchange_strong(expected,2,
              std::memory_order_acq_rel))
    expected=1;
}
void thread_3()
{
  while(sync.load(std::memory_order_acquire)<2);
  // ...
}
```

如果你使用“读-改-写”操作，选择语义就很重要了。在这个例子中，你想要同时进行获取和释放的语义，所以memory_order_acq_rel是一个合适的选择，但你也可以使用其他序列。使用memory_order_acquire语义的fetch_sub是不会和任何东西同步的，即使它存储了一个值，这是因为其没有释放操作。同样的，使用memory_order_release语义的fetch_or也不会和任何存储操作进行同步，因为对于fetch_or的读取，并不是一个获取操作。使用memory_order_acq_rel语义的“读-改-写”操作，每一个动作都包含获取和释放操作，所以可以和之前的存储操作进行同步，并且可以对随后的加载操作进行同步，就像上面例子中那样。

如果你将“获取-释放”操作和“序列一致”操作进行混合，“序列一致”的加载动作，就像使用了获取语义的加载操作；并且序列一致的存储操作，就如使用了释放语义的存储。“序列一致”的读-改-写操作行为，就像同时使用了获取和释放的操作。“自由操作”依旧那么自由，但其会和额外的同步进行绑定（也就是使用“获取-释放”的语义）。

尽管潜在的结果并不那么直观，每个使用锁的同学都不得不去解决同一个序列问题：锁住互斥量是一个获取操作，并且解锁这个互斥量是一个释放操作。随着互斥量的增多，你必须确保同一个互斥量在你读取变量或修改变量的时候是锁住的，并且同样适合于这里；你的获取和释放操作必须在同一个变量上，以保证访问顺序。当数据被一个互斥量所保护时，锁的性质就保证得到的结果是没有区别的，因为锁住与解锁的操作都是序列一致的操作。同样的，当你对原子变量使用获取和释放序列，为的是构建一个简单的锁，那么这里的代码必然要使用锁，即使内部操作不是序列一致的，其外部表现将会是序列一致的。

当你的原子操作不需要严格的序列一致序列，成对同步的“获取-释放”序列可以提供，比全局序列一致性操作，更加低廉的潜在同步。这里还需要对心理代价进行权衡，为了保证序列能够正常的工作，还要保证非直观的跨线程行为是没有问题的。

**获取-释放序列和memory_order_consume的数据相关性**

在介绍本章节的时候，我说过，memory_order_consume是“获取-释放”序列模型的一部分，但是在前面我们没有对其进行过多的讨论。这是因为memory_order_consume很特别：它完全依赖于数据，并且其展示了与线程间先行关系(可见5.3.2节)的不同之处。

这里有两种新关系用来处理数据依赖：前序依赖(*dependency-ordered-before*)和携带依赖(*carries-a-dependency-to*)。就像前列(*sequenced-before*)，携带依赖对于数据依赖的操作，严格应用于一个独立线程和其基本模型；如果A操作结果要使用操作B的操作数，而后A将携带依赖与B。如果A操作的结果是一个标量，比如int，而后的关系仍然适用于，当A的结果存储在一个变量中，并且这个变量需要被其他操作使用。这个操作是也是可以传递的，所以当A携带依赖B，并且B携带依赖C，就额可以得出A携带依赖C的关系。

当其不影响线程间的先行关系时，对于同步来说，这并未带来任何的好处，但是它做到：当A前序依赖B，那么A线程间也前序依赖B。

这种内存序列的一个很重要使用方式，是在原子操作载入指向数据的指针时。当使用memory_order_consume作为加载语义，并且memory_order_release作为之前的存储语义，你要保证指针指向的值是已同步的，并且不需要对其他任何非独立数据施加任何同步要求。下面的代码就展示了这么一个场景。

清单5.10 使用`std::memroy_order_consume`同步数据

```
struct X
{
int i;
std::string s;
};

std::atomic<X*> p;
std::atomic<int> a;

void create_x()
{
  X* x=new X;
  x->i=42;
  x->s="hello";
  a.store(99,std::memory_order_relaxed);  // 1
  p.store(x,std::memory_order_release);  // 2
}

void use_x()
{
  X* x;
  while(!(x=p.load(std::memory_order_consume)))  // 3
    std::this_thread::sleep(std::chrono::microseconds(1));
  assert(x->i==42);  // 4
  assert(x->s=="hello");  // 5
  assert(a.load(std::memory_order_relaxed)==99);  // 6
}

int main()
{
  std::thread t1(create_x);
  std::thread t2(use_x);
  t1.join();
  t2.join();
}
```

尽管，对a的存储①在存储p②之前，并且存储p的操作标记为memory_order_release，加载p的操作标记为memory_order_consume，这就意味着存储p仅先行那些需要加载p的操作。同样，也意味着X结构体中数据成员所在的断言语句④⑤，不会被触发，这是因为对x变量操作的表达式对加载p的操作携带有依赖。另一方面，对于加载变量a的断言就不能确定是否会被触发；这个操作并不依赖于p的加载操作，所以这里没法保证数据已经被读取。当然，这个情况也是很明显的，因为这个操作被标记为memory_order_relaxed。

有时，你不想为携带依赖增加其他的开销。你想让编译器在寄存器中缓存这些值，以及优化重排序操作代码，而不是对这些依赖大惊小怪。这种情况下，你可以使用`std::kill_dependecy()`来显式打破依赖链。`std::kill_dependency()`是一个简单的函数模板，其会复制提供的参数给返回值，但是依旧会打破依赖链。例如，当你拥有一个全局的只读数组，当其他线程对数组索引进行检索时，你使用的是`std::memory_order_consume`，那么你可以使用`std::kill_dependency()`让编译器知道这里不需要重新读取该数组的内容，就像下面的例子一样：

```
int global_data[]={ … };
std::atomic<int> index;

void f()
{
  int i=index.load(std::memory_order_consume);
  do_something_with(global_data[std::kill_dependency(i)]);
}
```

当然，你不需要在如此简单的场景下使用`std::memory_order_consume`，但是你可以在类似情况，且代码较为复杂时，调用`std::kill_dependency()`。你必须记住，这是为了优化，所以这种方式必须谨慎使用，并且需要性能数据证明其存在的意义。

现在，我们已经讨论了所有基本内存序列，是时候看看更加复杂的同步关系了————释放队列。

### 5.3.4 释放队列与同步

回到5.3.1节，我提到过，通过其他线程，即使有(有序的)多个“读-改-写”操作(所有操作都已经做了适当的标记)在存储和加载操作之间，你依旧可以获取原子变量存储与加载的同步关系。现在，我已经讨论所有可能使用到的内存序列“标签”，我在这里可以做一个简单的概述。当存储操作被标记为memory_order_release，memory_order_acq_rel或memory_order_seq_cst，加载被标记为memory_order_consum，memory_order_acquire或memory_order_sqy_cst，并且操作链上的每一加载操作都会读取之前操作写入的值，因此链上的操作构成了一个释放序列(*release sequence*)，并且初始化存储同步(对应memory_order_acquire或memory_order_seq_cst)或是前序依赖(对应memory_order_consume)的最终加载。操作链上的任何原子“读-改-写”操作可以拥有任意个存储序列(甚至是memory_order_relaxed)。

为了了解这些操作意味着什么，以及其重要性，考虑一个atomic<int>用作对一个共享队列的元素进行计数：

清单5.11 使用原子操作从队列中读取数据

```
#include <atomic>
#include <thread>

std::vector<int> queue_data;
std::atomic<int> count;

void populate_queue()
{
  unsigned const number_of_items=20;
  queue_data.clear();
  for(unsigned i=0;i<number_of_items;++i)
  {
    queue_data.push_back(i);
  }

  count.store(number_of_items,std::memory_order_release);  // 1 初始化存储
}

void consume_queue_items()
{
  while(true)
  {
    int item_index;
    if((item_index=count.fetch_sub(1,std::memory_order_acquire))<=0)  // 2 一个“读-改-写”操作
    {
      wait_for_more_items();  // 3 等待更多元素
      continue;
    }
    process(queue_data[item_index-1]);  // 4 安全读取queue_data
  }
}

int main()
{
  std::thread a(populate_queue);
  std::thread b(consume_queue_items);
  std::thread c(consume_queue_items);
  a.join();
  b.join();
  c.join();
}
```

一种处理方式是让线程产生数据，并存储到一个共享缓存中，而后调用count.store(number_of_items, memory_order_release)①让其他线程知道数据是可用的。线程群消耗着队列中的元素，之后可能调用count.fetch_sub(1, memory_order_acquire)②向队列索取一个元素，不过在这之前，需要对共享缓存进行完整的读取④。一旦count归零，那么队列中就没有更多的元素了，当元素耗尽时线程必须等待③。

当有一个消费者线程时还好，fetch_sub()是一个带有memory_order_acquire的读取操作，并且存储操作是带有memory_order_release语义，所以这里存储与加载同步，线程是可以从缓存中读取元素的。当有两个读取线程时，第二个fetch_sub()操作将看到被第一个线程修改的值，且没有值通过store写入其中。先不管释放序列的规则，这里第二个线程与第一个线程不存在先行关系，并且其对共享缓存中值的读取也不安全，除非第一个fetch_sub()是带有memory_order_release语义的，这个语义为两个消费者线程间建立了不必要的同步。无论是释放序列的规则，还是带有memory_order_release语义的fetch_sub操作，第二个消费者看到的是一个空的queue_data，无法从其获取任何数据，并且这里还会产生条件竞争。幸运的是，第一个fetch_sub()对释放顺序做了一些事情，所以store()能同步与第二个fetch_sub()操作。这里，两个消费者线程间不需要同步关系。这个过程在图5.7中展示，其中虚线表示的就是释放顺序，实线表示的是先行关系。

![](https://raw.githubusercontent.com/xiaoweiChen/Cpp_Concurrency_In_Action/master/images/chapter5/5-7.png)

图5.7 清单5.11中对队列操作的释放顺序

操作链中可以有任意数量的链接，但是提供的都是“读-改-写”操作，比如fetch_sub()，store()，每一个都会与使用memory_order_acquire语义的操作进行同步。在这里例子中，所有链接都是一样的，并且都是获取操作，但它们可由不同内存序列语义组成的操作混合。(译者：也就是不是单纯的获取操作)

虽然，大多数同步关系，是对原子变量的操作应用了内存序列，但这里依旧有必要额外介绍一个对排序的约束——栅栏(*fences*)。

### 5.3.5 栅栏

如果原子操作库缺少了栅栏，那么这个库就是不完整的。栅栏操作会对内存序列进行约束，使其无法对任何数据进行修改，典型的做法是与使用memory_order_relaxed约束序的原子操作一起使用。栅栏属于全局操作，执行栅栏操作可以影响到在线程中的其他原子操作。因为这类操作就像画了一条任何代码都无法跨越的线一样，所以栅栏操作通常也被称为“内存栅栏”(*memory barriers*)。回忆一下5.3.3节，自由操作可以使用编译器或者硬件的方式，在独立的变量上自由的进行重新排序。不过，栅栏操作就会限制这种自由，并且会介绍之前没有介绍到的“先行”和“同步”关系。

我们给在不同线程上的两个原子操作中添加一个栅栏，代码如下所示：

清单5.12 栅栏可以让自由操作变的有序

```
#include <atomic>
#include <thread>
#include <assert.h>

std::atomic<bool> x,y;
std::atomic<int> z;

void write_x_then_y()
{
  x.store(true,std::memory_order_relaxed);  // 1
  std::atomic_thread_fence(std::memory_order_release);  // 2
  y.store(true,std::memory_order_relaxed);  // 3
}

void read_y_then_x()
{
  while(!y.load(std::memory_order_relaxed));  // 4
  std::atomic_thread_fence(std::memory_order_acquire);  // 5
  if(x.load(std::memory_order_relaxed))  // 6
    ++z;
}

int main()
{
  x=false;
  y=false;
  z=0;
  std::thread a(write_x_then_y);
  std::thread b(read_y_then_x);
  a.join();
  b.join();
  assert(z.load()!=0);  // 7
}
```

释放栅栏②与获取栅栏⑤同步，这是因为加载y的操作④读取的是在③处存储的值。所以，在①处存储x先行与⑥处加载x，最后x读取出来必为true，并且断言不会被触发⑦。原先不带栅栏的存储和加载x都是无序的，并且断言是可能会触发的。需要注意的是，这两个栅栏都是必要的：你需要在一个线程中进行释放，然后在另一个线程中进行获取，这样才能构建出同步关系。

在这个例子中，如果存储y的操作③标记为memory_order_release，而非memory_order_relaxed的话，释放栅栏②也会对这个操作产生影响。同样的，当加载y的操作④标记为memory_order_acquire时，获取栅栏⑤也会对之产生影响。使用栅栏的一般想法是：当一个获取操作能看到释放栅栏操作后的存储结果，那么这个栅栏就与获取操作同步；并且，当加载操作在获取栅栏操作前，看到一个释放操作的结果，那么这个释放操作同步于获取栅栏。当然，你也可以使用双边栅栏操作，举一个简单的例子，当一个加载操作在获取栅栏前，看到一个值有存储操作写入，且这个存储操作发生在释放栅栏后，那么释放栅栏与获取栅栏是同步的。

虽然，栅栏同步依赖于读取/写入的操作发生于栅栏之前/后，但是这里有一点很重要：同步点，就是栅栏本身。当你执行清单5.12中的write_x_then_y，并且在栅栏操作之后对x进行写入，就像下面的代码一样。这里，触发断言的条件就不保证一定为true了，尽管写入x的操作在写入y的操作之前发生。

```
void write_x_then_y()
{
  std::atomic_thread_fence(std::memory_order_release);
  x.store(true,std::memory_order_relaxed);
  y.store(true,std::memory_order_relaxed);
}
```

这里里的两个操作，就不会被栅栏分开，并且也不再有序。只有当栅栏出现在存储x和存储y操作之间，这个顺序是硬性的。当然，栅栏是否存在不会影响任何拥有先行关系的执行序列，这种情况是因为一些其他原子操作。

这个例子，以及本章中的其他例子，变量使用的都是完整的原子类型。不过，正真的好处在于，使用原子操作去执行一个序列，可以避免对于一些数据竞争的未定义行为，可以会看一下清单5.2。

### 5.3.6 原子操作对非原子的操作排序

当你使用一个普通的非原子bool类型来替换清单5.12中的x(就如同你下面看到的代码)，行为和替换前完全一样。

清单5.13 使用非原子操作执行序列

```
#include <atomic>
#include <thread>
#include <assert.h>

bool x=false;  // x现在是一个非原子变量
std::atomic<bool> y;
std::atomic<int> z;

void write_x_then_y()
{
  x=true;  // 1 在栅栏前存储x
  std::atomic_thread_fence(std::memory_order_release);
  y.store(true,std::memory_order_relaxed);  // 2 在栅栏后存储y
}

void read_y_then_x()
{
  while(!y.load(std::memory_order_relaxed));  // 3 在#2写入前，持续等待
  std::atomic_thread_fence(std::memory_order_acquire);
  if(x)  // 4 这里读取到的值，是#1中写入
    ++z;
}
int main()
{
  x=false;
  y=false;
  z=0;
  std::thread a(write_x_then_y);
  std::thread b(read_y_then_x);
  a.join();
  b.join();
  assert(z.load()!=0);  // 5 断言将不会触发
}
```

栅栏仍然为存储x①和存储y②，还有加载y③和加载x④提供一个执行序列，并且这里仍然有一个先行关系，在存储x和加载x之间，所以断言⑤不会被触发。②中的存储和③中对y的加载，都必须是原子操作；否则，将会在y上产生条件竞争，不过一旦读取线程看到存储到y的操作，栅栏将会对x执行有序的操作。这个执行顺序意味着，x上不存在条件竞争，即使它被另外的线程修改或被其他线程读取。

不仅是栅栏可对非原子操作排序。你在清单5.10中看到memory_order_release/memory_order_consume对，也可以用来排序非原子访问，为的是可以动态分配对象，并且本章中的许多例子都可以使用普通的非原子操作，去替代标记为memory_order_relaxed的操作。

对非原子操作的排序，可以通过使用原子操作进行，这里“前序”作为“先行”的一部分，就显得十分重要了。如果一个非原子操作是“序前”于一个原子操作，并且这个原子操作需要“先行”与另一个线程的一个操作，那么这个非原子操作也就“先行”于在另外线程的那个操作了。 这一序列操作，就是在清单5.13中对x的操作，并且这也就是清单5.2能工作的原因。对于C++标准库的高阶同步工具来说，这些都是基本，例如互斥量和条件变量。可以回看它们都是如何工作的，可以对清单5.1中简单的自旋锁展开更加深入的思考。

使用`std::memory_order_acquire`序列的lock()操作是在flag.test_and_set()上的一个循环，并且使用`std::memory_order_release`序列的unlock()调用flag.clear()。当第一个线程调用lock()时，标志最初是没有的，所以第一次调用test_and_set()将会设置标志，并且返回false，表示线程现在已锁，并且结束循环。之后，线程可以自由的修改由互斥量保护的数据。这时，任何想要调用lock()的线程，将会看到已设置的标志，而后会被test_and_set()中的循环所阻塞。

当线程带锁线程完成对保护数据的修改，它会调用unlock()，相当于调用带有`std::memory_order_release`语义的flag.clear()。这与随后其他线程访问flag.test_and_set()时调用lock()同步(见5.3.1节)，这是因为对lock()的调用带有`std::memory_order_acquire`语义。因为对于保护数据的修改，必须先于unlock()的调用，所以修改“先行”于unlock()，并且还“先行”于之后第二个线程对lock()的调用(因为同步关系是在unlock()和lock()中产生的)，还“先行”于当第二个线程获取锁后，对保护数据的任何访问。

虽然，其他互斥量的内部实现不尽相同，不过基本原理都是一样的:在某一内存位置上，lock()作为一个获取操作存在，在同样的位置上unlock()作为一个释放操作存在。

## 5.4 总结

在本章中，已经对C++11内存模型的底层只是进行详尽的了解，并且了解了原子操作能在线程间提供基本的同步。这里包含基本的原子类型，由`std::atomic<>`类模板特化后提供；接口，以及对于这些类型的操作，还要有对内存序列选项的各种复杂细节，都由原始`std::atomic<>`类模板提供。

我们也了解了栅栏，了解其如何让执行序列中，对原子类型的操作同步成对。最后，我们回顾了本章开始的一些例子，了解了原子操作可以在不同线程上的非原子操作间，进行有序执行。

在下一章中，我们将看到如何使用高阶同步工具，以及原子操作并发访问的高效容器设计，还有我们将写一些并行处理数据的算法。