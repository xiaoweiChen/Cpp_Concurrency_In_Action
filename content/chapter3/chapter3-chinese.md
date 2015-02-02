#第三章 线程间共享数据

**本章主要内容**

>共享数据带来的问题<br>
>使用互斥量保护数据<br>
>数据保护的替代方案<br>

使用线程并发的一个好处是可以直接共享数据，我们在本章开始前，已经对线程管理有所了解了，现在让我们来看一下“共享数据的那些事”。

想象一下，当你和你的一个朋友合租在一个公寓中。公寓只有一个厨房和一个卫生间。当你要使用卫生间的时候，你的朋友一直在占用卫生间，那么你就会不能使用了（除非你们特别友好，好到可以在同一时间使用一个房间）。同样的问题也会出现在厨房，当有一个组合式烤箱，在烤香肠的时候，也在做一个蛋糕，那么我们可能得不到我们想要的食物（会得到不想要的香肠味的蛋糕）。此外，当我们在公共空间将一件事做到一半时，发现某些需要的东西被别人接走了，或者，当我们离开的一小段时间内有些东西被改动了，这都会令我们沮丧。

同样的问题，也困扰着线程。当你想要在线程见共享数据的时候，你必须定一些规矩用来限定线程可访问的数据位，还有，当一个线程更新了共享数据，需要对其他线程进行告知。从易用性上来说，在同一进程中的多个线程进行数据共享，既有利，又有弊。错误的共享数据使用是导致并发先关bug产生的一个主要原因，并且后果要比香肠味的蛋糕更加严重。

本章就在C++中，进行安全的数据共享为主题。避免上述及其他潜在问题的发生，并将共享数据的优势发挥到最大。

##3.1 共享数据带来的问题

当问题涉及到共享数据时，则该问题很可能是共享数据被修改所导致。如果所有共享数据都是只读的，那么没问题，因为只读操作不会影响到数据，更不会对数据进行修改，所以所有线程获取到的都是同样的数据。但是，当一个或多个线程要修改共享数据时，就会产生很多潜在的麻烦。在这种情况下，你就必须小心并谨慎的确保一切都工作正常。

不变量（*invariants*）的概念已经广泛使用，并且对程序员们编写的程序产生帮助——关于特殊结构体的描述，比如，“此变量包含列表中的项数”。这些不变量通常会在一次更新中被破坏，特别是哪些比较复杂的数据结构，或者一次更新就要改动很多值。

一个双链表，每一个节点都有一个指针指向列表中下一个节点，还有另外一个指针指向前一个节点。其中不变量就是节点A中指向“下一个”节点B的指针，还有前向指针。为了从列表中删除一个节点，其两边的节点的指针都需要更新。当其中一边更新完成时，不变量就被破坏了，直到另一边也完成更新；
在两边都完成更新后，不变量就又稳定了。

从一个列表中删除一个节点的步骤如下（如图3.1）<br>
1. 找到要删除的节点N<br>
2. 更新前一个节点指向N的指针，让这个指针指向N的下一个节点<br>
3. 更新后一个节点指向N的指针，让这个指正指向N的前一个节点<br>
4. 删除节点N<br>

![](https://raw.githubusercontent.com/xiaoweiChen/Cpp_Concurrency_In_Action/master/images/chapter3/3-1.png)

图3.1 从一个双链表中删除一个节点

如你所见，在图中的b和c中，在相同的方向上指向和原来已经不一致了，这就破坏不变量。

在线程间最简单的潜在问题就是修改共享数据，致使不变量遭到破坏。当你不做些事来确保在这个过程中不会有其他线程进行访问的话，可能就有线程访问到刚刚删除一边的节点，这样的话，线程就读取到要删除的那个节点的数据（因为只有一边的连接被修改，如图3.1中的b一般），所以，这里说不变量被破坏。破坏不变量的后果可能多种多样；当其他线程按从左往右的书序来访问列表，那么它将跳过那个被删除的节点。在另一方面，如有第二个线程尝试删除图中右边的节点，那么它可能会让数据结构产生永久性的损坏，并使程序崩溃。无论结果如何，这都是并行代码常见错误产生的原因：条件竞争（*race condition*）。

###3.1.1 条件竞争

假设你去电影院买电影票。如果这是一家很大的电影院，有很多收银员，那么很多人可以在同一时间进行电影票的选购。当另一个收银台也在卖你想看的这场电影的电影票，那么你的座位选择范围就取决于在之前已预定的座位。当只有少量的座位剩下，这就意味着：这可能是一场抢票比赛，看谁能抢到最后一张票。这就是一个条件竞争的例子：你的座位（或者你的电影票）都取决于两种购买方式的相对顺序。

在并发中，竞争条件的形成，取决于一个以上线程的相对执行顺序；每个线程都抢着完成自己的任务。大多数情况下，即便它们会改变执行顺序，这种竞争也是良性的，其结果也是可以接受的。例如，如果两个线程同时向一个处理队列中添加任务，谁先谁后就没有什么影响，系统提供的不变量保持不变。当不变量遭到破坏时，才会导致条件竞争，例如双向链表的那个例子。在并发中，条件竞争这个数据通常用于表示“有问题的”（*problematic*）条件竞争；我们对良性的，且不产生问题的条件竞争不感兴趣。在C++标准中，也定义了数据竞争（*data race*）这个术语，因为再并发中去修改一个独立对象（参考5.1.2节），所以这是一种特殊的条件竞争；数据竞争是（可怕的）未定义行为（*undefine behavior*）的起因。

有问题的条件竞争通常发生在，完成对多于一个的数据块的修改时，例如，对两个连接指针的修改（在图3.1中）。因为操作要访问两个独立的数据块，数据块将会被独立的指令所修改，并且另一个线程可能在只完成一半的时，就对数据块进行访问。条件竞争很难找，并且很难复现，因为概率太低。当如CPU指令那样的连续修改完成后，这个问题能再次复现出的几率就相当低了，即使这个数据结构可以让其他并发线程访问。当系统负载增加时，随着执行数量增加，执行序列的问题复现的概率也在增加。这几乎是不可避免的，这样的问题会出现在负载比较大的情况下。条件竞争通常是时间敏感的，所以，当运行在调试模式时，它们常会完全消失；因为，调试模式会影响程序的执行时间，即使影响不多。

当你以写多线程程序为生，条件竞争就会成为你生命中不快乐的源泉；在编写软件时，会使用大量复杂的操作，用来避免恶性条件竞争。

###3.1.2 避免恶性条件竞争

这里有很多方法来解决恶性条件竞争。最简单的办法是对数据结构采用某种保护机制，确保只有进行修改的线程才能看到不变量被破坏时的中间状态。从其他访问线程的角度来看，改变要不就是已经完成了，要不就还没开始。C++标准库提供很多类似的机制，我们会在本章进行介绍。

另一个选择是改动数据结构和不变量的设计，修改完的结构需能完成一系列不可分割的变化，也就是保证每个不变量保持不变的状态。这就是无锁编程（*lock-free programming*）的方式，不过，这种方式很难得到正确的结果。如果你做到这个级别上了，无论是内存模型存在细微的差异，还是，线程有潜能看到这组值，都会让工作变的复杂。内存模型的问题将在第5章讨论，无锁编程将在第7章讨论。

另一种处理条件竞争的方式是，使用事务（*transacting*）的方式去处理数据结构的更新，这里的处理就如同数据库进行更新一样。所需的一些修改数据和读取都存储在一个事务日志中，然后将之前的操作合为一步，再进行提交。当数据结构已经被另一个线程修改后，或处理已经重启的情况下，提交会无法处理。这称作为“软件事务内存”（*software transactional memory ( STM )*），在理论研究中，这是一个很活跃研究领域。这个概念将不会在本书中再进行介绍，因为在C++中没有对STM进行直接支持。但是，基本思想我们会在后面提及。

保护共享数据结构的最基本的方式，是使用C++标准库提供的互斥量（*mutex*），我先来看一下这个。

##3.2 使用互斥量保护共享数据

当你的程序中有共享数据，你肯定不想让其陷入条件竞争，或是不变量被破坏。如果你将所有代码可访问的数据标为互相排斥（*mutually exclusive*），就很不优雅；这样的话，当其中任一线程运行访问数据时，其他线程想要访问数据，就必须等待第一个线程结束吗？这样很有可能会让一个线程看到已破坏的不变量，也就是第一个线程正在做修改的时候。

当访问共享数据前，使用互斥量将相关数据锁住，并且当访问结束后，将数据解锁。线程库需要保证，当一个线程使用一个特定互斥量锁住共享数据，其他所有的线程想要访问被锁住的数据，都必须等到之前那个线程对数据进行解锁后，才能再进行访问。这就保证了所有线程只能看到自治区域内的共享数据，并避免不变量破坏。

互斥量是C++中用来保护数据的一种常用手段，但是互斥量不是“银弹”；它可以用来保护正确的数据（见3.2.2节），也可以避免接口中存在着的条件竞争（见3.2.3节）。互斥量自身也有问题，造成死锁（见3.2.4节），对数据保护的太多（或太少）（见3.2.8节）。

让我们从最基本的开始吧。

###3.2.1 在C++中使用互斥量

在C++中，你可以使用`srd::mutex`来创建一个互斥量实例，通过调用成员函数`lock()`进行上锁；同样的，也可以调用成员函数`unlock()`进行解锁。不过，这里不推荐实践中直接去调用成员函数，因为调用成员函数就以为着，你必须记着在每个函数出口都要去调用`unlock()`，也包括异常的情况。C++标准库提供了一个`std::lack_guard`类模板，实现了是RAII风格的互斥量；其会在构造的时候提供已锁的互斥量，并在析构的时候进行解锁，这就保证了一个已锁的互斥量可以被正确的解锁。在下面的程序清单中，展示了如何在多线程程序中，使用`std::mutex`构造的`std::lock_guard`实例，对一个列表进行访问保护。`std::mutex`和`std::lock_guard`都在<mutex>头文件中声明。

列表3.1 使用互斥量保护列表
```c++
#include <list>
#include <mutex>
#include <algorithm>

std::list<int> some_list;    // 1
std::mutex some_mutex;    // 2

void add_to_list(int new_value)
{
  std::lock_guard<std::mutex> guard(some_mutex);    // 3
  some_list.push_back(new_value);
}
bool list_contains(int value_to_find)
{
  std::lock_guard<std::mutex> guard(some_mutex);    // 4
  return std::find(some_list.begin(),some_list.end(),value_to_find) != some_list.end();
}
```

在清单3.1中，这里有一个全局变量 ①，这个全局变量又一个全局的互斥量来保护 ②。在**add_to_list()** ③ 和**list_contains()** ④ 函数中使用`std:;lock_guard<std::mutex>`，使得在这两个函数中对数据的访问是互斥的：**list_contains()**不可能看到被**add_to_list()**修改到一半的列表。

在某些情况下，这个全局变量使用的是没问题的；不过，在大多数情况下，需要保护的数据通常会与互斥量放在同一个类中，而不是定义成全局变量。这是一个标准的面向对象设计准则：将其放在一个类中，就可让他们联系在一起，也可对类的功能进行封装，并进行数据保护。在这种情况下，函数**add_to_list**和**list_contains**可以作为这个类的成员函数，并且互斥量和要保护的数据，在类中都需要定义为private成员，这就会让访问数据的代码变的清晰，并且容易看出在什么时候对互斥量进行上锁。当所有成员函数都会在调用时对数据上锁，在结束的时候对数据解锁，那么这就为所有的访问保证了数据的不变量不被破坏。

当然，也不是什么时候都这么理想，聪明的你一定早就注意到了：当其中一个成员函数返回的是保护数据的指针或引用时，即使成员函数都很好的对数据进行了保护，但是你破坏了他们的保护。具有访问能力的指针或引用可以访问（并可能修改）被保护的数据，而不会被互斥锁限制。互斥量保护的数据需要对接口的设计相当谨慎，要确保互斥量能锁住任何对保护数据的访问，并且不留后门。

###3.2.2 保护共享数据的结构

如你之前所见，单使用互斥量保护数据是很困难的，需要配合`std::lock_guard`对象在每个成员函数中，才能做到正真的保护；一个迷失的指针或引用，将会让这种保护形同虚设。在这个层面上，检查迷失指针或引用是很容易的；只要让成员函数不返回指针或引用，通过函数的返回值或一个输出参数，来保证数据的安全性。如果你想更深入的讨论，那就不会这么简单了——或者说没有什么是简单的。在确保成员函数不会传出指针或引用的同时，检查不通过指针或引用的方式来调用成员函数也是很重要的（当这个操作不在你的控制下）。这是很危险的：这些函数可能在互斥量没有保护到的地方，存储着指针或者引用。特别危险的是：将保护数据作为一个运行时参数，如同下面清单中所示那样。

清单3.2 无意中传递了保护数据的引用
```c++
class some_data
{
  int a;
  std::string b;
public:
  void do_something();
};

class data_wrapper
{
private:
  some_data data;
  std::mutex m;
public:
  template<typename Function>
  void process_data(Function func)
  {
    std::lock_guard<std::mutex> l(m);
    func(data);    // 1 传递“保护”数据给用户函数
  }
};

some_data* unprotected;

void malicious_function(some_data& protected_data)
{
  unprotected=&protected_data;
}

data_wrapper x;
void foo()
{
  x.process_data(malicious_function);    // 2 传递一个恶意函数
  unprotected->do_something();    // 3 在无保护的情况下访问保护数据
}
```

在这个例子中，**process_data**看起来是十分害的。虽然`std::lack_guard`能够很好的保护数据，但当调用了用户提供的函数**func** ① ，就会让**foo**传递**malicious_function**函数指针，这个函数指针就可以用来传出保护数据 ②；在之后的**do_something()**调用时，就可以直接使用保护数据，并完全避开了互斥锁。

幸运的是，这段问题代码还没有做什么出格的事情：因为代码已标记所有可访问的数据结构是互斥的。但不幸的是，这种情况下，C++线程库无法提供帮助；它只能在程序中，用正确的互斥锁去保护你的数据。从乐观的角度上看，在这种情况下，你是有方法可循的：切勿在互斥锁范围外传递保护数据的指针或引用，无论是函数返回值，还是存储在外部可见内存，亦或是将他们已参数的形式传递到用户提供的函数中去。

这在问题在使用互斥量保护共享数据时经常发生，从长远角度来看，这就会形成一个潜在的陷阱。

在下一节中，你将会看到，即便是使用了互斥量对数据进行了保护，但是条件竞争依旧会存在。

###3.2.3 接口中的条件竞争

只因为你使用了互斥量或其他机制保护了共享数据，你就不必再为条件竞争所担忧；你依旧需要对保护所做的保护措施进行确定。可以参照一下双向列表那个例子。当节点正在删除，或节点正在修改时，为了让一个线程安全的删除一个节点，你需要防止对这三个节点的并发访问。如果你只对指向每个节点的指针进行访问保护，那就和没有使用互斥量一样，因为条件竞争仍会发生——每步中的指针是不需要保护的，但是整个数据结构和整个删除操作，是需要保护的。这里，最简单的解决方案就是使用一个互斥量来保护整个链表，如同清单3.1所做。

尽管每个对链表的独立操作是安全的，但不意味着你就能走出困境；在一个很简单的接口中，你依旧可能遇到条件竞争。在一个`std::stack`的栈结构数据容器（清单3.3）。除了构造和`swap()`（互换）以外，你只能对`std::stack`做五件事：`push()`一个新元素进栈，`pop()`一个元素出栈，`top()`查看栈顶元素，`empty()`判断栈是否是空栈，`size()`了解栈中有多少个元素。当你修改`top()`，使其返回一个拷贝，而非一个引用，并且对内部数据使用一个互斥量进行保护，那么这个接口就会有条件竞争的存在。这不仅仅是基于互斥量实现的问题；在一个无锁实现的接口中，条件竞争依旧会产生。

清单3.3 `std::stack`容器的实现
```c++
template<typename T,typename Container=std::deque<T> >
class stack
{
public:
  explicit stack(const Container&);
  explicit stack(Container&& = Container());
  template <class Alloc> explicit stack(const Alloc&);
  template <class Alloc> stack(const Container&, const Alloc&);
  template <class Alloc> stack(Container&&, const Alloc&);
  template <class Alloc> stack(stack&&, const Alloc&);
  
  bool empty() const;
  size_t size() const;
  T& top();
  T const& top() const;
  void push(T const&);
  void push(T&&);
  void pop();
  void swap(stack&&);
};
```

`empty()`和`size()`的实现有问题，其结果是不能信赖。虽然在调用并返回时可能是正确的，但其他线程也是可以随意的访问栈的，并且可能`push()`多个新元素到栈中，也可能`pop()`出一些已在栈中的元素。这样的话，之前从`empty()`和`size()`得到的结果就有问题了。

特别是，当栈实现是非共享的，使用`empty()`检查完栈，当栈非空时，再调用`top()`访问栈顶部的元素。如下代码所示：

```c++
stack<int> s;
if (! s.empty()){    // 1
  int const value = s.top();    // 2
  s.pop();    // 3
  do_something(value);
}
```

以上是单线程安全代码：对一个空栈使用`top()`，是未定义行为，会产生各种奇怪的结果。对与一个共享的栈对象，这样的调用顺序就未必安全了，因为可能会发生在其他线程调用`pop()`对栈中的元素进行删除时，另一个线程正在调用`empty()` ①，并在判断非空后调用` top()`②。这是一个经典的条件竞争，使用互斥量对内部数据进行保护，但依旧不会阻止条件竞争的发生。

这个问题有解吗？有！

这个问题发生在接口设计上，所以解决的方法也就是改变接口实现。有人会问：怎么改？在这个简单的例子中，当调用`top()`时，发现栈已经是空的了，那么就抛出异常。虽然这能直接解决这个问题，但这是一个笨拙的解决方案，这样的话，即使`empty()`返回false的情况下，你都异常捕获机制。本质上，这样的改变会让`empty()`成为一个冗余函数。

当你仔细的观察过之前的代码段，你就会发现另一个潜在的条件竞争，就是在调用`top()` ② 和`pop()` ③ 之间。当两个线程运行在之前的代码中，并且都引用同一个栈对象。这就不是寻常的情况了；当为了性能使用线程，多个线程在使用不同的数据执行相同的操作，这是很平常的；并且，一个共享栈对象是理想的分工工具。假设，初始栈中只有两个元素，这时`empty()`和`top()`不会有竞争，同时考虑一下潜在的运行模式。

当栈被一个内部互斥量所保护时，在任一时间内，只有一个线程可以调用栈的成员函数，所以调用可以很好交错运行，并且do_something()是可以并发运行的。在表3.1中，会展示一种可能的执行顺序。

表3.1 一种可能执行顺序

| Thread A | Thread B |
| ------------ |--------------|
|if (!s.empty);||
||if(!s.empty);|
|int const value = s.top();||
||int const value = s.top();|
|s.pop();||
|do_something(value);|s.pop();|
||do_something(value);|

如你所见，当线程运行时，在`top()`的两次调用中，栈没被修改，所以每个线程能得到同样的值。不仅是这样，在调用`pop()`函数调用的过程中（两次），`pop()`函数没有被调用。这样，在其中一个值再读取的时候，虽然不会出现“写后读”的情况了，但其值被处理了两次。这是另一种条件竞争，比未定义`empty()/top()`的竞争更加严重；虽然其结果依赖于`do_something()`的结果，但是，因为看起来没有任何错误，就会让这个漏洞变的很难找。

这就需要接口设计上需要更大的改动，提议之一就是使用同一互斥量来保护`top()`和`pop()`。Tom Cargill[1] 指出当一个对象的拷贝构造函数在栈中抛出一个异常，这样的处理方式就会有问题。在Herb Sutter[2]看来，这个问题可以从“异常安全”的角度完美解决，不过潜在的条件竞争，可能达成一些新的条件组合。

说一些大家没有意识到的问题：假设有一个`stack<vector<int>>`，`vector`是一个动态容器，所以当你拷贝一个`vetcor`，标准库会从堆上分配很多内存来完成这次拷贝。当这个系统处在重度负荷，或有严重的资源限制的情况下，这种内存分配就会失败，所以`vector`的拷贝构造函数可能会抛出一个`std::bad_alloc`异常。当`vector`中存有大量元素时，这种情况发生的坑内性更大。当`pop()`函数返回“弹出值”时（也就是从栈中将这个值移除），就会有一个潜在的问题：当这个值被返回到调用函数的时候，栈才被改变；但是当拷贝数据的时候，调用函数抛出一个异常会怎么样？ 如果这种事情真的发生了，要弹出的数据将会丢失；它的确从栈上一出了，但是拷贝失败了！`std::stack`的设计人员将这个操作分为两部分：先获取顶部元素（`top()`），然后从栈中移除（`pop()`）。这样，在不能安全的将元素拷贝出去的情况下，栈中的这个数据还依旧存在，没有丢失。当问题是堆空间不足，应用可能会释放一些内存，然后再进行尝试。

不幸的是，这样的分割却制造了你本想避免，或消除的条件竞争。幸运的是，我们还有的别的选项，但是这些选项是要付出代价的。

**选项1： 传入一个引用**

第一个选项是将一个变量的引用作为一个参数，传入pop()函数中来获取想要的“弹出值”：

```c++
std::vector<int> result;
some_stack.pop(result);
```

在很多情况下，这种方式表现的很好。但是，其也存在明显的缺点：需要提前构造出一个堆中类型的实例，来接收目标值。对于一些类型，这样做是不现实的，因为提前构造一个实例，从时间和资源的角度上来看，都是不经济的。对于其他的一些类型，这样做也不总能行，因为构造函数需要的一些参数，在代码的这个阶段不一定是可用的。最后，他需要存储类型是可赋值的。这是一个很重要的限制：很多用户定义类型都不支持赋值，虽然他们可能支持移动构造，甚至是拷贝构造（从而允许返回一个值）。

**选项2：无异常抛出的拷贝构造函数或移动构造函数**

对于一个有返回值的pop()函数来说，只有“异常安全”问题（当返回值时可以抛出一个异常）。很多类型都有拷贝构造函数，它们是不会抛出异常的，并且随着新标准中对“右值引用”的支持（详见附录A，A.1节），在这个前提下，很多类型都将会有一个移动构造函数，即使他们和拷贝构造函数做着相同的事情，它也不会抛出异常。一个有用的选项是能限制对线程安全的栈的使用，并且能让栈安全的返回返回所需的值，而不会抛出异常。

虽然安全，但非可靠。尽管能在编译时，使用`std::is_no_throw_copy_constructible`和`std::is_nothrow_move_constructible`类型特征，能让拷贝或移动构造函数不抛出异常，但是这个局限性太强。很多用户定义类型有可抛出异常的拷贝构造函数，没有移动构造函数；或是，都不抛出异常的构造函数（这种改变会随着C++11中左值引用，越来越为大众所用）。如果一种类型不能存储线程安全的堆栈，想想是多么的不幸呀。

**选项3：返回指向弹出值的指针**

第三个选择是返回一个指向弹出元素的指针，而不是直接返回值。指针的优势是自由拷贝，并且不会产生异常，这样你就能避免Cargill提到的异常问题了。缺点就是返回一个指针需要对对象的内存分配进行管理，不过对于简单类型（比如：int），内存管理的开销要远大于直接进行值的返回。对于选择这个方案的接口，`std::shared_ptr`是一个不错的选择；不仅是因为它能避免内存泄露（因为当对象中指针销毁时，对象也会被销毁），而且标准库能够完全控制内存分配方案，其也就不需要new和delete操作了。这种优化是很重要的：因为每个在堆栈中的对象，都需要用new进行独立的内存分配，这样一来，相较于非线程安全版本，这个方案的开销是相当大的。

**选项4：“选项1 + 选项2”或 “选项1 + 选项3”**

对于通用的代码来说，灵活性不应忽视。当你已经选择了选项2或3时，再去选择1也是很容易的。这些选项提供给用户，让用户自己选择对于他们自己来说最合适，最经济的方案。

**例：定义线程安全的堆栈**

清单3.4中是一个接口没有条件竞争的堆栈类的定义，并且它实现了选项1和选项3：这里重载了pop()，使用一个局部引用去存储弹出值，并返回一个`std::shared_ptr<>`对象。它有一个简单的接口，只有两个函数：push()和pop();

清单3.4 线程安全的堆栈类定义（概述）

```c++
#include <exception>
#include <memory>  // For std::shared_ptr<>

struct empty_stack: std::exception
{
  const char* what() const throw();
};

template<typename T>
class threadsafe_stack
{
public:
  threadsafe_stack();
  threadsafe_stack(const threadsafe_stack&);
  threadsafe_stack& operator=(const threadsafe_stack&) = delete; // 1 赋值操作被删除

  void push(T new_value);
  std::shared_ptr<T> pop();
  void pop(T& value);
  bool empty() const;
};
```

削减接口可以获得最大程度的安全；甚至限制对栈的一些操作。这个栈是不能赋值的，因为赋值操作已经删除了 ① （详见附录A，A.2节），并且这里没有swap()函数。这个栈可以拷贝的，假设栈中的元素是可以拷贝的。当栈为空时，pop()函数会抛出一个empty_stack异常，所以在empty()函数被调用后，其他部件还能正常工作。如选项3描述的那样，使用`std::shared_ptr`可以避免内存分配管理的问题，并避免多次使用new和delete操作。堆栈中的五个操作，现在就剩下三个：push(), pop()和empty()。这里empty()都有些多余。简化接口更有利于数据控制；你就可以保证互斥量将一个操作完全锁住。下面清代中的代码将展示一个简单的实现——对`std::stack<>`的封装。

清单3.5 扩充线程安全的堆栈
```c++
#include <exception>
#include <memory>
#include <mutex>
#include <stack>

struct empty_stack: std::exception
{
  const char* what() const throw();
};

template<typename T>
class threadsafe_stack
{
private:
  std::stack<T> data;
  mutable std::mutex m;
public:
  threadsafe_stack(){}
  threadsafe_stack(const threadsafe_stack& other)
  {
  std::lock_guard<std::mutex> lock(other.m);
  data = other.data; // 1 在构造函数体中的执行拷贝
  }
  threadsafe_stack& operator=(const threadsafe_stack&) = delete;

  void push(T new_value)
  {
    std::lock_guard<std::mutex> lock(m);
    data.push(new_value);
  }
  std::shared_ptr<T> pop()
  {
  std::lock_guard<std::mutex> lock(m);
  if(data.empty()) throw empty_stack(); // 在调用pop前，检查栈是否为空
  std::shared_ptr<T> const res(std::make_shared<T>(data.top())); // 在修改堆栈前，分配出返回值
  data.pop();
  return res;
  }
  void pop(T& value)
  {
  std::lock_guard<std::mutex> lock(m);
  if(data.empty()) throw empty_stack();
  value=data.top();
  data.pop();
  }
  bool empty() const
  {
  std::lock_guard<std::mutex> lock(m);
  return data.empty();
  }
};
```

这个堆栈的实现是可拷贝的——拷贝构造函数对互斥量上锁，然后再去拷贝内部堆栈。在构造函数体中①做的拷贝可以使用互斥量确保复制的结果，这种方式要比成员初始化列表好很多。

如同在之前对top()和pop()函数的讨论，有问题的条件竞争已经出现了，因为锁住的粒度太小；需要保护的操作并未全覆盖到。使用互斥量的问题是，锁住的颗粒过大；还有一个问题就是一个全局互斥量，要去保护全部共享数据。在一个系统中存在有大量的共享数据，这会抵消并发带来的性能提升，因为线程可以强制运行一次，甚至可以访问不同地方的数据。在为多处理系统设计的第一版Linux内核中，其使用了一个全局内核锁。虽然这个锁能正常工作，但在双核处理系统的性能要远差与两个单核系统的性能，四核系统就更不能提了。有太多请求去竞争占用内核，这使得依赖于处理器运行的线程没有办法很好的工作。随后修正的Linux内核加入了一个细粒度锁方案，这时四核处理系统的性能就和单核处理四通的四倍差不多了，因为少了很多内核竞争。

细粒度锁的问题是，有时候你需要不止一个互斥量去锁住一个操作，进而保护所有的数据。如前所述，有时候正确的事，会增大被互斥量覆盖数据的粒度，所以只需要一个互斥量需要锁住。但是，有时这种方案并不受欢迎，比如，当互斥量正在保护一个独立的类实例。在这种情况下，锁定的状态的下一个阶段，要不离开锁定区域，将锁定区域还给用户，要不又一个独立的互斥量去保护这个类的全部实例，这两种方式都不是很理想。

当你的一些操作需要两个或两个以上的互斥量时，另一个潜在的问题将会显现：死锁(*deadlock*)。它与条件竞争完全相反：不同于两个线程去争第一，这里他们会互相等待，从而什么都没有做。

###3.2.4 死锁：问题描述及解决方案

试想你有一个玩具，这个玩具由两部分组成，你必须拿到这两个部分，你才能去玩，例如，一个玩具鼓，还需要一个鼓锤才能玩。现在你有两个小孩，他们都很喜欢玩这个玩具。当其中一个孩子拿到了鼓和鼓锤是，那就可以尽情的玩耍了。当另一孩子想要玩，他就得等待另一孩子玩玩才行。再试想，鼓和鼓锤被放在不同的玩具箱里，并且两个孩子在同一时间里都想要去敲鼓。之后，他们就去玩具箱里面找这个鼓。其中一个找到了鼓，并且另外一个找到了鼓锤。现在问题就来了，除非其中一个孩子决定另一个先玩，他可以把自己的那部分给另外一个孩子；但是，当他们都紧握着自己所有的部分，而不给予，那么这个鼓谁都没法玩。

你没有孩子去争抢玩具，但是线程有对锁的竞争：一对线程需要对他们所有的互斥量做一些操作，其中没有线程都有一个互斥量，且等待另一个解锁。没有线程能工作，因为他们都在等待对方释放互斥量。这种情况就是死锁，它的最大问题就是由两个或两个以上的互斥量来锁定一个操作。

对于避免死锁的一般建议，就是让两个互斥量总以相同的顺序上锁：当你总在互斥量B之前锁住互斥量A，就永远不会死锁。某些情况下是可以这样用的，因为不同的互斥量用于不同的地方，当有时事情却没那么简单，比如：当有多个互斥量保护同一个类的一个独立实例时。试想，一个操作对同一个类的两个不同实例进行数据的交换操作；为了保证数据交换操作的正确性，就要避免数据被并发修改，并确保每个实例上的互斥量都能锁住自己要保护的区域。不过，当选择一个固定的顺序时（例如，实例提供的第一互斥量作为第一个参数，提供的第二个互斥量为第二个参数），可能会适得其反：**在参数交换了之后**，两个线程试图在相同的两个实例间进行数据交换，程序死锁了！

我们很幸运，C++标准库有办法解决这个问题，`std::lock`——可以一次性锁住多个（两个以上）的互斥量，并且没有副作用（死锁风险）。在下面的程序清单中，就来看一下怎么再一个简单的交换操作中使用`std::lock`。

清单3.6 在交换操作中使用`std::lock()`和`std::lock_guard`
```c++
// 这里的std::lock()需要包含<mutex>头文件
class some_big_object;
void swap(some_big_object& lhs,some_big_object& rhs);
class X
{
private:
  some_big_object some_detail;
  std::mutex m;
public:
  X(some_big_object const& sd):some_detail(sd){}

  friend void swap(X& lhs, X& rhs)
  {
    if(&lhs==&rhs)
      return;
    std::lock(lhs.m,rhs.m); // 1
    std::lock_guard<std::mutex> lock_a(lhs.m,std::adopt_lock); // 2
    std::lock_guard<std::mutex> lock_b(rhs.m,std::adopt_lock); // 3
    swap(lhs.some_detail,rhs.some_detail);
  }
};
```

首先，需要检查参数，查看它们是否是不同的实例，因为操作试图获取`std::mutex`对象上的锁，所以当其已经被获取时，所发生的动作就为无法确定的(*undefined behavior*)。(一个互斥量可以在同一线程上多次上锁，标准库中`std::recursive_mutex`提供这样的功能。详情见3.3.3节)。然后，调用`std::lock()`①锁住两个互斥量，并且两个` std:lock_guard`实例已经创建好了②③，还有一个互斥量。提供`std::adopt_lock`参数除了表示`std::lock_guard`的对象已经上锁外，还表示应使用互斥量现成的锁，而非尝试创建新的互斥锁。

这就能保证在大多数情况下，函数退出时互斥量能被正确的解锁(这里保护操作可能会抛出一个异常)；它也允许使用一个简单的“return”作为返回。还有，需要注意的是，当使用`std::lock`去锁lhs.m或rhs.m时，可能会抛出异常；这种情况下，异常会传播到`std::lock`之外。当`std::lock`成功的获取一个互斥量上的锁，并且当其尝试从另一个互斥量上再获取锁时，就会有异常抛出，第一个锁也会随着异常的产生而自动释放：`std::lock`对要可上锁的互斥量提供“全或无语义”（*all-or-nothing semantics*）。

虽然` std::lock`可以帮助你在这情况下(获取两个以上的锁)避免死锁，但它没办法帮助你获取他们其中之一。在这种情况下，你不得不依赖于作为开发者的纪律性(译者：这里也就是经验)，来确保你的程序不会死锁。这并不简单：死锁是多线程编程中一个令人相当头痛的问题，并且死锁经常是不可预见的，因为在大多数时间里，所有工作都能很好的完成。不过，这里也一些相对简单的规则能帮助写出“无死锁”(*deadlock-free*)的代码。

###3.2.5 避免死锁的进阶指引

死锁不仅仅发生在锁上，即使在锁上发生是最常见的原因；你可以通过两个线程来构造死锁，在无锁的情况下，只需要每个`std::thread`对象调用join()。在这种情况下，没有线程可以继续进行，因为他们正在等待其他线程结束，就像孩子们为他们的玩具而争吵一样。这个简单的循环会发生在任何地方，一个线程会等待另一个线程执行一些操作，其他线程同时也会等待第一个线程结束，这种情况不仅限于两个线程：三个或更多线程的循环也会发生死锁。这里避免死锁的知道可以总结为一点：当机会等着你时，不要拱手让人(*don’t wait for another thread if there’s a chance it’s waiting for you*)。这里提供一些个人的指导建议，去识别并消除让其他线程等待你的可能。

**避免嵌套锁**

第一个建议是最简单的：当已获得一个锁时，再别去获取第二个(*don’t acquire a lock if you already hold one*)。如果你能坚持这个建议，在锁的使用方面，你就不可能看到死锁的情况，因为每个线程只持有一个锁。不过，你可能会在其他方面受到死锁的困扰(比如：线程间的互相等待)，即使互斥锁造成的死锁是最常见的。当你需要获取多个所，使用一个`std::lock`来做这件事，避免产生死锁。

**避免在持有锁时调用用户提供的代码**

第二个建议是第二简单的。因为代码是用户提供的，你就没有办法确定用户要做什么；用户程序可能做任何事情，包括获取锁。你在持有锁的情况下，调用用户提供的代码。如果用户代码要获取一个所，你就会违反第一个指导意见，并造成死锁。有时，这是无法避免的；当你正在写一份通用代码，例如3.2.3中的栈，每一个操作的参数类型，都在用户提供的代码中定义。在这种情况下，你需要其他指导意见来帮助你。

**使用固定顺序获取锁**

当有硬性条件，要求你获取两个以上（包括两个）的锁，并且你不能使用如`std::lock`单独操作来获取它们，那么最好在每个线程上，用固定的顺序获取它们获取它们(锁)。我在3.2.4节中提到一种当需要获取两个互斥量时，避免死锁的方法：关键是如何在线程之间，一致性的定义获取顺序。在一些情况下，这种方式相对简单。比如，3.2.3节中的栈——每个栈实例中都内置有互斥量，但是对数据成员存储的操作上，这个栈需要带调用用户提供的代码。碎蛋，你可以添加一些约束，对栈上的存储的数据项不做任何操作，对数据项的处理仅限于栈自身。这会给用户提供的栈增加一些负担，但是一个容器去访问另一个容器中存储的数据，这是很罕见的，并且当发生时会很明显，所以这不是一个特别沉重的负担。

在其他情况下，这就不会那么直接了，例如，3.2.4节中的交换操作。至少再这中情况下你可能同时锁住多个互斥量，但是有时不会发生。当你回看3.1节中那个列表连接例子时，将会看到列表中的每个节点都会有一个互斥量保护。然后，为了访问列表，线程必须获取他们感兴趣节点上的互斥锁。当一个线程删除一个节点，它必须获取三个节点上的互斥锁：将要删除的节点，两个邻接节点(因为他们也会被修改)。同样的，为了让列表畅通(连接起来)，线程必须保证在获取当前节点的互斥锁前提下，获得下一个节点的锁，要保证指向下一个节点的指针不会同时被修改。一旦下一个节点上的锁被获取，那么第一个节点的锁就可以释放了，因为已经没有必要在持有它了。

这种“手递手”(hand-over-hand)锁的模式允许多个线程访问列表，为每一个访问的线程提供不同的节点。但是，为了避免死锁，节点必须以同样的顺序上锁：如果两个线程试图用互为反向的顺序，使用“手递手”锁遍历列表，他们将执行到列表中间部分时，发生死锁。当节点A和B在列表中相邻，当前线程可能会同时尝试获取A和B上的锁。另一个线程可能已经获取了节点B上的锁，并且试图获取节点A上的锁——经典的死锁场景。

当A、C节点中的B节点正在被删除时，如果有线程在已获取A和C上的锁后，还要获取B节点上的锁时，当一个线程遍历列表的时候，这样的情况就可能发生死锁。这样的线程可能会试图首先锁住A节点或C节点(根据遍历的方向)，但是后面就会发现，它无法获得B上的锁，因为线程在执行删除任务的时候，已经获取了B上的锁，并且同时也获取了A和C上的锁。

这里提供一种避免死锁的方式，定义遍历的顺序，所以一个线程必须先锁住A才能获取B的锁，在锁住B之后才能获取C的锁。这将消除死锁发生的可能性，在不允许反向遍历的列表上。类似的约定常被用来建立其他的数据结构。

**使用锁的层次结构**

虽然，这对于定义锁的顺序，的确是一个特殊的情况，但锁的层次的意义在于提供对运行时约定是否被坚持的检查。这个建议需要对你的应用进行分层，并且识别在给定层上所有可上锁的互斥量。当代码试图对一个互斥量上锁，在该层锁已被低层持有时，上锁是不允许的。你可以在运行时对其进行检查，通过分配层数到每个互斥量上，以及记录被每个线程上锁的互斥量。下面的代码列表中将展示两个线程如何使用分层互斥。

列表3.7 使用层次锁来避免死锁
```c++
hierarchical_mutex high_level_mutex(10000); // 1
hierarchical_mutex low_level_mutex(5000);  // 2

int do_low_level_stuff();

int low_level_func()
{
  std::lock_guard<hierarchical_mutex> lk(low_level_mutex); // 3
  return do_low_level_stuff();
}

void high_level_stuff(int some_param);

void high_level_func()
{
  std::lock_guard<hierarchical_mutex> lk(high_level_mutex); // 4
  high_level_stuff(low_level_func()); // 5
}

void thread_a()  // 6
{
  high_level_func();
}

hierarchical_mutex other_mutex(100); // 7
void do_other_stuff();

void other_stuff()
{
  high_level_func();  // 8
  do_other_stuff();
}

void thread_b() // 9
{
  std::lock_guard<hierarchical_mutex> lk(other_mutex); // 10
  other_stuff();
}
```

thread_a() ⑥ 遵守规则，所以它运行的没问题。另一方面，thread_b() ⑨ 无视规则，因此在运行的时候肯定会失败。thread_a()调用high_level_func()，让high_level_mutex④上锁(其层级值为10000①)，在这互斥量上锁是为了获取high_level_stuff()的参数，之后调用low_level_func()⑤。low_level_func()会对low_level_mutex上锁，这就没有问题了，因为这个互斥量有一个低层值5000②。

thread_b()运行就不会顺利了。首先，它锁住了other_mutex⑩，这个互斥量的层级值只有100⑦。这就意味着，超低层级的数据(*ultra-low-level data*)已被保护。当other_stuff()调用high_level_func()⑧时，就违反了层级结构：high_level_func()试图获取high_level_mutex，这个互斥量的层级值是10000，要比当前层级值100大很多。因此hierarchical_mutex将会产生一个错误，可能会是抛出一个异常，或直接终止程序。在层级互斥量上产生死锁，是不可能的，因为互斥量本身会严格遵循约定顺序，进行上锁。这也意味，当多个互斥量在是在同一级上时，这你不能同时持有多个锁，所以“手递手”锁的方案需要每个互斥量在一条链上，并且每个互斥量都比其前一个有更低的层级值，这在某些情况下，就有些不切实际了。

例子也展示了另一点，`std::lock_guard<>`模板与用户定义的互斥量类型一起使用。hierarchical_mutex不是C++标准的一部分，但是它写起来很容易；一个简单的实现在列表3.8中展示出来。尽管它是一个用户定义类型，它可以用于`std::lock_guard<>`模板中，因为它的实现有三个成员函数为了满足互斥量操作：lock(), unlock() 和 try_lock()。虽然你还没见过try_lock()怎么使用，但是其使用起来很简单：当互斥量上的锁被一个线程持有，它将返回false，而不是等待调用的线程，直到能够获取互斥量上的锁为止。在`std::lock()`的内部实现中，try_lock()会作为避免死锁算法的一部分。

列表3.8 简单的层级互斥量实现
```c++
class hierarchical_mutex
{
  std::mutex internal_mutex;
  unsigned long const hierarchy_value;
  unsigned long previous_hierarchy_value;
  static thread_local unsigned long this_thread_hierarchy_value;  // 1
  void check_for_hierarchy_violation()
  {
    if(this_thread_hierarchy_value <= hierarchy_value)  // 2
    {
      throw std::logic_error(“mutex hierarchy violated”);
    }
  }
  void update_hierarchy_value()
  {
    previous_hierarchy_value=this_thread_hierarchy_value;  // 3
    this_thread_hierarchy_value=hierarchy_value;
  }
public:
  explicit hierarchical_mutex(unsigned long value):
      hierarchy_value(value),
      previous_hierarchy_value(0)
  {}
  void lock()
  {
    check_for_hierarchy_violation();
    internal_mutex.lock();  // 4
    update_hierarchy_value();  // 5
  }
  void unlock()
  {
    this_thread_hierarchy_value=previous_hierarchy_value;  // 6
    internal_mutex.unlock();
  }
  bool try_lock()
  {
    check_for_hierarchy_violation();
    if(!internal_mutex.try_lock())  // 7
      return false;
    update_hierarchy_value();
    return true;
  }
};
thread_local unsigned long
     hierarchical_mutex::this_thread_hierarchy_value(ULONG_MAX);  // 7
```

这里重点是使用了thread_local的值来代表当前线程的层级值：this_thread_hierarchy_value①。它被初始话为最大值⑧，所以最初所有线程都能被锁住。因为其声明中有thread_local，所以每个线程都有其拷贝副本，这样在线程中变量的状态就完全独立了，当从另一个线程进行读取时，变量的状态也是完全独立的。参见附录A，A.8节，有更多与thread_local相关的内容。

所以，第一次线程锁住一个hierarchical_mutex时，this_thread_hierarchy_value的值是ULONG_MAX。由于其本身的性质，这个值会大于其他任何值，所以会通过check_for_hierarchy_vilation()②的检查。在这种检查方式下，lock()代表内部互斥锁已被锁住④。一旦成功锁住，你可以更新层级值了⑤。

当你现在锁住另一个hierarchical_mutex时，还持有第一个锁，this_thread_hierarchy_value的值将会显示第一个互斥量的层级值。第二个互斥量的层级值必须小于已经持有互斥量检查函数②才能通过。

现在，最重要的是为当前线程存储之前的层级值，所以你可以调用unlock()⑥对层级值进行保存；否则，你就锁不住任何互斥量(第二个互斥量的层级数高于第一个互斥量)，即使线程没有持有任何锁。因为你保存了之前的层级值，只有当你持有internal_mutex③，并且在解锁内部互斥量⑥之前存储它的层级值，你才能安全的将hierarchical_mutex自身进行存储。这是因为hierarchical_mutex被内部互斥量的锁所保护着。

try_lock()与lock()的功能相似，除非在调用internal_mutex的try_lock()⑦失败时，然后你就不能持有对应锁了，所以不必更新层级值，并直接返回false就好。

虽然是运行时检测，但是它至少没有时间依赖性——你不必去等待那些导致死锁出现的罕见条件。同时，设计过程需要去拆分应用，互斥量在这样的情况下可以帮助消除很多可能导致死锁的情况。这是很值得去做的设计练习，即使你之后没有去做，代码也会在运行时进行检查。

**超越锁的延伸扩展**

如我在本节开头提到的那样，死锁不仅仅会发生在锁之间；死锁也会发生在任何同步构造中(可能会产生一个等待循环)。因此也需要有指导意见能覆盖到这个层面上。例如，正如你要去避免获取嵌套锁，等待一个持有锁的线程是一个很糟糕的决定，因为线程为了能继续运行可能需要获取对应的锁。类似的，如果去等待一个线程结束，它应该可以确定一个线程的层级，这样的话，一个线程只需要等待比起层级低的线程结束即可。可以用一个简单的办法去确定，以添加的线程是否在同一函数中被启动，如同在3.1.2节和3.3节中描述的那样。

当你已经将你的代码设计成避免死锁的，`std::lock()`和`std::lack_guard`能组成简单的锁覆盖大多数需要锁情况，但是有时需要更多的灵活性。在这些情况，可以使用标准库提供的`std::unique_lock`模板。如` std::lock_guard`，这是一个参数化的互斥量模板类，并且它提供很多RAII类型锁用来管理`std::lock_guard`类型，这样就有更加的灵活了。

###3.2.6 std::unique_lock——灵活的锁

`std::unqiue_lock`通过对不变量的放松(*by relaxing the invariants*)，会比`std:lock_guard`更加灵活；一个`std::unique_lock`实现不会总是拥有与互斥量相关的数据类型。首先，就像你能将`std::adopt_lock`作为第二个参数传入到构造函数，对互斥所进行管理，你也可以把`std::defer_lock`作为第二个参数传递进去，为了表明互斥量在结构上应该保持解锁状态。这样，就可以被后面调用lock()函数的`std::unique_lock`对象（不是互斥量）所获取，或传递`std::unique_lock`对象本身到`std::lock()`中。清单3.6可以很容易被改写为清代3.9中中的代码，使用`std::unique_lock`和`std::defer_lock`①，而非`std::lock_guard`和`std::adopt_lock`。代码长度相同，且几乎等价，唯一不同的就是：`std::unique_lock`会占用比较多的空间，并且比`std::lock_guard`运行的稍慢一些。保证灵活性是要付出代价的，这个代价就允许`std::unique_lock`实例不携带互斥量：该信息已被存储，且已被更新。

清单3.9 在交换操作中使用`std::lock()`和`std::unique_lock`
```c++
class some_big_object;
void swap(some_big_object& lhs,some_big_object& rhs);
class X
{
private:
  some_big_object some_detail;
  std::mutex m;
public:
  X(some_big_object const& sd):some_detail(sd){}
  friend void swap(X& lhs, X& rhs)
  {
    if(&lhs==&rhs)
      return;
    std::unique_lock<std::mutex> lock_a(lhs.m,std::defer_lock); // 1 
    std::unique_lock<std::mutex> lock_b(rhs.m,std::defer_lock); // 1 std::def_lock 留下未上锁的互斥量
    std::lock(lock_a,lock_b); // 2 互斥量在这里上锁
    swap(lhs.some_detail,rhs.some_detail);
  }
};
```

在列表3.9中，将`std::unique_lock`对象传递到`std::lock()`②，这是因为`std::unique_lock`支持lock(), try_lock()和unlock()成员函数。这些同名的成员函数在互斥低层做着实际的工作，并且仅更新`std::unique_lock`实例中的标识，来确定该实例是否拥有特定的互斥量。这个标志确保unlock()在析构函数中被正确调用。如果实例拥有互斥量，那么析构函数必须调用unlock()；但当实例中没有互斥量时，析构函数就不能去调用unlock()。这个标志可以通过owns_lock()成员变量进行查询。

可能如你期望的那样，这个标志是被存储在某个地方的。因此，`std::unique_lock`对象的体积通常要比`std::lock_guard`对象大；当使用`std::unique_lock`替代`std::lock_guard`，因为会对标志进行适当的更新或检查，就会些轻微的性能惩罚。当`std::lock_guard`已经能够满足你的需求，那么我还是建议你继续使用它。当在需要更加灵活的锁时，你就最好选择`std::unique_lock`，因为它更适合于你的任务。你已经看到一个递延锁的例子；另外一种情况是锁的所有权需要从一个域转到另一个。

###3.2.7 在不同域中传递互斥量所有权

因为`std::unique_lock`实例没有与自身相关的互斥量，一个互斥量的所有权可以通过移动操作，在不同的实例中进行传递。在某些情况下，这种转移是自动发生的，例如，当函数返回一个实例；另些情况下，你需要显式的调用`std::move()`来执行移动操作。从本质上来说，者需要依赖于源值是否是左值(*lvalue*) ——一个实际的值或是引用——或一个右值(*rvalue*)——一个临时类型。当源值是一个右值，就必须显式移动成左值，为了避免转移所有权过程出错。`std::unique_lock`是一个可移动，但不可赋值的类型。参考附录A，A.1.1节，有更多与移动语句相关的信息。

一种使用可能是允许一个函数去锁住一个互斥量，并且将转移所有权移到调用者上，所以调用者可以在同一个锁保护的范围内，执行额外的动作。下面的程序片段展示了：函数get_lock()锁住了互斥量，然后准备数据，返回锁的调用函数：

```c++
std::unique_lock<std::mutex> get_lock()
{
  extern std::mutex some_mutex;
  std::unique_lock<std::mutex> lk(some_mutex);
  prepare_data();
  return lk;  // 1
}
void process_data()
{
  std::unique_lock<std::mutex> lk(get_lock());  // 2
  do_something();
}
```

因为lk在函数中被声明为自动变量，它不需要调用`std::move()`，可以直接返回①；编译器负责调用移动构造函数。process_data()函数直接转移`std::unique_lock`实例的所有权②，调用do_something()可依赖与已经准备好的正确数据(数据没有受到其他线程的修改)。

通常这种模式会用于已锁的互斥量，其依赖于当前程序的状态，或依赖于传入返回类型为`std::unique_lock`的函数的一个参数。这样的用法不会直接返回锁，不过网关类的一个数据成员可用来确认已经对保护数据的访问权限进行上锁。这种情况下，所有的访问都必须通过网关类：当你想要访问数据，需要获取网关类的实例(如同前面的例子，通过调用如同get_lock()的函数)来获取锁。之后你就可以通过网关类的成员函数对数据进行访问。当你完成访问，你可以销毁这个网关类对象，这样就可以释放锁，让别的线程来访问保护数据。这样的一个网关类可能是可移动的(所以他可以从一个函数进行返回)，在这种情况下锁对象的数据必须是可移动的。

`std::unique_lock`的灵活性同样也允许实例在销毁之前放弃其拥有的锁。你可以使用unlock()来做这件事，就如同一个互斥量：`std::unique_lock`的成员函数提供类似于锁定和解锁互斥量的功能。`std::unique_lock`实例有在销毁前释放锁的能力，这就意味着，当锁没有必要在持有的时候，你可以在特定的代码分支对其进行选择性的释放。这对于应用性能来说很重要；持有锁的时间增加会导致性能下降，因为其他线程会等待这个锁的释放，避免超越必要的程序。

###3.2.8 锁的粒度

在3.2.3节中，我们已经对锁的粒度有所了解：锁的粒度是一个“摆手术语”(*hand-waving term*)用来描述通过一个锁保护着的数据量。一个细粒度锁(*a fine-grained lock*)能够保护较小的数据量，一个粗粒度锁(*a coarse-grained lock*)能够保护较多的数据量。选择一个粒度够粗的锁是很重要的，为了确保要求的数据被保护，而且保证锁适合它需要的操作也是很重要的。我们都知道，在超市等待结账的时候，正在结账的顾客突然意识到他忘记拿蔓越莓酱了，然后离开柜台去拿，并让其他的人都等待他回来，或者，当收银员，准备收钱时，顾客才去翻钱包拿钱，这样的情况都会让等待的顾客很无奈。当每个人都检查了自己要拿的东西，且能随时为拿到的商品进行支付，那么的每件事都进行顺利。

这样的道理同样适用于线程：如果很多线程正在等待同一个资源(等待收银员对自己拿到的商品进行清点)，在当有线程持有锁的时间过长，这就会增加等待的时间(别等到结账的时候，才想起来蔓越莓酱没拿)。在可能的情况下，锁住互斥量的同时只能对共享数据进行访问；试图对锁外数据进行处理。特别是，别做一些费时的动作，比如：对文件的输入/输(*file I/O*)操作进行上锁。文件输入/输出通常要比从内存中读或写同样长度的数据慢成百上千倍(这里有些夸张)。所以，除非锁已经打算去保护对文件的访问，执行I/O操作将会将延迟其他线程执行的时间，这很没有必要(因为他们被这个文件锁阻塞住了)，这样多线程带来的性能提高就会被抵消。

`std::unique_lock`在这种情况下工作正常，因为当代码不需要再访问共享数据时，你可以调用unlock()；而后当再次需要对共享数据进行访问时，就可以再调用lock()了。下面代码就是这样的一种情况：

```c++
void get_and_process_data()
{
  std::unique_lock<std::mutex> my_lock(the_mutex);
  some_class data_to_process=get_next_data_chunk();
  my_lock.unlock();  // 1 不要让锁住的互斥量越过process()函数的调用
  result_type result=process(data_to_process);
  my_lock.lock(); // 2 为了写入数据，对互斥量再次上锁
  write_result(data_to_process,result);
}
```

你不需要让锁住的互斥量越过process()函数的调用，所以你可以在函数调用①前对互斥量手动解锁，并且在之后对其再次上锁②。

希望这能表示只有一个互斥量保护整个数据结构时的情况，不仅可能会有更多对锁的竞争，也会增加锁持锁的时间。较多的操作步骤需要获取同一个互斥量上的锁，所以持有锁的时间会更长。成本上上的双重打击也算是为向细粒度锁转移提供了双重激励和可能性。

如同上面的例子，锁不仅是能锁住合适粒度的数据；而且还要控制锁的持有时间，以及什么操作在执行的同时能够拥有锁。一般情况下，执行必要的操作时，尽可能将持有锁的时间缩减到最小(*In general, a lock should be held for only the minimum possible time needed to perform the required operations.*)。这也就意味这那些浪费时间操作，比如：获取另外一个锁(即使你知道这不会造成死锁)，或等待输入/输出操作完成时没有必要持有一个锁，除非绝对需要。

在清单3.6和3.9中，交换操作需要锁住两个互斥量，其明确的要求并发访问两个对象。假设你用来做比较的是一个简单的数据类型，比如int类型。将会有什么不同么？int的拷贝很廉价，所以你可以很容的进行数据复制，并且每个被比较的对象都持有该对象的锁，在比较之后进行数据拷贝。这就意味这你在最短时间内持有每个互斥量，并且你不会在持有一个锁的同时再去获取另一个。下面的清单中展示了一个在这样情景中的Y类，并且展示了一个相等比较运算符的等价实现。

列表3.10 在比较操作符中一次锁住一个互斥量
```c++
class Y
{
private:
  int some_detail;
  mutable std::mutex m;
  int get_detail() const
  {
    std::lock_guard<std::mutex> lock_a(m); // 1
    return some_detail;
  }
public:
  Y(int sd):some_detail(sd){}

  friend bool operator==(Y const& lhs, Y const& rhs)
  {
    if(&lhs==&rhs)
      return true;
    int const lhs_value=lhs.get_detail();  // 2
    int const rhs_value=rhs.get_detail();  // 3
    return lhs_value==rhs_value;  // 4
  }
};
```

在这个例子中，比较操作符首先通过调用get_detail()成员函数检索要比较的值②③。函数在索引值时被一个锁保护着①。比较操作符会在之后比较索引出来的值④。注意：虽然这样能减少锁持有的时间，一个锁只持有一次(这样能消除死锁的可能性)，这里有一个微妙的语义操作(*this has subtly changed the semantics of the operation*)同时对两个锁住的值进行比较。在列表3.10中，当操作符返回true时，那就意味着在这个时间点上的lhs.some_detail与在另一个时间点的rhs.some_detail相同。这两个值在读取之后，可能会被任意的方式所修改；两个值会在②和③出进行交换，这样的话，会失去比较的意义。等价比较可能会返回true，来表明这两个值时相等的，实际上这两个值相等的情况可能就发生在一瞬间。这样的变化要小心，语义操作是无法改变一个问题的比较方式的：当你持有锁的时间没有达到整个操作时间时，你就会让自己处于条件竞争的状态(*if you don’t hold the required locks for the entire duration of an operation, you’re exposing yourself to race conditions.*)。

有时，只是没有一个合适粒度级别，因为并不是所有对数据结构的访问都需要同一级的保护。在这个例子中，就需要寻找一个合适的机制，去替换`std::mutex`。

##3.3 保护共享数据的替代设施

虽然互斥量是最通用的机制，但其并非保护共享数据的唯一方式；这里有很多替代方式可以在特定情况下，提供更加合适的保护。

一个特别极端(但十分常见)的情况就是，共享数据在并发访问时和初始化的时候，都需要保护，但是之后需要进行隐式同步。这可能是因为数据是作为只读方式创建的，所以没有同步问题；或者可能因为必要的保护作为对数据操作的一部分，隐式的执行了。在任何情况下，在数据初始化后锁住一个互斥量，纯粹是为了保护其初始化过程，这是没有必要的，并且这会给性能带来不必要的冲击。出于以上的原因，C++标准提供了一种纯粹保护共享数据初始化过程的机制。

###3.3.1 保护共享数据的初始化过程

假设你与一个共享源，构建代价很昂贵；可能它会打开一个数据库连接或分配出很多的内存。延迟初始化(*Lazy initialization*)在单线程代码很常见——每一个操作都需要先对源进行检查，为了了解数据是否被初始化，然后在其使用前决定，数据是否需要初始化：

```c++
std::shared_ptr<some_resource> resource_ptr;
void foo()
{
  if(!resource_ptr)
  {
    resource_ptr.reset(new some_resource);  // 1
  }
  resource_ptr->do_something();
}
```

当共享数据对于并发访问是安全的，①是转为多线程代码时，需要保护的，但是下面天真的转换(*naïve【法语】：天真的*)会使得线程资源产生不必要的序列化。这是因为每个线程必须等待互斥量，为了确定数据源已经初始化了。

清单 3.11 使用一个互斥量的延迟初始化(线程安全)过程
```c++
std::shared_ptr<some_resource> resource_ptr;
std::mutex resource_mutex;
void foo()
{
  std::unique_lock<std::mutex> lk(resource_mutex);  // 所有线程在此序列化 
  if(!resource_ptr)
  {
    resource_ptr.reset(new some_resource);  // 只有初始化过程需要保护 
  }
  lk.unlock();
  resource_ptr->do_something();
}
```

这段代码相当常见了，也足够表现出没必要的线程话问题，很多人能想出更好的一些的办法来做这件事，包括声名狼藉的双重检查锁(*Double-Checked Locking*)模式：

```c++
void undefined_behaviour_with_double_checked_locking()
{
  if(!resource_ptr)  // 1
  {
    std::lock_guard<std::mutex> lk(resource_mutex);
    if(!resource_ptr)  // 2
    {
      resource_ptr.reset(new some_resource);  // 3
    }
  }
  resource_ptr->do_something();  // 4
}
```

指针第一次读取数据不需要获取锁①，并且只有在指针为NULL时才需要获取锁。然后，当获取锁之后，指针会被再次检查一遍② (这就是双重检查的部分)，避免另一的线程在第一次检查后再做初始化，并且让当前线程获取锁。

这个模式为什么声名狼藉呢？因为这里有潜在的条件竞争，因为外部的读取锁①没有与内部的写入锁进行同步③。因此就会产生条件竞争，这个条件竞争不仅覆盖指针本身，还会影响到其指向的对象；即使一个线程知道另一个线程完成对指针进行写入，它可能没有看到新创建的some_resource实例，然后调用do_something()④后，得到不正确的结果。这个例子是在一种典型的条件竞争——数据竞争，C++标准中这就会被指定为“未定义行为”(*underfined behavior*)。这种竞争肯定是可以避免的。可以阅读第5章，那里有更多对内存模型的讨论，包括数据竞争(*data race*)的构成。

C++标准委员会也认为条件竞争的处理很重要，所以C++标准库提供了`std::once_flag`和`std::call_once`来处理这种情况。比起锁住互斥量，并显式的检查指针，每个线程只需要使用`std::call_once`，在`std::call_once`的结束时，就能安全的知道指针已经被其他的线程初始化了。使用`std::call_once`比显式使用互斥量消耗的资源更少，特别是当初始化完成后。下面的例子展示了与清单3.11中的同样的操作，这里使用了`std::call_once`。在这种情况下，初始化通过调用函数完成，同样这样操作使用类中的函数操作符来实现同样很简单。如同大多数在标准库中的函数一样，或作为函数被调用，或作为参数被传递，`std::call_once`可以和任何函数或可调用对象一起使用。

```c++
std::shared_ptr<some_resource> resource_ptr;
std::once_flag resource_flag;  // 1

void init_resource()
{
  resource_ptr.reset(new some_resource);
}

void foo()
{
  std::call_once(resource_flag,init_resource);  // 可以完整的进行一次初始化
  resource_ptr->do_something();
}
```

在这个例子中，`std::once_flag`①和初始化好的数据都是命名空间区域的对象，但是`std::call_once()`可仅作为延迟初始化的类型成员，如同下面的例子一样：

清单3.12 使用`std::call_once`作为类成员的延迟初始化(线程安全)

```c++
class X
{
private:
  connection_info connection_details;
  connection_handle connection;
  std::once_flag connection_init_flag;

  void open_connection()
  {
    connection=connection_manager.open(connection_details);
  }
public:
  X(connection_info const& connection_details_):
      connection_details(connection_details_)
  {}
  void send_data(data_packet const& data)  // 1
  {
    std::call_once(connection_init_flag,&X::open_connection,this);  // 2
    connection.send_data(data);
  }
  data_packet receive_data()  // 3
  {
    std::call_once(connection_init_flag,&X::open_connection,this);  // 2
    return connection.receive_data();
  }
};
```

在这个例子中，第一个调用send_data()①或receive_data()③的线程完成初始化过程。使用成员函数open_connection()去初始化数据，也需要将this指针传进去。和其在在标准库中的函数一样，其接受可调用对象，比如`std::thread`的构造函数和`std::bind()`，通过向`std::call_once()`②传递一个额外的参数来完成这个操作。

值得注意的是，`std::mutex`和`std::one_flag`的实例就不能拷贝和移动，所以当你使用它们作为类成员函数，如果你需要用到他们，你就得显示定义这些特殊的成员函数。

还有一种情形的初始化过程中潜存着条件竞争：其中一个局部变量被声明为static类型。这种变量的在声明后就已经完成初始化；对于多线程调用的函数，这就意味着这里有条件竞争——抢着去定义这个变量。在很多在前C++11编译器(译者：不支持C++11标准的编译器)，在实践过程中，这样的条件竞争是确实存在的，因为在多线程中，每个线程都认为他们是第一个初始化这个变量线程；或一个线程对变量进行初始化，而另外一个线程要使用这个变量时，初始化过程还没完成。在C++11标准中，这些问题都被解决了：初始化及定义完全在一个线程中发生，并且没有其他线程可在初始化完成前对其进行处理，条件竞争终止于初始化阶段，这样比在之后再去处理好的多。在只需要一个全局实例情况下，这里提供一个`std::call_once`的替代方案

```c++
class my_class;
my_class& get_my_class_instance()
{
  static my_class instance;  // 线程安全的初始化过程
  return instance;
}
```

多线程可以安全的调用get_my_class_instance()①函数，不用为数据竞争而担心。

只在初始化时保护数据，对于更一般的情况来说就是一个特例：对于很少有更新的数据结构来说。在大多数情况下，这种数据结构是只读的，并且多线程对其并发的读取也是很愉快的，不过一旦数据结构需要更新，那么就会有条件竞争的产生。必须承认，这里的确也需要一种保护机制。

###3.3.2 保护很少更新的数据结构

试想，为了将域名解析为其相关IP地址，我们在缓存(*cache*)中的存放了一张DNS入口表。通常情况下，一个给定的DNS条目将会在很长的一段时间内保持不变——在很多情况下，很多DNS入口会有很多年保持不变。虽然，新的入口可能会随着时间的推移，在用户访问不同网站时，被添加到表中，但是这些数据很有可能在其生命周期内保持不变。所以定期检查缓存中入口的有效性，就变的十分重要了；但是这也需要一次更新，也许这次更新只是对一些细节做了改动。

虽然更新频度很低，但更新也是有可能发生的，并且当这个可缓存被多个线程访问，这个缓存就需要适当的保护措施，来对其处于更新状态时进行保护，也为了确保线程读到缓存中的有效数据。

在没有使用专用数据结构的情况下，这种方式是符合预期，并且为并发更新和读取特别设计的(更多的例子在第6和第7章中介绍)。这样的更新要求更新线程独占数据结构的访问权，直到其完成更新操作。当改变完成，数据结构对于并发多线程访问又会是安全的。使用一个`std::mutex`来保护数据结构，这的确有些反应过度，因为在没有发生修改时，它将削减并发读取数据的可能性；这里需要另一种不同的互斥量。这种新的互斥量常被称为“读者-写者锁”（*reader-writer mutex*），因为其允许两中不同的使用方式：一个“作者”线程独占访问和共享访问，让多个“读者”线程并发访问。

新的C++标准库应该不提供这样的互斥量，虽然已经提交给了标准委员会[3]。因为建议没有被采纳，这个例子在本节中使用的实现是Boost库提供的，其采纳了这个建议。你将在第8中看到，这种锁的也不能包治百病，其性能依赖与参与其中的处理器数量，同业也与读者和更新者线程的负载有关。为了确保增加复杂度后还能得到受益，在目标系统上的代码性能就很重要了。

比起使用`std::mutex`实例进行同步，不如使用`boost::shared_mutex`来做同步。对于更新操作，可以使用`std::lock_guard<boost::shared_mutex>`和`std::unique_lock<boost::shared_mutex>`进行上锁，作为`std::mutex`的替代方案。与`std::mutex`所做的一样，这就能保证更新线程的独占访问。其他线程不需要去修改数据结构，其实现可以使用`boost::shared_lock<boost::shared_mutex>`获取共享(*share*)访问权。这与使用`std::unique_lock`一样，除非多线要在同时得到同一个`boost::shared_mutex`上有共享锁。唯一的限制就是，当任一线程拥有一个共享锁时，这个线程就会尝试获取一个独占锁，直到其他线程放弃他们的锁；同样的，当任一线程拥有一个独占锁是，其他线程就无法获得共享锁或独占锁，直到第一个线程放弃其拥有的锁。

下面的代码清单展示了一个简单的DNS缓存，如同之前描述的那样，使用`std::map`持有缓存数据，使用`boost::shared_mutex`进行保护。

清单3.13 使用`boost::shared_mutex`对数据结构进行保护
```c++
#include <map>
#include <string>
#include <mutex>
#include <boost/thread/shared_mutex.hpp>

class dns_entry;

class dns_cache
{
  std::map<std::string,dns_entry> entries;
  mutable boost::shared_mutex entry_mutex;
public:
  dns_entry find_entry(std::string const& domain) const
  {
    boost::shared_lock<boost::shared_mutex> lk(entry_mutex);  // 1
    std::map<std::string,dns_entry>::const_iterator const it=
       entries.find(domain);
    return (it==entries.end())?dns_entry():it->second;
  }
  void update_or_add_entry(std::string const& domain,
                           dns_entry const& dns_details)
  {
    std::lock_guard<boost::shared_mutex> lk(entry_mutex);  // 2
    entries[domain]=dns_details;
  }
};
```

在清代3.13中，find_entry()使用了`boost::shared_lock<>`实例来保护其共享和只读权限①；这就使得，多线程可以同时调用find_entry()，且不会出错。另一方面，update_or_add_entry()使用`std::lock_guard<>`实例，当表格需要更新时②，为其提供独占访问权限；在update_or_add_entry()函数调用时，独占锁会阻止其他线程对数据结构进行修改，并且这些线程在这时，也不能调用find_entry()。

***
[1] Tom Cargill, “Exception Handling: A False Sense of Security,” in C++ Report 6, no. 9 (November–December 1994). Also available at http://www.informit.com/content/images/020163371x/supplements/Exception_Handling_Article.html.

[2] Herb Sutter, Exceptional C++: 47 Engineering Puzzles, Programming Problems, and Solutions (Addison Wesley Pro-fessional, 1999).

[3] Howard E. Hinnant, “Multithreading API for C++0X—A Layered Approach,” C++ Standards Committee Paper N2094, http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2006/n2094.html.














