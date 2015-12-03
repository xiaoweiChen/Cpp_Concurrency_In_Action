#第2章 线程管理

**本章主要内容**

- 启动新线程<br>
- 等待线程与分离线程<br>
- 线程唯一标识符<br>

好的！看来你已经决定使用多线程了。先做点什么呢?启动线程、结束线程，还是如何监管线程？在C++标准库中只需要管理`std::thread`关联的线程，无需把注意力放在其他方面。不过，标准库太灵活，所以管理起来不会太容易。

本章将从基本开始：启动一个线程，等待这个线程结束，或放在后台运行。再看看怎么给已经启动的线程函数传递参数，以及怎么将一个线程的所有权从当前`std::thread`对象移交给另一个。最后，再来确定线程数，以及识别特殊线程。

##2.1 线程管理的基础

每个程序至少有一个线程：执行main()函数的线程，其余线程有其各自的入口函数。线程与原始线程(以main()为入口函数的线程)同时运行。如同main()函数执行退出一样，当线程执行完入口函数后，线程也会退出。在为一个线程创建了一个`std::thread`对象后，需要等待这个线程结束；不过，线程需要先进行启动。下面就来启动线程。

###2.1.1 启动线程

第1章中，线程在`std::thread`对象创建(为线程指定任务)时启动。最简单的情况下，任务也会很简单，通常是无参数无返回(*void-returning*)的函数。这种函数在其所属线程上运行，直到函数执行完毕，线程也就结束了。在一些极端情况下，线程运行时，任务中的函数对象需要通过某种通讯机制进行参数的传递，或者执行一系列独立操作;可以通过通讯机制传递信号，让线程停止。线程要做什么，以及什么时候启动，其实都无关紧要。总之，使用C++线程库启动线程，可以归结为构造`std::thread`对象：

```c++
void do_some_work();
std::thread my_thread(do_some_work);
```

为了让编译器识别`std::thread`类，这个简单的例子也要包含`<thread>`头文件。如同大多数C++标准库一样，`std::thread`可以用可调用（*callable*）类型构造，将带有函数调用符类型的实例传入`std::thread`类中，替换默认的构造函数。

```c++
class background_task
{
public:
  void operator()() const
  {
    do_something();
    do_something_else();
  }
};

background_task f;
std::thread my_thread(f);
```

代码中，提供的函数对象会复制到新线程的存储空间当中，函数对象的执行和调用都在线程的内存空间中进行。函数对象的副本与原始函数对象的行为是相同的，但函数对象副本得到的结果，有时却与我们的期望不同。

有件事需要注意，当把函数对象传入到线程构造函数中时，需要避免“[最令人头痛的语法解析](http://en.wikipedia.org/wiki/Most_vexing_parse)”(*C++’s most vexing parse*, [中文简介](http://qiezhuifeng.diandian.com/post/2012-08-27/40038339477))。如果你传递了一个临时变量，而不是一个命名的变量；C++编译器会将其解析为函数声明，而不是类型对象的定义。

例如：
```c++
std::thread my_thread(background_task());
```

这里相当与声明了一个名为my_thread的函数，这个函数带有一个参数(函数指针指向没有参数并返回background_task对象的函数)，返回一个`std::thread`对象的函数，而非启动了一个线程。

使用在前面命名函数对象的方式，或使用多组括号①，或使用新统一的初始化语法②，可以避免这个问题。

如下所示：

```c++
std::thread my_thread((background_task()));  // 1
std::thread my_thread{background_task()};    // 2
```

使用lambda表达式也能避免这个问题。lambda表达式是C++11的一个新特性，它允许使用一个可以捕获局部变量的局部函数(可以避免传递参数，参见2.2节)。想要具体的了解lambda表达式，可以阅读附录A的A.5节。之前的例子可以改写为lambda表达式的类型：

```c++
std::thread my_thread([]{
  do_something();
  do_something_else();
});
```

启动了线程，你需要明确是要等待线程结束(*加入式*——参见2.1.2节)，还是让其自主运行(*分离式*——参见2.1.3节)。如果`std::thread`对象销毁之前还没有做出决定，程序就会终止(`std::thread`的析构函数会调用`std::terminate()`)。因此，即便是有异常存在，也需要确保线程能够正确的加入(*joined*)或分离(*detached*)。2.1.3节中，会介绍对应的方法来处理这两种情况。需要注意的是，必须在`std::thread`对象销毁之前做出决定——加入或分离线程之前。如果线程就已经结束，想再去分离它，线程可能会在`std::thread`对象销毁之后继续运行下去。

如果不等待线程，就必须保证线程结束之前，可访问的数据得有效性。这不是一个新问题——单线程代码中，对象销毁之后再去访问，也会产生未定义行为——不过，线程的生命周期增加了这个问题发生的几率。

这种情况很可能发生在线程还没结束，函数已经退出的时候，这时线程函数还持有函数局部变量的指针或引用。下面的清单中就展示了这样的一种情况。

清单2.1  函数已经结束，线程依旧访问局部变量
```c++
struct func
{
  int& i;
  func(int& i_) : i(i_) {}
  void operator() ()
  {
    for (unsigned j=0 ; j<1000000 ; ++j)
    {
      do_something(i);           // 1. 潜在访问隐患：悬空引用
    }
  }
};

void oops()
{
  int some_local_state=0;
  func my_func(some_local_state);
  std::thread my_thread(my_func);
  my_thread.detach();          // 2. 不等待线程结束
}                              // 3. 新线程可能还在运行
```

这个例子中，已经决定不等待线程结束(使用了detach()②)，所以当oops()函数执行完成时③，新线程中的函数可能还在运行。如果线程还在运行，它就会去调用do_something(i)函数①，这时就会访问已经销毁的变量。如同一个单线程程序——允许在函数完成后继续持有局部变量的指针或引用；当然，这从来就不是一个好主意——这种情况发生时，错误并不明显，会使多线程更容易出错。

处理这种情况的常规方法：使线程函数的功能齐全，将数据复制到线程中，而非复制到共享数据中。如果使用一个可调用的对象作为线程函数，这个对象就会复制到线程中，而后原始对象就会立即销毁。但对于对象中包含的指针和引用还需谨慎，例如清单2.1所示。使用一个能访问局部变量的函数去创建线程是一个糟糕的主意(除非**十分确定**线程会在函数完成前结束)。此外，可以通过加入的方式来确保线程在函数完成前结束。

###2.1.2 等待线程完成

如果需要等待线程，相关的`std::tread`实例需要使用**join()**。清单2.1中，将`my_thread.detach()`替换为`my_thread.join()`，就可以确保局部变量在线程完成后，才被销毁。在这种情况下，因为原始线程在其生命周期中并没有做什么事，使得用一个独立的线程去执行函数变得收益甚微，但在实际编程中，原始线程要么有自己的工作要做；要么会启动多个子线程来做一些有用的工作，并等待这些线程结束。

**join()**是简单粗暴的等待线程完成或不等待。当你需要对等待中的线程有更灵活的控制时，比如，看一下某个线程是否结束，或者只等待一段时间(超过时间就判定为**超时**)。想要做到这些，你需要使用其他机制来完成，比如条件变量和期待(*futures*)，相关的讨论将会在第4章继续。调用**join()**的行为，还清理了线程相关的存储部分，这样`std::thread`对象将不再与已经完成的线程有任何关联。这意味着，只能对一个线程使用一次**join()**;一旦已经使用过**join()**，`std::thread`对象就不能再次加入了，当对其使用**joinable()**时，将返回**否**（*false*）。

###2.1.3 特殊情况下的等待

如前所述，需要对一个还未销毁的`std::thread`对象使用**join()**或**detach()**。如果想要分离一个线程，可以在线程启动后，直接使用**detach()**进行分离。如果打算等待对应线程，则需要细心挑选调用**join()**的位置。当在线程运行之后产生异常，在join()调用之前抛出，就意味着很这次调用会被跳过。

避免应用被抛出的异常所终止，就需要作出一个决定。通常，当倾向于在无异常的情况下使用**join()**时，需要在异常处理过程中调用**join()**，从而避免生命周期的问题。下面的程序清单是一个例子。

清单 2.2 等待线程完成
```c++
struct func; // 定义在清单2.1中
void f()
{
  int some_local_state=0;
  func my_func(some_local_state);
  std::thread t(my_func);
  try
  {
    do_something_in_current_thread();
  }
  catch(...)
  {
    t.join();  // 1
    throw;
  }
  t.join();  // 2
}
```

清单2.2中的代码使用了`try/catch`块确保访问本地状态的线程退出后，函数才结束。当函数正常退出时，会执行到②处；当函数执行过程中抛出异常，程序会执行到①处。`try/catch`块能轻易的捕获轻量级错误，所以这种情况，并非放之四海而皆准。如需确保线程在函数之前结束——查看是否因为线程函数使用了局部变量的引用，以及其他原因——而后再确定一下程序可能会退出的途径，无论正常与否，可以提供一个简洁的机制，来做解决这个问题。

一种方式是使用“资源获取即初始化方式”(RAII，Resource Acquisition Is Initialization)，并且提供一个类，在析构函数中使用**join()**，如同下面清单中的代码。看它如何简化f()函数。

清单 2.3 使用RAII等待线程完成
```c++
class thread_guard
{
  std::thread& t;
public:
  explicit thread_guard(std::thread& t_):
    t(t_)
  {}
  ~thread_guard()
  {
    if(t.joinable()) // 1
    {
      t.join();      // 2
    }
  }
  thread_guard(thread_guard const&)=delete;   // 3
  thread_guard& operator=(thread_guard const&)=delete;
};

struct func; // 定义在清单2.1中

void f()
{
  int some_local_state=0;
  func my_func(some_local_state);
  std::thread t(my_func);
  thread_guard g(t);
  do_something_in_current_thread();
}    // 4
```

当线程执行到④处时，局部对象就要被逆序销毁了。因此，**thread_guard**对象**g**是第一个被销毁的，这时线程在析构函数中被加入②到原始线程中。即使**do_something_in_current_thread**抛出一个异常，这个销毁依旧会发生。

在**thread_guard**的析构函数的测试中，首先判断线程是否已加入①，如果没有会调用**join()**②进行加入。这很重要，因为**join()**只能对给定的对象调用一次，所以对给已加入的线程再次进行加入操作时，将会导致错误。

拷贝构造函数和拷贝赋值操作被标记为`=delete`③，是为了不让编译器自动生成它们。直接对一个对象进行拷贝或赋值是危险的，因为这可能会弄丢已经加入的线程。通过删除声明，任何尝试给**thread_guard**对象赋值的操作都会引发一个编译错误。想要了解删除函数的更多知识，请参阅附录A的A.2节。

如果不想等待线程结束，可以分离(*detaching*)线程，从而避免异常安全(*exception-safety*)问题。不过，这就打破了线程与`std::thread`对象的联系，即使线程仍然在后台运行着，分离操作也能确保`std::terminate()`在`std::thread`对象销毁才被调用。

###2.1.4 后台运行线程

使用detach()会让线程在后台运行，这就意味着主线程不能与之产生直接交互。也就是说，不会等待这个线程结束；如果线程分离，那么就不可能有`std::thread`对象能引用它，分离线程的确在后台运行，所以分离线程不能被加入。不过C++运行库保证，当线程退出时，相关资源的能够正确回收，后台线程的归属和控制C++运行库都会处理。

通常称分离线程为守护线程(*daemon threads*),UNIX中守护线程是指，且没有任何用户接口，并在后台运行的线程。这种线程的特点就是长时间运行；线程的生命周期可能会从某一个应用起始到结束，可能会在后台监视文件系统，还有可能对缓存进行清理，亦或对数据结构进行优化。另一方面，分离线程的另一方面只能确定线程什么时候结束，"发后即忘"(*fire and forget*)的任务就使用到线程的这种方式。

如2.1.2节所示，调用`std::thread`成员函数1detach()来分离一个线程。之后，相应的`std::thread`对象就与实际执行的线程无关了，并且这个线程也无法进行加入：

```c++
std::thread t(do_background_work);
t.detach();
assert(!t.joinable());
```

为了从`std::thread`对象中分离线程(前提是有可进行分离的线程):不能对没有执行线程的`std::thread`对象使用detach(),也是join()的使用条件，并且要用同样的方式进行检查——当`std::thread`对象使用t.joinable()返回的是true，就可以使用t.detach()。

试想如何能让一个文字处理应用同时编辑多个文档。无论是用户界面，还是在内部应用内部进行，都有很多的解决方法。虽然，这些窗口看起来是完全独立的，每个窗口都有自己独立的菜单选项，但他们却运行在同一个应用实例中。一种内部处理方式是，让每个文档处理窗口拥有自己的线程；每个线程运行同样的的代码，并隔离不同窗口处理的数据。如此这般，打开一个文档就要启动一个新线程。因为是对独立的文档进行操作，所以没有必要等待其他线程完成。因此，这里就可以让文档处理窗口运行在分离的线程上。

下面代码简要的展示了这种方法：

清单2.4 使用分离线程去处理其他文档
```c++
void edit_document(std::string const& filename)
{
  open_document_and_display_gui(filename);
  while(!done_editing())
  {
    user_command cmd=get_user_input();
    if(cmd.type==open_new_document)
    {
      std::string const new_name=get_filename_from_user();
      std::thread t(edit_document,new_name);  // 1
      t.detach();  // 2
    }
    else
    {
       process_user_input(cmd);
    }
  }
}
```

如果用户选择打开一个新文档，为了让迅速打开文档，需要启动一个新线程去打开新文档①，并分离线程②。与当前线程做出的操作一样，新线程只不过是打开另一个文件而已。所以，edit_document函数可以复用，通过传参的形式打开新的文件。

这个例子也展示了传参启动线程的方法：不仅可以向`std::thread`构造函数①传递函数名，还可以传递函数所需的参数(实参)。当然，也有其他方法完成这项功能，比如:使用一个带有数据成员的成员函数，代替一个需要传参的普通函数。C++线程库的方式也不是很复杂。

##2.2 向线程函数传递参数

清单2.4中，向`std::thread`构造函数中的可调用对象，或函数传递一个参数很简单。需要注意的是，默认参数要拷贝到线程独立内存中，即使参数是引用的形式，也可以在新线程中进行访问。再来看一个例子：

```c++
void f(int i, std::string const& s);
std::thread t(f, 3, "hello");
```

代码创建了一个调用f(3, "hello")的线程。注意，函数f需要一个`std::string`对象作为第二个参数，但这里使用的是字符串的字面值，也就是`char const *`类型。之后，在线程的上下文中完成字面值向`std::string`对象的转化。需要特别要注意，当指向动态变量的指针作为参数传递给线程的情况，代码如下：

```c++
void f(int i,std::string const& s);
void oops(int some_param)
{
  char buffer[1024]; // 1
  sprintf(buffer, "%i",some_param);
  std::thread t(f,3,buffer); // 2
  t.detach();
}
```

这种情况下，buffer②是一个指针变量，指向本地变量，然后本地变量通过buffer传递到新线程中②。并且，函数有很大的可能，会在字面值转化成`std::string`对象之前崩溃(*oops*)，从而导致线程的一些未定义行为。解决方案就是在传递到`std::thread`构造函数之前就将字面值转化为`std::string`对象。

```c++
void f(int i,std::string const& s);
void not_oops(int some_param)
{
  char buffer[1024];
  sprintf(buffer,"%i",some_param);
  std::thread t(f,3,std::string(buffer));  // 使用std::string，避免悬垂指针
  t.detach();
}
```

这种情况下的问题是，想要依赖隐式转换将字面值转换为函数期待的`std::string`对象，但因`std::thread`的构造函数会复制提供的变量，就只复制了没有转换成期望类型的字符串字面值。

不过，也有成功的情况：复制一个引用。成功的传递一个引用，会发生在线程更新数据结构时。

```c++
void update_data_for_widget(widget_id w,widget_data& data); // 1
void oops_again(widget_id w)
{
  widget_data data;
  std::thread t(update_data_for_widget,w,data); // 2
  display_status();
  t.join();
  process_widget_data(data); // 3
}
```

虽然update_data_for_widget①的第二个参数期待传入一个引用，但是`std::thread`的构造函数②并不知晓；构造函数无视函数期待的参数类型，并盲目的拷贝已提供的变量。当线程调用update_data_for_widget函数时，传递给函数的参数是data变量内部拷贝的引用，而非数据本身的引用。因此，当线程结束时，内部拷贝数据将会在数据更新阶段被销毁，且process_widget_data将会接收到没有修改的data变量③。使用`std::bind`，就可以解决这个问题，使用`std::ref`将参数转换成引用的形式。这种情况下，可将线程的调用，改成以下形式：

```c++
std::thread t(update_data_for_widget,w,std::ref(data));
```

在这之后，update_data_for_widget就会接收到一个data变量的引用，而非一个data变量拷贝的引用。

如果你熟悉`std::bind`，就应该不会对以上述传参的形式感到奇怪，因为`std::thread`构造函数和`std::bind`的操作都在标准库中定义好了，可以传递一个成员函数指针作为线程函数，并提供一个合适的对象指针作为第一个参数：

```c++
class X
{
public:
  void do_lengthy_work();
};
X my_x;
std::thread t(&X::do_lengthy_work,&my_x); // 1
```

这段代码中，新线程将my_x.do_lengthy_work()作为线程函数；my_x的地址①作为指针对象提供给函数。也可以为成员函数提供参数：`std::thread`构造函数的第三个参数就是成员函数的第一个参数，以此类推(代码如下，译者自加)。

```c++
class X
{
public:
  void do_lengthy_work(int);
};
X my_x;
int num(0);
std::thread t(&X::do_lengthy_work, &my_x, num);
```

有趣的是，提供的参数可以"移动"(*move*)，但不能"拷贝"(*copy*)。"移动"是指:原始对象中的数据转移给另一对象，而转移的这些数据就不再在原始对象中保存了(译者：比较像在文本编辑时"剪切"操作)。`std::unique_ptr`就是这样一种类型(译者：C++11中的智能指针)，这种类型为动态分配的对象提供内存自动管理机制(译者：类似垃圾回收)。同一时间内，只允许一个`std::unique_ptr`实现指向一个给定对象，并且当这个实现销毁时，指向的对象也将被删除。移动构造函数(*move constructor*)和移动赋值操作符(*move assignment operator*)允许一个对象在多个`std::unique_ptr`实现中传递(有关"移动"的更多内容，请参考附录A的A.1.1节)。使用"移动"转移原对象后，就会留下一个空指针(*NULL*)。移动操作可以将对象转换成可接受的类型，例如:函数参数或函数返回的类型。当原对象是一个临时变量时，自动进行移动操作，但当原对象是一个命名变量，那么转移的时候就需要使用`std::move()`进行显示移动。下面的代码展示了`std::move`的用法，展示了`std::move`是如何转移一个动态对象到一个线程中去的：

```c++
void process_big_object(std::unique_ptr<big_object>);

std::unique_ptr<big_object> p(new big_object);
p->prepare_data(42);
std::thread t(process_big_object,std::move(p));
```

在`std::thread`的构造函数中指定`std::move(p)`,big_object对象的所有权就被首先转移到新创建线程的的内部存储中，之后传递给process_big_object函数。

标准线程库中`std::unique_ptr`和`std::thread`在所属权上有相似语义的类型。虽然，`std::thread`实例不会如`std::unique_ptr`去占有一个动态对象所有权，但是它会占用一部分资源的所有权：每个实例都管理一个执行线程。`std::thread`所有权可以在多个实例中互相转移，因为这些实例是可移动(*movable*)且不可复制(*aren't copyable*)。在同一时间点，就能保证只关联一个执行线程；同时，也允许程序员能在不同的对象之间转移所有权。

##2.3 转移线程所有权

假设你要写一个用来在后台启动线程的函数，但现在想通过新线程返回的所有权去调用这个函数，而不是等待这个等待线程结束再去调用；或完全与之相反的想法：创建一个线程，并想要在函数中转移所有权，都必须要等待线程结束。不管怎样，想要完成你的想法，新线程的所有权都需要转移。
这就是移动引入`std::thread`的原因。在C++标准库中有很多资源占有（*resource-owning*）类型，比如`std::ifstream`,`std::unique_ptr`还有`std::thread`，都是可移动（*movable*），且不可拷贝（*cpoyable*）的。这就说明执行线程的所有权是可以在`std::thread`实例中移动，下面将展示一个例子。在这个例子中，创建了两个执行线程，并且在`std::thread`实例之间（t1,t2和t3）转移所有权：

```c++
void some_function();
void some_other_function();
std::thread t1(some_function);			// 1
std::thread t2=std::move(t1);			// 2
t1=std::thread(some_other_function);	// 3
std::thread t3;							// 4
t3=std::move(t2);						// 5
t1=std::move(t3);						// 6 赋值操作将使程序崩溃
```

首先，一个与t1相关的新线程启动 ① 。当显式使用` std::move()`创建t2后 ② ，t1的所有权就转移给了t2。现在，t1和执行线程已经没有关联了；执行**some_function**的函数现在与t2关联。
然后，与一个临时`std::thread`对象相关的线程启动了 ③ 。这里为什么不显式调用`std::move()`完成所有权的转移呢？因为，这里的所有者是一个临时对象——移动操作将会隐式的使用。
t3使用默认构造方式创建 ④，也就是没有与任何执行线程关联。当再次调用`std::move()`将与t2关联线程的所有权转移到t3中 ⑤。这里显式的调用了`std::move()`，是因为t2是一个命名对象。在移动操作⑤完成之后，t1与执行**some_other_function**的线程相关联，t2与任何线程都无关联，t3与执行**some_function**的线程相关联。
代码中最后一个移动操作，将执行**some_function**线程的所有权转移 ⑥ 给t1。这时，t1已经有了一个关联的线程（执行**some_other_function**的线程），所以这里会直接调用`std::terminate()`终止程序继续运行。终止操作将调用` std::thread`的析构函数，销毁所有对象（与C++中异常的处理方式很相似）。在2.1.1节时，你需要在线程对象被析构前，显式的等待一个线程完成，或者分离它；在进行复制时也需要满足这些条件（说明：你不能通过赋一个新值给`std::thread`对象的方式来“丢弃”一个线程）。
`std::thread`支持移动，就意味着线程的所有权，可以在函数外进行转移，就如下面程序清单中的程序一样。

清单2.5 从一个函数中返回一个`std::thread`对象
```c++
std::thread f()
{
  void some_function();
  return std::thread(some_function);
}
std::thread g()
{
  void some_other_function(int);
  std::thread t(some_other_function,42);
  return t;
}
```

同样的，当所有权可以在函数内部传递，那么这就允许将`std::thread`实例当做参数进行传递，代码如下：

```c++
void f(std::thread t);
void g()
{
  void some_function();
  f(std::thread(some_function));
  std::thread t(some_function);
  f(std::move(t));
}
```

`std::thread`支持移动的好处是，你可以创建**thread_guard**类的实例（定义见 清单2.3），并且拥有其线程的所有权。当**thread_guard**对象所持有的线程已经被引用时，移动操作就可以避免很多不必要的麻烦产生，并且这也意味着，当某个对象转移了线程的所有权后，它就不能对线程进行加入或分离的操作了。这也是为了确保线程在范围退出前完成，所以我在下面的代码里定义了*scoped_thread*类。现在，我们来看一下这段代码：

清单2.6 scoped_thread的用法
```c++
class scoped_thread
{
  std::thread t;
public:
  explicit scoped_thread(std::thread t_): 				// 1
    t(std::move(t_))
  {
    if(!t.joinable()) 									// 2
      throw std::logic_error(“No thread”);
  }
  ~scoped_thread()
  {
    t.join();											// 3
  }
  scoped_thread(scoped_thread const&)=delete;
  scoped_thread& operator=(scoped_thread const&)=delete;
};
 
struct func; // 定义在清单2.1中

void f()
{
  int some_local_state;
  scoped_thread t(std::thread(func(some_local_state)));	// 4
  do_something_in_current_thread();
}														// 5
```

这个例子与清单2.3相似，但这里新线程是直接传递到scoped_thread中④，而非创建了一个独立的命名变量。当主线程到达**f()**函数的末尾，**scoped_thread**对象将会销毁，然后加入 ③ 到的构造函数 ① 创建的线程对象中去。而在清单2.3中的**thread_guard**类，就要在析构的时候检查线程是否“可加入”。这里，我们把这个检查放在了构造函数中 ② ，并且当线程不可加入时，抛出一个异常。
对`std::thread`对象的容器，如果这个容器是移动敏感的（比如，新标准中的`std::vector<>`），那么移动操作同样适用于这些容器。了解这些后，你就可以写出类似清单2.7中的代码了，这部分代码量产了一些线程，并且等待他们结束。

清单2.7 量产一些线程，并且等待它们结束
```c++
void do_work(unsigned id);

void f()
{
  std::vector<std::thread> threads;
  for(unsigned i=0; i < 20; ++i)
  {
    threads.push_back(std::thread(do_work,i)); // 产生线程
  } 
  std::for_each(threads.begin(),threads.end(),
  				std::mem_fn(&std::thread::join)); // 对每个线程调用join()
}
```

我们经常需要线程去分割一个算法的总体工作，并且在算法结束的之前，所有的线程必须要结束。在清单2.7中的实现，说明线程所做的工作都是独立的，并且结果仅会受到共享数据的影响。如果**f()**有返回值，那么这个返回值就依赖于线程得到的结果。并在写入返回值之前，程序会检查使用共享数据的线程是否终止。操作结果在不同线程中转移的替代方案，我们会在第4章中再次讨论。
将`std::thread`放入`std::vector`是向线程自动化管理迈出的第一步：并非为这些线程创建独立的变量，并且将他们直接加入，可以把它们当做一个组。创建一组线程（数量在运行时确定），可使得这一步，迈的更大；而非像清单2.7那样创建固定数量的线程。

##2.4 在运行时决定线程数量

`std::thread::hardware_concurrency()`在新版C++标准库中，是一个很有用的函数。这个函数将返回能同时并发在一个程序中的线程数量。例如，在多核系统中，返回值就可以能是CPU核芯的数量。这个返回值也仅仅是一个提示，当这种信息无法获取，函数也会返回0。但是，这也无法掩盖这个函数对启动线程数量的确定是有帮助的。
清单2.8实现了一个并行版的`std::accumulate`。代码中将整体工作拆分成小任务交给每个线程去做，其中设置最小任务数，是为了避免产生太多的线程。程序可能会在操作数量为0的时候抛出异常。比如，`std::thread`构造函数无法启动一个执行线程，就会抛出一个异常。在这个算法中讨论异常处理，已经超出现阶段的讨论范围，这个问题我们将在第8章中再来讨论。

清单2.8 原生并行版的`std::accumulate`
```c++
template<typename Iterator,typename T>
struct accumulate_block
{
  void operator()(Iterator first,Iterator last,T& result)
  {
    result=std::accumulate(first,last,result);
  }
};
template<typename Iterator,typename T>
T parallel_accumulate(Iterator first,Iterator last,T init)
{
  unsigned long const length=std::distance(first,last);

  if(!length) // 1
    return init;

  unsigned long const min_per_thread=25;
  unsigned long const max_threads=
      (length+min_per_thread-1)/min_per_thread; // 2

  unsigned long const hardware_threads=
      std::thread::hardware_concurrency();

  unsigned long const num_threads=  // 3
      std::min(hardware_threads != 0 ? hardware_threads : 2, max_threads);

  unsigned long const block_size=length/num_threads; // 4

  std::vector<T> results(num_threads);
  std::vector<std::thread> threads(num_threads-1);  // 5

  Iterator block_start=first;
  for(unsigned long i=0; i < (num_threads-1); ++i)
  {
    Iterator block_end=block_start;
    std::advance(block_end,block_size);  // 6
    threads[i]=std::thread(     // 7
        accumulate_block<Iterator,T>(),
        block_start,block_end,std::ref(results[i]));
    block_start=block_end;  // 8
  }
  accumulate_block<Iterator,T>()(
      block_start,last,results[num_threads-1]); // 9
  std::for_each(threads.begin(),threads.end(),
       std::mem_fn(&std::thread::join));  // 10

  return std::accumulate(results.begin(),results.end(),init); // 11
}
```

函数看起来很长，但不复杂。如果输入的范围为空 ① ，你就会得到init的值。反之，如果范围内多于一个元素时，你都需要用范围内元素的总数量除以线程（块）中最小任务数，从而确定启动线程的最大数量 ② 。这就能避免无谓的计算资源的浪费。比如，在一台32芯的机器上，你只有5个数需要计算，却启动了32个线程。
启动线程数量的确定，计算量的最大值和硬件支持线程数中较小的值为启动线程数量 ③ 。因为上下文频繁的切换会降低线程的性能，所以你肯定不想启动的线程数多于硬件支持的线程数量（称为**超额认购**（*oversubscription*））。当`std::thread::hardware_concurrency()`返回0，你可以选择一个合适的数作为你的选择；在本例中我选择了“2”这个数。你肯定也不想在一台单核机器上启动太多的线程，这样反而会降低性能，最终放弃使用并发。
每个线程中处理的元素数量，是范围中元素的总量除以线程的个数得出的 ④ 。分配不均的问题，会在后面处理。
现在，知道要运行多少个线程，可以创建一个`std::vector<T>`容器存放中间结果，并为线程创建一个`std::vector<std::thread>`容器 ⑤。这里需要注意的是，启动的线程数必须比num_threads少1个，因为在启动之前你已经有一个线程了（主线程）。
使用简单的循环来启动线程：block_end迭代器指向当前块的末尾 ⑥ ，并启动一个新线程为当前块累加结果 ⑦。当迭代器指向当前块的末尾时，启动下一个块 ⑧。
在你启动所有线程后，⑨ 中的线程会处理最终块的结果。这里就是你之前考虑分配不均的地方：你知道最终块是最后一个，并且在这个块中有多少个元素已经无关紧要了。
当累加最终块的结果后，你可以等待`std::for_each` ⑩ 创建线程的完成
（如同在清单2.7中做的那样），之后使用`std::accumulate`将所有结果进行累加 ⑪。
在结束这个例子之前，需要明确的是对于T类型的加法运算是不满足结合律的（比如，对于float型或double型），**parallel_accumulate**所得到的结果可能与`std::accumulate`得到的结果有所不同，这是因为对范围中元素的分组所致。同样的，这里对迭代器的要求更加严格：它们必须都是向前迭代器（*forward iterator*），而`std::accumulate`可以在只传入迭代器（*input iterators*）的情况下工作，并且T必须是可以默认构造的，这样才能保证你能创建出results容器。对于算法并行来说，这些变化是比较普遍的；
不过，由于算法本身的特性，要使它们并行需要使用不用的方式。算法并行会在第8章有更加深入的讨论。还需要注意的是：因为你不能直接从一个线程中返回一个值，所以你需要传递results容器的引用到线程中去。另一个办法是，通过地址来获取线程执行的结果，在第4章中，我们将使用期望（*futures*）来完成这种方案。
当线程运行时，所有必要的信息都需要传入到线程中去，包括存储计算结果的位置。不过，并非总是如此：有时候这是识别线程的可行方案，你可以传递一个标识数。例如，在清单2.7中的i。不过，当需要标识的函数在调用栈的深层，并可让任何线程进行调用，那么使用标识数的办法就会变的捉襟见肘。好消息是，在设计C++的线程库时，我们就有预见过这种情况，在之后的实现中就给每个线程附加了唯一的标识符。

##2.5 识别线程

线程标识类型是`std::thread::id`，可以通过两种方式进行检索。第一种，可以通过调用` std::thread`对象的成员函数`get_id()`来直接获取。如果`std::thread`对象没有与任何执行线程相关联，那么`get_id()`将返回`std::thread::type`默认构造值，这个值表示“没有线程”。第二种，在当前线程中，调用`std::this_thread::get_id()`（这个函数定义在<thread>头文件中）也可以获得线程标识。
`std::thread::id`对象可以自由的拷贝和对比；标识符就可以复用。如果两个对象的`std::thread::id`相等，那它们就是同一个线程，或者都“没有线程”。如果不等，那么它们就代表了两个不同线程，或者一个有线程，另一没有。
线程库不会限制你去检查线程标识是否一样；`std::thread::id`类型对象提供相当丰富的对比操作，比如，提供为不同的值进行排序。这就意味着允许程序员将其当做为容器的键值，或做排序，或做其他方式的比较。比较操作将不同值的`std::thread::id`按默认顺序排序，所以这个行为可预见的：当```a<b```，```b<c```时，得```a<c```，等等。标准库也提供`std::hash<std::thread::id>`容器，这样`std::thread::id`也可以作为无序容器的键值了。
` std::thread::id`实例常用作检测线程是否需要进行一些操作。比如：当用线程来分割一项工作（如清单2.8），主线程可能要做一些与其他线程不同的工作。在这种情况下，在启动其他线程前，它可以将自己的线程ID通过`std::this_thread::get_id()`得到，并进行存储。然后，就是算法的核心部分（所有线程都一样的）：每个线程都要检查一下，其拥有的线程ID是否与初始线程的ID相同。

```c++
std::thread::id master_thread;
void some_core_part_of_algorithm()
{
  if(std::this_thread::get_id()==master_thread)
  {
    do_master_thread_work();
  }
  do_common_work();
}
```
另外，当前线程的`std::thread::id`将存储到一个数据结构中。之后会在这个结构体中对当前线程的ID与存储的线程ID做对比，来决定操作是做“允许”操作，还是“需要”操作（*permitted/required*）。
同样的，线程ID在容器中可作为键值，作为线程和本地存储不适配的替代方案。例如，容器可以由一个控制线程存储每个在其控制下线程的信息，或在多个线程中互传信息。
`std::thread::id`足够作为一个线程的通用标识符；当标识符只与语义相关（比如，一个数组的索引）时，就需要这个方案来解决了。你甚至可以使用输出流（`std::cout`）来记录一个`std::thread::id`对象的值。

```c++
std::cout<<std::this_thread::get_id();
```

这里确切的输出结果是严格依赖于具体实现的；C++标准的唯一要求就是要保证ID比较结果相等的线程必须有相同的输出，ID不相等的线程应该有不同的输出。这对于记录和调试来说，是很有帮助的。但是，打印出来的值没有任何语义，所以除了给记录和调试带来一些帮助之外，就没有其他用处了。

##2.6 总结

这章中讨论了C++标准库中基本的线程管理方式：启动线程，等待结束，和不等待结束（因为需要它们运行在后台）。并了解应该如何在线程启动前，向线程函数中传递参数，如何转移线程的所有权，如何使用线程组来分割任务。最后，讨论了使用线程标识来确定与之关联数据，和特殊线程的特殊解决方案。虽然，你现在已经可以纯粹的依赖线程，使用独立的数据，做独立的任务（如同清单2.8），但某些情况下线程运行时，确实需要有共享数据。第3章会讨论共享数据和线程的直接关系。第4章会讨论在（有/没有）共享数据情况下的线程同步操作。


