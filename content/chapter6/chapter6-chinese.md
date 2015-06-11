#第六章 基于锁的并发数据结构设计

**本章主要内容**

>并发数据结构设计的意义<br>
>指导如何设计<br>
>实现为并发设计的数据结构<br>

在上一章中，我们对底层原子操作和内存模型有了详尽的了解。在本章中，我们将先将底层的东西放在一边(将会在第7章再次提及)，来对数据结构做一些讨论。

数据结构的选择，对于程序来说，是其解决方案的重要组成部分，当然，并行程序也不例外。当数据结构可以被多个线程所访问，其要不就是绝对不变的(其值不可能发生变化，并且不需要同步)，要不程序就要对数据结构进行正确的设计，以确保其能在多线程环境下能够(正确的)同步。一种选择是使用独立的互斥量，其可以锁住需要保护的数据(这种方法已经在第3和第4章中提到)，另一种选择是设计一种能够并发访问的数据结构。

在设计并发数据结构时，你可以使用基本多线程应用中的构建块(之前章节中有提及)，比如，互斥量和条件变量。当然，你也已经在之前的章节的例子中看到，怎样联合不同的构建块，对数据结构进行写入，并且这些构建块都是在并发环境下，线程安全的。

在本章，我们将了解一些并发数据结构设计的基本准则。然后，我们将再次重温锁和条件变量的基本构建块。最后，将去了解更为复杂的数据结构。在第7章，我们将了解到，如何正确的“返璞归真”，并且使用第5章提到的原子操作，去构建无锁的数据结构。

好吧！多说无益，让我们来看一下并发数据结构的设计，都需要些*什么*。

#6.1 为并发设计的意义何在？ 

为并发设计数据结构，意味着多个线程可以并发的对这个数据结构进行访问，线程可对这个数据结构做相同的操作，也可以做不同的操作，并且每一个线程能在自治域中看到该数据结构。并在多线程环境下，无数据丢失和损毁，所有的变量都会维持原样，且无条件竞争。这样的数据结构，称之为“线程安全”的数据结构。通常，只对数据结构中的特定类型，进行并发访问，是安全的。线程安全意味着，当有多个线程对数据结构，进行同一种并发操作，而另一种操作则需要单线程独立访问数据结构。或许是，当线程执行不同操作时，对同一数据结构的并发访问是安全的，而多线程执行同样的操作，将会出现问题。

实际的设计意义要大于上面所提到的那样：其意味为线程提供并发访问数据结构的机会。本质上，互斥量提供互斥：在互斥量的保护下，同一时间只有一个线程可以获取互斥锁。互斥量为了保护数据结构，显式的阻止了线程对数据结构的并发访问。

这就是*序列化*(*serialzation*)：线程轮流访问被互斥量保护的数据；这时，线程其实是对数据进行串行的访问，而非并发。因此，你需要对数据结构的设计进行仔细考量，确保其能并发访问。虽然一些数据结构有着比其他数据结构多的并发访问范围，但是在所有情况下的思路都是一样的：少保护区域，少序列化操作，多并发访问潜力。

在我们了解数据结构的设计之前，让我们快速的浏览一下，“在设计并发时所要考量什么的”指导建议。

##6.1.1 数据结构并发设计的指导与建议(指南)

如我刚刚提到的，当设计并发数据结构时，有两方面需要考量：确保访问是安全的，且是能真正并发访问的。我在第3章的时候已经对，如何保证数据结构是线程安全，做过简单的描述：

- 确保无线程能够看到，数据结构的“不变量”在被某一线程破坏时的状态。

- 小心那些会引起条件竞争的结接口，提供完成操作的函数，而非操作步骤。

- 注意数据结构的行为是否会产生异常，从而确保“不变量”不被破坏。

- 将死锁的概率降到最低。使用数据结构的时候，需要限制锁的范围，并且避免嵌套锁的存在。

在你思考设计细节前，你还需要考虑这个数据结构对于使用者来说有什么限制；当一个线程通过一个特殊的函数对数据结构进行访问，那么还有哪些函数能被其他线程安全的调用呢？

这是一个很关键的问题。因为，普通的构造函数和析构函数需要独立访问数据结构，所以用户在使用的时候，就必须确保这两个函数不能再构造完成前，或在析构执行时，再次调用。当数据结构支持赋值，swap()，或拷贝构造，作为数据结构的这设计者，你需要保证这些操作在并发环境下是安全的，或他们是否需要确保访问独立，即使数据结构中有大量的函数被线程所操纵，并发访问也不会出现错误。

第二个方面需要考量的是，确保真正的并发访问。这里我没法提供更多的指导意见；不过，作为一个数据结构设计者，在设计数据结构时，可以自问以下问题：

- 在锁保护的范围中，是否允许其他部分，在锁外执行？

- 数据结构不同的区域是否能被不同的互斥量所保护？

- 所有操作都需要同级保护吗？

- 能否对数据结构进行简单的修改，就能增加并发访问的概率，且不影响操作语义？

这些问题都源于一个指导思想：如何让序列访问降到最低，并且最大程度的真正并发？能够允许线程并发读取的数据结构并不少见，而单线程对数据结构的修改，必须是独立访问。这种结构，类似于`boost::shared_mutex`。同样的，这种数据结构也很常见——支持在多线程执行不同的操作时，序列化去执行相同的操作的线程(你很快就能看到)。

最简单的线程安全数据结构，通常使用的是互斥量和锁对数据进行保护。虽然这么做还是有问题，如同在第3中提到的那样，其相对简单，且保证只有一个线程在同一时间对数据结构进行一次访问。为了让你轻松的设计线程安全的数据结构，接下来我们了解一下本章基于锁的数据结构，以及第7章将提到的无锁并发数据结构的设计。

##6.2 基于锁的并发数据结构

基于锁的并发数据结构设计，需要确保，当数据被访问时上锁，以及持有锁的时间最短。对于只有一个互斥量的数据结构来说，这挺难的。你需要保证数据不被锁之外的操作所访问到，并且还要保证不会在固有结构上产生条件竞争(如第3章所述)。当你使用多个互斥量来保护数据结构中不同的区域时，问题会暴露的更明显，当操作获取多于一个互斥锁时，就有可能产生死锁。所以，在设计时，使用多个互斥量时需要格外小心。

在本节中，你将使用6.1.1节中的指导建议，来设计一些简单的数据结构——使用互斥量和锁的方式来保护数据。在每一个例子中，你都能找到，在保证数据结构是线程安全的前提下，提高数据结构并发访问的概率(机会)。

我们先来看看在第3章中，栈的实现；这个实现就是一个非常非常简单的数据结构，并且它只使用了一个互斥量。不过，这个结构线程安全吗？它离真正的并发访问又差多少呢？

###6.2.1 线程安全栈——使用锁

我们先把第3章中线程安全的栈拿过来看看：(这里试图实现一个线程安全版的`std:stack<>`)

清单6.1 线程安全栈的类定义
```c++
#include <exception>

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
    data=other.data;
  }

  threadsafe_stack& operator=(const threadsafe_stack&) = delete;

  void push(T new_value)
  {
    std::lock_guard<std::mutex> lock(m);
    data.push(std::move(new_value));  // 1
  }
  std::shared_ptr<T> pop()
  {
    std::lock_guard<std::mutex> lock(m);
    if(data.empty()) throw empty_stack();  // 2
    std::shared_ptr<T> const res(
      std::make_shared<T>(std::move(data.top())));  // 3
    data.pop();  // 4
    return res;
  }
  void pop(T& value)
  {
    std::lock_guard<std::mutex> lock(m);
    if(data.empty()) throw empty_stack();
    value=std::move(data.top());  // 5
    data.pop();  // 6
  }
  bool empty() const
  {
    std::lock_guard<std::mutex> lock(m);
    return data.empty();
  }
};
```

让我们对照着指导意见，看看它们在这里是如何应用的。

首先，如你所见，互斥量m能保证基本的线程安全，那就是对每个成员函数今进行加锁保护。这就保证在同一时间，只有一个线程可以访问到数据，所以能都保证，数据结构的“不变量”被破坏时，不会被其他线程看到。

其次，在empty()和pop()成员函数之间会存在潜在的竞争，不过因为代码会在pop()函数上锁时，显式的查询栈是否为空，所以这里的竞争是非恶性的。pop()通过对弹出值的直接返回，可以避免`std::stack<>`中top()和pop()两成员函数之间的潜在竞争。

再次，这个类中也有一些异常源。对互斥量上锁是，可能会抛出异常，不过不仅是极其罕见的(因为这意味这问题不在锁上，就是在系统资源上)，其也是每个成员函数所做的第一个操作。因为无数据修改，所以其是安全的。因为解锁一个互斥量是不会失败的，所以这里会一直很安全，并且使用`std::lock_guard<>`也能保证互斥量上锁的状态。

对data.push()①的调用可能会抛出一个异常，不是拷贝/移动数据值时，就是内存不足的时候。不管是哪种，`std::stack<>`都能保证其实安全的，所以这里也没有问题。

在第一个重载pop()中，代码可能会抛出一个empty_stack的异常②，不过数据没有被修改，所以其是安全的。对于res的创建③，也可能会抛出一个异常，这有两方面的原因：对`std::make_shared`的调用，可能无法分配出足够的内存去创建新的对象，并且内部数据需要对新对象进行引用；或者，当拷贝或移动到新分配的内存中，拷贝或移动构造返回时，可能会抛出异常。两种情况下，c++运行库和标准库能确保，这里不会出现内存泄露，并且新创建的对象(如果有的话)都能被正确销毁。因为你还是没有对栈进行任何修改，所以这里也不会有问题。当调用data.pop()④时，其能确保不抛出异常，并且返回结果，所以这个重载pop()函数“异常-安全”。

第二个重载pop()类似，除了在拷贝赋值或移动赋值的时候会抛出异常⑤，当构造一个新对象和一个`std::shared_ptr`实例时都不会抛出异常。同样，在调用data.pop()⑥（这个成员函数保证不会抛出异常）之前，依旧没有对数据结构进行修改，所以这个函数也为“异常-安全”。

最后，empty()也不会修改任何数据，所以其也是“异常-安全”函数。

因为在调用用户代码会持有锁，所以这里有两个可能会产生死锁地方：拷贝构造或移动构造时(①，③)和在对数据项进行拷贝赋值或移动赋值操作⑤的时候；还有一个潜在死锁的地方是在，用户定义的操作符new上。当这些函数，无论是调用栈的成员函数，对已经插入或删除的数据进行操作，还是在栈的成员函数被调用时，以任意方式获取锁的方式，都有可能造成死锁。不过，用户要对栈负责，当栈未对一个数据进行拷贝或分配是，你就不能想当然的将其添加到栈中。

因为所有成员函数都使用`st::lack_guard<>`来保护数据，所以栈的成员函数能有“线程安全”的表现。当然，构造与析构函数不是“线程安全”的，不过这也不成问题，因为对象的构造与析构只能有一次。调用一个不完全构造对象或是已销毁对象的成员函数，无论在那种编程方式下，都是不可取的。所以，用户就要保证在栈对象完成构建前，其他线程是无法对其进行访问的；并且，一定要保证在栈对象销毁后，所有线程都要停止对其进行访问。

即使在多线程情况下，因为使用锁，并发的调用成员函数是安全的，也要保证在单线程的情况下，数据结构做出正确反应。序列化的线程会隐性的限制程序的性能，这里就是栈争议声最大的地方：当一个线程在等待锁时，它就会无所事事。同样的，对于栈来说，等待添加元素也是没有意义的，所以当一个线程需要等待时，其会定期检查empty()或pop()，以及对empty_stack异常进行关注。这样的现实会限制栈的实现的方式，在线程等待的时候，会浪费宝贵的资源去检查数据，或是用户必须写外部等待和提示代码(例如，使用条件变量)，这就使内部锁失去存在的意义——这就意味着资源的浪费。第4章中的队列，就是一种使用条件内部变量进行等待的数据结构，接下来我们就来了解一下。

###6.2.2 线程安全队列——使用锁和条件变量

第4章中的线程安全队列，在清单6.2中重现一下。和使用仿`std::stack<>`建立的栈很像，这里队列的建立也是参照了`std::queue<>`。不过，还是要与标准容器的接口不同的，因为我们要写的是能在多线程下安全并发访问的数据结构。

清单6.2 使用条件变量实现的线程安全队列
```c++
template<typename T>
class threadsafe_queue
{
private:
  mutable std::mutex mut;
  std::queue<T> data_queue;
  std::condition_variable data_cond;
public:
  threadsafe_queue()
  {}

  void push(T new_value)
  {
    std::lock_guard<std::mutex> lk(mut);
    data_queue.push(std::move(data));
    data_cond.notify_one();  // 1
  }

  void wait_and_pop(T& value)  // 2
  {
    std::unique_lock<std::mutex> lk(mut);
    data_cond.wait(lk,[this]{return !data_queue.empty();});
    value=std::move(data_queue.front());
    data_queue.pop();
  }

  std::shared_ptr<T> wait_and_pop()  // 3
  {
    std::unique_lock<std::mutex> lk(mut);
    data_cond.wait(lk,[this]{return !data_queue.empty();});  // 4
    std::shared_ptr<T> res(
      std::make_shared<T>(std::move(data_queue.front())));
    data_queue.pop();
    return res;
  }

  bool try_pop(T& value)
  {
    std::lock_guard<std::mutex> lk(mut);
    if(data_queue.empty())
      return false;
    value=std::move(data_queue.front());
    data_queue.pop();
    return true;
  }

  std::shared_ptr<T> try_pop()
  {
    std::lock_guard<std::mutex> lk(mut);
    if(data_queue.empty())
      return std::shared_ptr<T>();  // 5
    std::shared_ptr<T> res(
      std::make_shared<T>(std::move(data_queue.front())));
    data_queue.pop();
    return res;
  }

  bool empty() const
  {
    std::lock_guard<std::mutex> lk(mut);
    return data_queue.empty();
  }
};
```

除了在push()①中调用data_cond.notify_one()和wait_and_pop()，6.2中对队列的实现与6.1中对栈的实现十分相近。两个重载try_pop()除了在队列为空时抛出异常，其他的与6.1中pop()函数完全一样。不同的是，在6.1中当值可以检索的时候返回的是一个bool值，而在6.2中，当指针指向空值的时候会返回NULL指针⑤。着同样也是实现栈的一个有效途径。所以，即使你排除掉wait_and_pop()函数，之前对栈的分析依旧适用于这里。

新wiat_and_pop()函数，是等待队列向栈尽心输入的一个解决方案；比起持续调用empty()，等待线程调用wait_and_pop()函数和数据结构处理等待中的条件变量的方式要好很多。对于data_cond.wait()的调用，直到队列中有一个元素的时候，才会返回，所以你就不用担心会出现一个空队列的情况了，还有，数据会一直被互斥锁保护。这些函数不会添加新的条件竞争或是死锁的可能，因为不变量这里并未发生变化。

异常安全在这里的会有一些变化，当不止一个线程等待对队列进行推送操作是，只会有一个线程，因得到data_cond.notify_one()，而继续工作着。但是，如果这个工作线程在wait_and_pop()中抛出一个异常，例如构造新的`std::shared_ptr<>`对象④，那么其他线程则会永世长眠。当这种情况是不可接受时，这里的调用就需要改成data_cond.notify_all()，这个函数将唤醒所有的工作线程，但是，当大多线程发现队列依旧是空时，又会耗费很多资源让线程重新进入睡眠状态。第二种替代方案是，当有异常抛出的时候，让wait_and_pop()函数调用notify_one()，从而让个另一个线程可以去尝试索引存储的值。第三种替代方案就是，将`std::shared_ptr<>`的初始化过程，移到push()中，并且存储`std::shared_ptr<>`实例，而非直接使用数据的值。将`std::shared_ptr<>`拷贝到内部`std::queue<>`中，就不会抛出异常了，这样，wait_and_pop()又是安全的了。下面的程序清单，就是根据第三种方案进行修订的。

清单6.3 持有`std::shared_ptr<>`实例的线程安全队列
```c++
template<typename T>
class threadsafe_queue
{
private:
  mutable std::mutex mut;
  std::queue<std::shared_ptr<T> > data_queue;
  std::condition_variable data_cond;
public:
  threadsafe_queue()
  {}

  void wait_and_pop(T& value)
  {
    std::unique_lock<std::mutex> lk(mut);
    data_cond.wait(lk,[this]{return !data_queue.empty();});
    value=std::move(*data_queue.front());  // 1
    data_queue.pop();
  }

  bool try_pop(T& value)
  {
    std::lock_guard<std::mutex> lk(mut);
    if(data_queue.empty())
      return false;
    value=std::move(*data_queue.front());  // 2
    data_queue.pop();
    return true;
  }

  std::shared_ptr<T> wait_and_pop()
  {
    std::unique_lock<std::mutex> lk(mut);
    data_cond.wait(lk,[this]{return !data_queue.empty();});
    std::shared_ptr<T> res=data_queue.front();  // 3
    data_queue.pop();
    return res;
  }

  std::shared_ptr<T> try_pop()
  {
    std::lock_guard<std::mutex> lk(mut);
    if(data_queue.empty())
      return std::shared_ptr<T>();
    std::shared_ptr<T> res=data_queue.front();  // 4
    data_queue.pop();
    return res;
  }

  void push(T new_value)
  {
    std::shared_ptr<T> data(
    std::make_shared<T>(std::move(new_value)));  // 5
    std::lock_guard<std::mutex> lk(mut);
    data_queue.push(data);
    data_cond.notify_one();
  }

  bool empty() const
  {
    std::lock_guard<std::mutex> lk(mut);
    return data_queue.empty();
  }
};
```

让`std::shared_ptr<>`持有数据的结果显而易见：弹出函数会持有一个变量的引用，为了接收这个新值，必须对存储的指针进行解引用①，②；并且，在返回到调用函数前，弹出函数都会返回一个`std::shared_ptr<>`实例，这里实例可以在队列中做检索③，④。

数据被`std::shared_otr<>`持有的好处：当新的实例分配结束时，其不会被锁在push()⑤的锁当中，而在清单6.2中，只能在pop()持有锁时完成。因为内存分配通常是代价昂贵的操作，这种方式对队列的性能有很大的帮助，其削减了互斥量持有的时间，允许其他线程在分配内存的同时，对队列执行其他的操作。

就像栈的例子那样，使用互斥量保护整个数据结构，不过会限制队列对并发的支持；虽然，多线程可能被队列中的各种成员函数所阻塞，但是仍有一个线程能在任意时间内进行工作。不过，这种限制的部分来源是因为在实现中使用了`std::queue<>`；因为使用标准容器的原因，数据要不处于保护中，要不就没有。要对数据结构实现进行具体的控制，需要提供更多细粒度锁，来完成高等级的并发。

###6.2.3 线程安全队列——使用细粒度锁和条件变量

在清单6.2和6.3中，使用一个互斥量对一个数据(data_queue)进行保护。为了使用细粒度锁，需要看一下队列内部的组成结构，并且将一个互斥量与每个数据相关联。

对于队列来说，最简单的数据结构就是单链表了，就如图6.1那样。队列里包含一个头指针，其指向链表中的第一个元素，并且每一个元素都会指向下一个元素。从队列中删除数据，其实就是将头指针指向下一个元素，并将之前头指针指向的值进行返回。

向队列中添加元素是要从结尾进行的。为了做到这点，队列里还有一个尾指针，其指向链表中的最后一个元素。新节点的加入将会改变尾指针的next指针，之前最后一个元素将会指向新添加进来的元素，新添加进来的元素的next将会使新的尾指针。当链表为空时，头/尾指针皆为NULL。

![](https://raw.githubusercontent.com/xiaoweiChen/Cpp_Concurrency_In_Action/master/images/chapter6/6-1.png)

图6.1 用单链表表示的队列

下面的清单中的代码，是一个简单的队列实现，基于清单6.2代码的精简版本；因为这个队列仅供单线程使用，所以这实现中只有一个try_pop()函数；并且，没有wait_and_pop()函数。

清单6.4 队列实现——单线程版
```c++
template<typename T>
class queue
{
private:
  struct node
  {
    T data;
    std::unique_ptr<node> next;

    node(T data_):
    data(std::move(data_))
    {}
  };

  std::unique_ptr<node> head;  // 1
  node* tail;  // 2

public:
  queue()
  {}
  queue(const queue& other)=delete;
  queue& operator=(const queue& other)=delete;
  std::shared_ptr<T> try_pop()
  {
    if(!head)
    {
      return std::shared_ptr<T>();
    }
    std::shared_ptr<T> const res(
      std::make_shared<T>(std::move(head->data)));
    std::unique_ptr<node> const old_head=std::move(head);
    head=std::move(old_head->next);  // 3
    return res;
  }

  void push(T new_value)
  {
    std::unique_ptr<node> p(new node(std::move(new_value)));
    node* const new_tail=p.get();
    if(tail)
    {
      tail->next=std::move(p);  // 4
    }
    else
    {
      head=std::move(p);  // 5
    }
    tail=new_tail;  // 6
  }
};
```

首先，注意在清单呢6.4中使用了`std::unique_ptr<node>`来管理节点，因为其能保证节点(其引用数据的值)在删除时候，不需要使用delete操作显式删除。这样的关系链表，管理着从头结点到尾节点的每一个原始指针。

虽然，这种实现对于单线程来说没什么问题，但是，当你在多线程情况下，尝试使用细粒度锁时，就会出现问题。因为在给定的实现中有两个数据项(head①和tail②)；即使，使用两个互斥量，来保护头指针和尾指针，也会出现问题。

显而易见的问题就是push()可以同时修改头指针⑤和尾指针⑥，所以push()函数会同时获取两个互斥量。虽然会将两个互斥量都上锁，但这还不是太糟糕的问题。糟糕的问题是push()和pop()都能访问next指针指向的节点：push()可更新tail->next④，而后try_pop()读取read->next③。当队列中只有一个元素时，head==tail，所以head->next和tail->next是同一个对象，并且这个对象需要保护。不过，“在同一个对象在未被head和tail同时访问时，push()和try_pop()锁住的是同一个锁”，就不对了。所以，你就没有比之间实现更好的选择了。这里会“柳暗花明又一村”吗？

**通过分离数据实现并发**

你可以使用“预分配一个虚拟节点(无数据)，确保这个节点永远在队列的最后，用来分离头尾指针能访问的节点”的办法，走出这个困境。对于一个空队列来说，head和tail都属于虚拟指针，而非空指针。这个办法挺好，因为当队列为空时，try_pop()不能访问head->next了。当添加一个节点入队列时(这时有真实节点了)，head和tail现在指向不同的节点，所以就不会在head->next和tail->next上产生竞争。这里的缺点是，你必须额外添加一个间接层次的指针数据，来做虚拟节点。下面的代码描述了这个方案如何实现。

清单6.5 带有虚拟节点的队列
```c++
template<typename T>
class queue
{
private:
  struct node
  {
    std::shared_ptr<T> data;  // 1
    std::unique_ptr<node> next;
  };

  std::unique_ptr<node> head;
  node* tail;

public:
  queue():
    head(new node),tail(head.get())  // 2
  {}
  queue(const queue& other)=delete;
  queue& operator=(const queue& other)=delete;

  std::shared_ptr<T> try_pop()
  {
    if(head.get()==tail)  // 3
    {
      return std::shared_ptr<T>();
    }
    std::shared_ptr<T> const res(head->data);  // 4
    std::unique_ptr<node> old_head=std::move(head);
    head=std::move(old_head->next);  // 5
    return res;  // 6
  }

  void push(T new_value)
  {
    std::shared_ptr<T> new_data(
      std::make_shared<T>(std::move(new_value)));  // 7
    std::unique_ptr<node> p(new node);  //8
    tail->data=new_data;  // 9
    node* const new_tail=p.get();
    tail->next=std::move(p);
    tail=new_tail;
  }
};
```

对于try_pop()做了很少的修改。首先，你可以拿head和tail③进行比较，这就要比检查指针是否为空的好，因为虚拟节点意味着head不可能是空指针。因为head是一个`std::unique_ptr<node>`对象，你需要使用head.get()来做比较。其次，因为node现在存在数据指针中①，你就可以对指针进行直接检索④，而非构造一个T类型的新实例。push()函数改动最大：首先，你必须在堆上创建一个T类型的实例，并且让其与一个`std::shared_ptr<>`对象相关联⑦(节点使用`std::make_shared`就是为了避免内存二次分配，避免增加引用次数)。创建的新节点就成为了虚拟节点，所以你不需要为new_value提供构造函数⑧。反而这里你需要将new_value的副本赋给之前的虚拟节点⑨。最终，为了让虚拟节点存在在队列中，你不得不使用构造函数来创建它②。

那么现在，我确信你会对如何对如何修改队列，让其变成一个线程安全的队列感到惊讶。好吧，现在的push()只能访问tail，而不能访问head，这就是一个进步try_pop()可以访问head和tail，但是tail只需在最初进行比较，所以所存在的时间很短。重大的提升在于，虚拟节点意味着try_pop()和push()不能对同一节点进行操作，所以这里已经不再需要互斥了。那么，你只需要使用一个互斥量来保护head和tail就够了。那么，现在应该锁哪里？

我们的目的是为了最大程度的并发化，所以你需要上锁的时间，要尽可能的小。push()很简单：互斥量需要对tail的访问进行上锁，这就意味着你需要你需要对每一个新分配的节点进行上锁⑧，还有在你对当前尾节点进行赋值的时候⑨也需要上锁。锁需要持续到函数结束时才能解开。

try_pop()就不简单了。首先，你需要使用互斥量锁住head，一直到head弹出。本质上，互斥量决定了哪一个线程来进行弹出操作。一旦head被改变⑤，你擦能解锁互斥量；当在返回结果时，互斥量就不需要进行上锁了⑥。这使得访问tail需要一个尾互斥量。因为，你需要只需要访问tail一次，且只有在访问时才需要互斥量。这个操作最好是通过函数进行包装。事实上，因为代码只有在成员需要head时，互斥量才上锁，这项也需要包含在包装函数中。最终代码如下所示。

清单6.6 线程安全队列——细粒度锁版
```c++
template<typename T>
class threadsafe_queue
{
private:
  struct node
  {
    std::shared_ptr<T> data;
    std::unique_ptr<node> next;
  };
  std::mutex head_mutex;
  std::unique_ptr<node> head;
  std::mutex tail_mutex;
  node* tail;

  node* get_tail()
  {
    std::lock_guard<std::mutex> tail_lock(tail_mutex);
    return tail;
  }

  std::unique_ptr<node> pop_head()
  {
    std::lock_guard<std::mutex> head_lock(head_mutex);
    if(head.get()==get_tail())
    {
      return nullptr;
    }
    std::unique_ptr<node> old_head=std::move(head);
    head=std::move(old_head->next);
    return old_head;
  }
public:
  threadsafe_queue():
  head(new node),tail(head.get())
  {}
  threadsafe_queue(const threadsafe_queue& other)=delete;
  threadsafe_queue& operator=(const threadsafe_queue& other)=delete;

  std::shared_ptr<T> try_pop()
  {
     std::unique_ptr<node> old_head=pop_head();
     return old_head?old_head->data:std::shared_ptr<T>();
  }

  void push(T new_value)
  {
    std::shared_ptr<T> new_data(
      std::make_shared<T>(std::move(new_value)));
    std::unique_ptr<node> p(new node);
    node* const new_tail=p.get();
    std::lock_guard<std::mutex> tail_lock(tail_mutex);
    tail->data=new_data;
    tail->next=std::move(p);
    tail=new_tail;
  }
};
```

让我们用挑剔的目光来看一下上面的代码，并考虑6.1.1节中给出的知道意见。在你观察不变量前，你需要确定的状态有：

- tail->next == nullptr

- tail->data == nullptr

- head == taill(意味着空列表)

- 单元素列表 head->next = tail

- 在列表中的每一个节点x，x!=tail且x->data指向一个T类型的实例，并且x->next指向列表中下一个节点。x->next == tail意味着x就是列表中最后一个节点

- 顺着head的next节点找下去，最终会找到tail

这里的push()很简单：仅修改了被tail_mutex的数据，因为新的尾节点是一个空节点，并且其data和next都为旧的尾节点(实际上的尾节点)设置好，所以其能维持不变量的状态。

有趣的部分在于try_pop()上。事实证明，不仅需要对tail_mutex上锁，来保护对tail的读取；还要保证在从头读取数据时，不会产生数据竞争。如果没有这些互斥量，当一个线程调用try_pop()的同时，另一个线程调用push()，那么这里操作顺序将不可预测。尽管，每一个成员函数都持有一个互斥量，这些互斥量能保护数据不会同时被多个线程访问到；并且，队列中的所有数据来源，都是通过调用push()得到的。因为线程可能会无序的方位同一数据，所以这里就会有数据竞争(正如你在第5章看到的那样)，以及未定义行为。幸运的是，在get_tail()中的tail_mutex结局的了所有的问题。因为调用get_tail()将会锁住同名锁，就像push()一样，这就为两个操作规定好了顺序。要不就是get_tail()在push()之前被调用，这种情况下，线程可以看到旧的尾节点，要不就是在push()之后完成，这种情况下，线程就能看到tail的新值，以及新数据前的真正tail的值。

当get_tail()调用前，head_mutex已经上锁，这一步也是很重要的哦。如果不这样，调用pop_head()时就会被get_tail()和head_mutex所卡住，因为其他线程调用try_pop()(以及pop_head())时，都需要先获取锁，然后阻止从下面的过程中初始化线程：

```c++
std::unique_ptr<node> pop_head() // 这是个有缺陷的实现
{
  node* const old_tail=get_tail();  // ① 在head_mutex范围外获取旧尾节点的值
  std::lock_guard<std::mutex> head_lock(head_mutex);

  if(head.get()==old_tail)  // ②
  {
    return nullptr;
  }
  std::unique_ptr<node> old_head=std::move(head);
  head=std::move(old_head->next);  // ③
  return old_head;
}
```

这是一个有缺陷的实现，调用get_tail()是在锁的范围之外，你可能也许会发现head和tail，在你初始化线程，并获取head_mutex时，发生了改变。并且，不只是返回尾节点时，返回的不是尾节点了，其值甚至都不列表中的值了。即使head是最后一个节点，这也意味着head和old_tail②比较失败。因此，当你更新head③时，可能会将head移到tail之后，这样的话就意味着数据结构遭到了破坏。在正确实现中(清单6.6)，需要保证在head_mutex保护的范围内调用get_tail()。这就嫩保证没有其他线程能对head进行修改，并且tail会向正确的方向移动(当有新节点添加进来时)，这样就很安全了。head不会传递给get_tail()的返回值，所以不变量的状态时稳定的。

当使用pop_head()更新head时(从队列中删除节点)，互斥量就已经上锁了，并且try_pop()可以提取数据，并在确实有个数据的时候删除一个节点(若没有数据，则返回`std::shared_ptr<>`的空实例)，因为只有一个线程可以访问这个节点，所以根据我们所掌握的知识，认为这个操作是安全的。

接下来，外部接口就相当于清单6.2代码中的子集了，所以同样的分析结果：对于固有接口来说，不存在条件竞争。

异常是很有趣的东西。虽然，你已经改变了数据的分配模式，但是异常可以从别的地方袭来。try_pop()中的对锁的操作会产生异常，并直到锁获取才能对数据进行修改。因此，try_pop()是异常安全的。另一方面，push()可以在堆上新分配出一个T的实例，以及一个node的新实例，这里可能会抛出异常。但是，所有分配的对象都赋给了智能指针，那么当异常发生时，他们就会被释放掉。一旦锁被获取，push()中的操作就不会抛出异常，所以push()也是异常安全的。

因为你没有修改任何接口，所以他们不会死锁。在实现内部也不会有死锁；唯一需要获取两个锁的是pop_head()，这个函数需要获取head_mutex和tail_mutex，所以不会产生死锁。

那么剩下的问题就都在并发实际的可行性上了。这个结构对并发访问的考虑要多于清单6.2中的代码，因为这里锁粒度更加的小，并且更多的数据不在锁的保护范围内。比如，在push()中，新节点和新数据的分配都不需要锁来保护。这就意味着多线程情况下，节点及数据的分配是“安全”并发的。同一时间内，只有一个线程可以将它的节点和数据添加到队列中，所以代码中只是简单使用了指针赋值的形式，相较于基于`std::queue<>`的实现中，对于`std::queue<>`的内部操作进行上锁，这个结构中就不需要了。

同样，try_pop()持有tail_mutex也只有很短的时间，只为保护对tail的读取。因此，当有数据push进队列后，try_pop()几乎及可以完全并发调用了。同样在执行中，对head_mutex的持有时间也是极短的。当并发访问时，这就会增加对try_pop()的访问次数；且只有一个线程，在同一时间内可以访问pop_head()，且多线程情况下可以删除队列中的旧节点，并且安全的返回数据。

**等待数据弹出**

OK，所以清单6.6提供了一个使用细粒度锁的线程安全队列，不过只有try_pop()可以并发访问(且只有一个重载存在)。那么在清单6.2中方便的wait_and_pop()呢？你能通过细粒度锁实现一个相同功能的接口吗？

当然，答案是“是的”，不过的确有些困难，困难在哪里？修改push()是相对简单的：只需要在函数体末尾添加data_cond.notify_ont()函数的调用即可(如同清单6.2中那样)。当然，事实并没有那么简单：你使用细粒度锁，是为了保证最大程度的并发。当你将互斥量和notify_one()混用的时，如果被通知的线程在互斥量解锁后被唤醒，那么这个线程就不得不等待互斥量上锁。另一方面，当解锁操作在notify_one()之前调用，那么互斥量可能会等待线程醒来，来获取互斥锁(假设没有其他线程对互斥量上锁)。这可能是一个微小的改动，但是对于一些情况来说，就显的很重要了。

wait_and_pop()就有些复杂了，因为需要确定在哪里等待，也就是函数在哪里之心，并且需要确定那些互斥量需要上锁。等待的条件是“队列非空”，这就意味着head!=tail。这样写的话，就需要同时获取head_mutex和tail_mutex，并对其进行上锁，不过你在清单6.6中已经使用tail_mutex来保护对tail的读取，以及不用和自身记性比较，所以这种逻辑也同样适用于这里。如果有函数让head!=get_tail()，你只需要持有head_mutex，然后你就可以使用锁，对data_cond.wait()的调用进行保护。当你将等待逻辑添加入结构当中，那么实现的方式与try_pop()基本上是一样的。

对于try_pop()和wait_and_pop()的重载都需要深思熟虑。当你将返回`std::shared_ptr<>`替换为，从old_head后索引出值，并且拷贝赋值给value参数进行返回，那么这里将会存在异常安全问题。这里，数据项在互斥锁未上锁的情况下被删除；将剩下的数据返回给调用者。不过，当拷贝赋值抛出异常(可能性很大)，数据项将会丢失，因为它没有被返回队列原来的位置上。

当T类型有无异常抛出的移动赋值操作，或无异常抛出的交换操作，你可以使用它，不过，你肯定喜欢一种通用的解决方案，无论T是什么类型，这个方案都能使用。在这种情况下，在节点从列表中删除前，你就不得不将有可能抛出异常的代码，放在锁保护的范围内，来保证异常安全性。这也就意味着你需要对pop_head()进行重载，查找索引值在列表改动前的位置。

相比之下，empty()就更加的简单:只需要锁住head_mutex，并且检查head==get_tail()(详见清单6.10)就可以了。最终的代码，在清单6.7，6.8，6.9和6.10中。

清单6.7 可上锁和等待的线程安全队列——内部机构及接口
```c++
template<typename T>
class threadsafe_queue
{
private:
  struct node
  {
    std::shared_ptr<T> data;
    std::unique_ptr<node> next;
  };

  std::mutex head_mutex;
  std::unique_ptr<node> head;
  std::mutex tail_mutex;
  node* tail;
  std::condition_variable data_cond;
public:
  threadsafe_queue():
    head(new node),tail(head.get())
  {}
  threadsafe_queue(const threadsafe_queue& other)=delete;
  threadsafe_queue& operator=(const threadsafe_queue& other)=delete;

  std::shared_ptr<T> try_pop();
  bool try_pop(T& value);
  std::shared_ptr<T> wait_and_pop();
  void wait_and_pop(T& value);
  void push(T new_value);
  void empty();
};
```

向队列中添加新节点是相当简单的——下面的实现与上面的代码差不多。

清单6.8 可上锁和等待的线程安全队列——推入新节点
```c++
template<typename T>
void threadsafe_queue<T>::push(T new_value)
{
  std::shared_ptr<T> new_data(
  std::make_shared<T>(std::move(new_value)));
  std::unique_ptr<node> p(new node);
  {
    std::lock_guard<std::mutex> tail_lock(tail_mutex);
    tail->data=new_data;
    node* const new_tail=p.get();
    tail->next=std::move(p);
    tail=new_tail;
  }
  data_cond.notify_one();
}
```

如同之前所提到的，复杂部分都在pop那边，所以提供很多帮助性函数去简化这部分就很重要了。下一个清单中将展示wait_and_pop()的实现，以及先关的帮助函数。

清单6.9 可上锁和等待的线程安全队列——wait_and_pop()
```c++
template<typename T>
class threadsafe_queue
{
private:
  node* get_tail()
  {
    std::lock_guard<std::mutex> tail_lock(tail_mutex);
    return tail;
  }

  std::unique_ptr<node> pop_head()  // 1
  {
    std::unique_ptr<node> old_head=std::move(head);
    head=std::move(old_head->next);
    return old_head;
  }

  std::unique_lock<std::mutex> wait_for_data()  // 2
  {
    std::unique_lock<std::mutex> head_lock(head_mutex);
    data_cond.wait(head_lock,[&]{return head.get()!=get_tail();});
    return std::move(head_lock);  // 3
  }

  std::unique_ptr<node> wait_pop_head()
  {
    std::unique_lock<std::mutex> head_lock(wait_for_data());  // 4
    return pop_head();
  }

  std::unique_ptr<node> wait_pop_head(T& value)
  {
    std::unique_lock<std::mutex> head_lock(wait_for_data());  // 5
    value=std::move(*head->data);
    return pop_head();
  }
public:
  std::shared_ptr<T> wait_and_pop()
  {
    std::unique_ptr<node> const old_head=wait_pop_head();
    return old_head->data;
  }

  void wait_and_pop(T& value)
  {
    std::unique_ptr<node> const old_head=wait_pop_head(value);
  }
};
```

清单6.9中所示的pop部分的实现中有一些帮助函数来降低代码的复杂度，例如pop_head()①和wait_for_data()②，这些函数分别是删除头结点和等待队列中有数据弹出的。wait_for_data()特别值得关注，因为其不仅等待使用lambda函数对条件变量进行等待，而且它还会将锁的实例返回给调用者③。这就需要保证，同一个锁在执行与wait_pop_head()重载④⑤，相关的操作时，锁已持有的。pop_head()是对try_pop()代码的复用，将在下面进行展示：

清单6.10 可上锁和等待的线程安全队列——try_pop()和empty()
```c++
template<typename T>
class threadsafe_queue
{
private:
  std::unique_ptr<node> try_pop_head()
  {
    std::lock_guard<std::mutex> head_lock(head_mutex);
    if(head.get()==get_tail())
    {
      return std::unique_ptr<node>();
    }
    return pop_head();
  }

  std::unique_ptr<node> try_pop_head(T& value)
  {
    std::lock_guard<std::mutex> head_lock(head_mutex);
    if(head.get()==get_tail())
    {
      return std::unique_ptr<node>();
    }
    value=std::move(*head->data);
    return pop_head();
  }
public:
  std::shared_ptr<T> try_pop()
  {
    std::unique_ptr<node> old_head=try_pop_head();
    return old_head?old_head->data:std::shared_ptr<T>();
  }

  bool try_pop(T& value)
  {
    std::unique_ptr<node> const old_head=try_pop_head(value);
    return old_head;
  }

  void empty()
  {
    std::lock_guard<std::mutex> head_lock(head_mutex);
    return (head.get()==get_tail());
  }
};
```

这个队列的实现将作为第7章无锁队列的基础。这是一个无界(*unbounded*)队列;线程可以持续向队列中添加数据项，即使没有元素被删除。与之相反的就是有界(*bounded*)队列，在有界队列中，队列在创建的时候最大长度就已经是固定的了。当有界队列满载时，尝试在向其添加元素的操作将会失败或者阻塞，直到有元素从队列中弹出。在任务执行时(详见第8章)，有界队列对于线程间的工作花费是很有帮助的。其会阻止线程对队列进行填充，并且可以避免线程从较远的地方对数据项进行索引。

这里无界队列的实现，很容易扩展成，可在push()中等待跳进变量的定长队列。相对于等待队列中具有数据项(pop()执行完成后)，你就需要等待队列中数据项小于最大值就可以了。对于有界队列更多的讨论，超出了本书的范围，就不在多说；现在越过队列，想想更加复杂的数据结构进发。

##6.3 基于锁设计更加复杂的数据结构

栈和队列都很简单：接口相对固定，并且它们应用于比较特殊的情况。并不是所有数据结构都像它们一样简单；大多数数据结构支持更加多样化的操作。原则上，这将增大并行的可能性，但是也让对数据保护变得更加困难，因为要考虑对所有能访问到的部分。当为了并发访问对数据结构进行设计时，这一系列原有的操作，就变得越发重要，需要重点处理。

先来看看，在查询表的设计中，所遇到的一些问题。

###6.3.1 编写一个使用锁的线程安全查询表

查询表或字典是一种类型的值(键值)和另一种类型的值进行关联(映射的方式)。一般情况下，这样的结构允许代码通过键值对相关的数据值进行查询。在C++标准库中，这种相关工具有：`std::map<>`, `std::multimap<>`, `std::unordered_map<>`以及`std::unordered_multimap<>`。

查询表的使用与栈和队列不同。栈和队列上，几乎每个操作都会对数据结构进行修改，不是添加一个元素，就是删除一个，而对于查询表来说，几乎不需要什么修改。清单3.13中有个例子，是一个简单的域名系统(DNS)缓存，其特点是，相较于`std::map<>`削减了很多的接口。和队列和栈一样，标准容器的接口不适合多线程进行并发访问，因为这些接口在设计的时候都存在固有的条件竞争，所以这些接口需要砍掉，以及重新修订。

并发访问时，`std::map<>`接口最大的问题在于——迭代器。虽然，在多线程访问(或修改)容器时，可能会有提供安全访问的迭代器，但这就问题棘手之处。要想正确的处理迭代器，你可能会碰到下面这个问题：当迭代器引用的元素被其他线程删除时，迭代器在这里就是个问题了。线程安全的查询表，第一次接口削减，需要绕过迭代器。`std::map<>`(以及标准库中其他相关容器)给定的接口对于迭代器的依赖是很严重的，其中有些接口需要先放在一边，先对一些简单接口进行设计。

查询表的基本操作有：

- 添加一对“键值-数据”

- 修改指定键值所对应的数据

- 删除一组值

- 通过给定键值，获取对应数据

容器也有一些操作是非常有用的，比如：查询容器是否为空，键值列表的完整快照和“键值-数据”的完整快照。

如果你坚持之前的简单线程安全指导意见，例如：不要返回一个引用，并且用一个简单的互斥锁对每一个成员函数进行上锁，以确保每一个函数线程安全。最有可能的条件竞争在于，当一对“键值-数据”加入时；当两个线程都添加一个数据，那么肯定一个先一个后。一种方式是合并“添加”和“修改”操作，为一个成员函数，就像清单3.13对域名系统缓存所做的那样。

从接口角度看，有一个问题很是有趣，那就是任意(*if any*)部分获取相关数据。一种选择是允许用户提供一个“默认”值，在键值没有对应值的时候进行返回：

```c++
mapped_type get_value(key_type const& key, mapped_type default_value);
```

在种情况下，当default_value没有明确的给出时，默认构造出的mapped_type实例将被使用。也可以扩展成返回一个`std::pair<mapped_type, bool>`来代替mapped_type实例，其中bool代表返回值是否是当前键对应的值。另一个选择是，返回一个有指向数据的智能指针；当指针的值是NULL时，那么这个键值就没有对应的数据。

如我们之前所提到的，当接口确定时，那么(假设没有接口间的条件竞争)就需要保证线程安全了，可以通过对每一个成员函数使用一个互斥量和一个简单的锁，来保护底层数据。不过，当独立的函数对数据结构进行读取和修改时，就会降低并发的可能性。一个选择是使用一个互斥量去面对多个读者线程，或一个作者线程，如同在清单3.13中对`boost::shared_mutex`的使用一样。虽然，这将提高并发访问的可能性，但是在同一时间内，也只有一个线程能对数据结构进行修改。理想很美好，现实很骨感？我们应该能做的更好！

**为细粒度锁设计一个映射结构**

在对队列的讨论中(在6.2.3节)，为了允许细粒度锁能正常工作，需要对于数据结构的细节进行仔细的考虑，而非直接使用已存在的容器，例如`std::map<>`。这里列出三个常见关联容器的方式：

- 二叉树，比如：红黑树

- 有序数组

- 哈希表

二叉树的方式，不会对提高并发访问的概率；每一个查找或者修改操作都需要访问根节点，因此，根节点需要上锁。虽然，访问线程在向下移动时，这个锁可以进行释放，但相比横跨整个数据结构的单锁，并没有什么优势。

有序数组是最坏的选择，因为你无法提前言明数组中哪段是有序的，所以你需要用一个锁将整个数组锁起来。

那么就剩哈希表了。假设有固定数量的桶，每个桶都有一个键值(关键特性)，以及散列函数。这就意味着你可以安全的对每个桶上锁。当你再次使用互斥量(支持多读者单作者)时，你就能将并发访问的可能性增加N倍，这里N是桶的数量。当然，缺点也是有的：对于键值的操作，需要有合适的函数。C++标准库提供`std::hash<>`模板，可以直接使用。对于特化的类型，比如int，以及通用库类型`std::string`，并且用户可以简单的对键值类型进行特化。如果你去效仿标准无序容器，并且获取函数对象的类型作为哈希表的模板参数，用户可以选择是否特化`std::hash<>`的键值类型，或者提供一个独立的哈希函数。

那么，让我们来看一些代码吧。怎样的实现才能完成一个线程安全的查询表？下面就是一种方式。

清单6.11 线程安全的查询表
```c++
template<typename Key,typename Value,typename Hash=std::hash<Key> >
class threadsafe_lookup_table
{
private:
  class bucket_type
  {
  private:
    typedef std::pair<Key,Value> bucket_value;
    typedef std::list<bucket_value> bucket_data;
    typedef typename bucket_data::iterator bucket_iterator;

    bucket_data data;
    mutable boost::shared_mutex mutex;  // 1

    bucket_iterator find_entry_for(Key const& key) const  // 2
    {
      return std::find_if(data.begin(),data.end(),
      [&](bucket_value const& item)
      {return item.first==key;});
    }
  public:
    Value value_for(Key const& key,Value const& default_value) const
    {
      boost::shared_lock<boost::shared_mutex> lock(mutex);  // 3
      bucket_iterator const found_entry=find_entry_for(key);
      return (found_entry==data.end())?
        default_value:found_entry->second;
    }

    void add_or_update_mapping(Key const& key,Value const& value)
    {
      std::unique_lock<boost::shared_mutex> lock(mutex);  // 4
      bucket_iterator const found_entry=find_entry_for(key);
      if(found_entry==data.end())
      {
        data.push_back(bucket_value(key,value));
      }
      else
      {
        found_entry->second=value;
      }
    }

    void remove_mapping(Key const& key)
    {
      std::unique_lock<boost::shared_mutex> lock(mutex);  // 5
      bucket_iterator const found_entry=find_entry_for(key);
      if(found_entry!=data.end())
      {
        data.erase(found_entry);
      }
    }
  };

  std::vector<std::unique_ptr<bucket_type> > buckets;  // 6
  Hash hasher;

  bucket_type& get_bucket(Key const& key) const  // 7
  {
    std::size_t const bucket_index=hasher(key)%buckets.size();
    return *buckets[bucket_index];
  }

public:
  typedef Key key_type;
  typedef Value mapped_type;

  typedef Hash hash_type;
  threadsafe_lookup_table(
    unsigned num_buckets=19,Hash const& hasher_=Hash()):
    buckets(num_buckets),hasher(hasher_)
  {
    for(unsigned i=0;i<num_buckets;++i)
    {
      buckets[i].reset(new bucket_type);
    }
  }

  threadsafe_lookup_table(threadsafe_lookup_table const& other)=delete;
  threadsafe_lookup_table& operator=(
    threadsafe_lookup_table const& other)=delete;

  Value value_for(Key const& key,
                  Value const& default_value=Value()) const
  {
    return get_bucket(key).value_for(key,default_value);  // 8
  }

  void add_or_update_mapping(Key const& key,Value const& value)
  {
    get_bucket(key).add_or_update_mapping(key,value);  // 9
  }

  void remove_mapping(Key const& key)
  {
    get_bucket(key).remove_mapping(key);  // 10
  }
};
```

这个实现中使用了`std::vector<std::unique_ptr<bucket_type>>`⑥来保存桶，其允许在构造函数中指定构造桶的数量。默认为19个，其是一个任意的[质数](http://zh.wikipedia.org/zh-cn/%E7%B4%A0%E6%95%B0);哈希表在有质数个桶时，工作效率最高。每一个桶都会被一个`boost::shared_mutex`①实例锁保护，来允许并发读取，或对每一个桶，只有一个线程对其进行修改。

因为桶的数量是固定的，所以get_bucket()⑦可以无锁调用，⑧⑨⑩也都一样。并且对桶的互斥量上锁，要不就是共享(只读)所有权的时候③，要不就是在获取唯一(读/写)权的时候④⑤。这里的互斥量，可适用于每个成员函数。

这三个函数都使用到了find_entry_for()成员函数②，在桶上用来确定数据是否在桶中。每一个桶都包含一个“键值-数据”的`std::list<>`列表，所以添加和删除数据，就会很简单。

已经从并发的角度考虑了，并且所有成员都会被互斥锁保护，所以这样的实现就是“异常安全”的吗？value_for是不能修改任何值的，所以其不会有问题；如果value_for抛出异常，也不会对数据结构有任何影响。remove_mapping修改链表时，将会调用erase，不过这就能保证没有异常抛出，那么这里也是安全的。那么就剩add_or_update_mapping了，其可能会在其两个if分支上抛出异常。push_back是异常安全的，如果有异常抛出，其也会将链表恢复成原来的状态，所以这个分支是没有问题的。唯一的问题就是在赋值阶段，这将替换已有的数据；当复制阶段抛出异常，用于原依赖的始状态没有改变。不过，这不会影响数据结构的整体，以及用户提供类型的属性，所以你可以放心的将问题交给用户处理。

在本节开始时，我提到查询表的一个“可有可无”(*nice-to-have*)的特性，会将选择当前状态的快照，例如，一个`std::map<>`。这将要求锁住整个容器，用来保证拷贝副本的状态是可以索引的，这将要求锁住所有的桶。因为对于查询表的“普通”的操作，需要在同一时间获取一个桶上的一个锁，而这个操作将要求查询表将所有桶都锁住。因此，只要每次以相同的顺序进行上锁(例如，递增桶的索引值)，就不会产生死锁。实现如下所示：

清单6.12 获取整个threadsafe_lookup_table作为一个`std::map<>`
```c++
std::map<Key,Value> threadsafe_lookup_table::get_map() const
{
  std::vector<std::unique_lock<boost::shared_mutex> > locks;
  for(unsigned i=0;i<buckets.size();++i)
  {
    locks.push_back(
      std::unique_lock<boost::shared_mutex>(buckets[i].mutex));
  }
  std::map<Key,Value> res;
  for(unsigned i=0;i<buckets.size();++i)
  {
    for(bucket_iterator it=buckets[i].data.begin();
        it!=buckets[i].data.end();
        ++it)
    {
      res.insert(*it);
    }
  }
  return res;
}
```

清单6.11中的查询表实现，就增大的并发访问的可能性，这个查询表作为一个整体，通过单独的操作，对每一个桶进行锁定，并且通过使用`boost::shared_mutex`允许读者线程对每一个桶进行并发访问。如果细粒度锁和哈希表结合起来，会更有效的增加并发的可能性吗？

在下一节中，你将使用到一个线程安全列表(支持迭代器)。

###6.3.2 编写一个使用锁的线程安全链表

链表类型是数据结构中的一个基本类型，所以应该是比较好修改成线程安全的，对么？其实这取决于你要添加什么样的功能，这其中需要你提供迭代器的支持，为了让基本数据类型的代码不会太复杂，我去掉了一些功能。迭代器的问题在于，STL类的迭代器需要持有容器内部属于的引用。当容器可被其他线程修改时，有时这个引用还是有效的；实际上，这里就需要迭代器持有锁，对指定的结构中的部分进行上锁。在给定STL类迭代器的生命周期中，让其完全脱离容器的控制是很糟糕的。

替代方案就是提供迭代函数，例如，将for_each作为容器本身的一部分。这就能让容器来对迭代的部分进行负责和锁定，不过这将违反第3章指导意见对避免死锁建议。为了让for_each在任何情况下都有用，在其持有内部锁的时候，必须调用用户提供的代码。不仅如此，而且需要传递一个对容器中元素的引用到用户代码中，为的就是让用户代码对容器中的元素进行操作。你可以为了避免传递引用，而传出一个拷贝到用户代码中；不过当数据很大时，拷贝所要付出的代价也很大。

所以，可以将避免死锁的工作(因为用户提供的操作需要获取内部锁)，还有避免对引用(不被锁保护)进行存储时的条件竞争，交给用户去做。这样的链表就可以被查询表所使用了，这样很安全，因为你知道这里的实现不会有任何问题。

那么剩下的问题就是哪些操作需要列表所提供。如果你愿在花点时间看一下清单6.11和6.12中的代码，你会看到下面这些操作是需要的：

- 向列表添加一个元素

- 当某个条件满足时，就从链表中删除某个元素

- 当某个条件满足时，从链表中查找某个元素

- 当某个条件满足时，更新链表中的某个元素

- 将当前容器中链表中的每个元素，复制到另一个容器中

提供了这些操作，我们的链表才能是一个比较好的通用容器，这将帮助我们添加更多功能，比如，在指定位置上插入元素，不过这对于我们查询表来说就没有必要了，所以这里就算是给读者们留一个作业吧。

使用细粒度锁最初的想法，是为了让链表每个节点都拥有一个互斥量。当链表很长时，那么就会有很多的互斥量!这样的好处是对于链表中每一个独立的部分，都能实现真实的并发：其真正感兴趣的是对持有的节点群进行上锁，并且在移动到下一个节点的时，对当前节点进行释放。下面的清单中将展示这样的一个链表实现。

清单6.13 线程安全链表——支持迭代器
```c++
template<typename T>
class threadsafe_list
{
  struct node  // 1
  {
    std::mutex m;
    std::shared_ptr<T> data;
    std::unique_ptr<node> next;
    node():  // 2
      next()
    {}

    node(T const& value):  // 3
      data(std::make_shared<T>(value))
    {}
  };

  node head;

public:
  threadsafe_list()
  {}

  ~threadsafe_list()
  {
    remove_if([](node const&){return true;});
  }

  threadsafe_list(threadsafe_list const& other)=delete;
  threadsafe_list& operator=(threadsafe_list const& other)=delete;

  void push_front(T const& value)
  {
    std::unique_ptr<node> new_node(new node(value));  // 4
    std::lock_guard<std::mutex> lk(head.m);
    new_node->next=std::move(head.next);  // 5
    head.next=std::move(new_node);  // 6
  }

  template<typename Function>
  void for_each(Function f)  // 7
  {
    node* current=&head;
    std::unique_lock<std::mutex> lk(head.m);  // 8
    while(node* const next=current->next.get())  // 9
    {
      std::unique_lock<std::mutex> next_lk(next->m);  // 10
      lk.unlock();  // 11
      f(*next->data);  // 12
      current=next;
      lk=std::move(next_lk);  // 13
    }
  }

  template<typename Predicate>
  std::shared_ptr<T> find_first_if(Predicate p)  // 14
  {
    node* current=&head;
    std::unique_lock<std::mutex> lk(head.m);
    while(node* const next=current->next.get())
    {
      std::unique_lock<std::mutex> next_lk(next->m);
      lk.unlock();
      if(p(*next->data))  // 15
      {
         return next->data;  // 16
      }
      current=next;
      lk=std::move(next_lk);
    }
    return std::shared_ptr<T>();
  }

  template<typename Predicate>
  void remove_if(Predicate p)  // 17
  {
    node* current=&head;
    std::unique_lock<std::mutex> lk(head.m);
    while(node* const next=current->next.get())
    {
      std::unique_lock<std::mutex> next_lk(next->m);
      if(p(*next->data))  // 18
      {
        std::unique_ptr<node> old_next=std::move(current->next);
        current->next=std::move(next->next);
        next_lk.unlock();
      }  // 20
      else
      {
        lk.unlock();  // 21
        current=next;
        lk=std::move(next_lk);
      }
    }
  }
};
```

清单6.13中的threadsafe_list<>是一个单链表，可从node的结构①中看出。一个默认构造的node，作为链表的head，其next指针②指向的是NULL。新节点都是被push_front()函数添加进去的；构造第一个新节点④，其将会在堆上分配内存③来对数据进行存储，同时将next指针置为NULL。然后，你需要获取head节点的互斥锁，为了让设置next的值⑤，也就是插入节点到列表的头部，让头节点的head.next指向这个新节点⑥。目前，还没有什么问题：你只需要锁住一个互斥量，就能将一个新的数据添加进入链表，所以这里不存在死锁的问题。同样，(缓慢的)内存分配操作在锁的范围外，所以锁能保护需要更新的一对指针。那么，现在来看一下迭代功能。

首先，来看一下for_each()⑦。这个操作需要对队列中的每个元素执行Function(函数指针)；在大多数标准算法库中，都会通过传值方式来执行这个函数，这里要不就传入一个通用的函数，要不就传入一个有函数操作的类型对象。在这种情况下，这个函数必须接受类型为T的值作为参数。在链表中，会有一个“手递手”的上锁过程。在这个过程开始时，你需要锁住head及节点⑧的互斥量。然后，安全的获取指向下一个节点的指针(使用get()获取，这是因为你对这个指针没有所有权)。当指针不为NULL⑨，就需要对指向的节点进行上锁⑩，为了继续对数据进行处理。当你已经锁住了那个节点，就可以对上一个节点进行释放了⑪，并且调用指定函数⑫。当函数执行完成时，你就可以更新当前指针所指向的节点(刚刚处理过的节点)，并且将所有权从next_lk移动移动到lk⑬。因为for_each传递的每个数据都是能被Function接受的，所以当需要的时，需要拷贝到另一个容器的时，或其他情况时，你都可以考虑使用这种方式更新每个元素。如果函数的行为没什么问题，这种方式是完全安全的，因为在获取节点互斥锁时，已经获取锁的节点正在被函数所处理。

find_first_if()⑭和for_each()很相似；最大的区别在于find_first_if支持函数(谓词)在匹配的时候返回true，在不匹配的时候返回false⑮。当条件匹配，只需要返回找到的数据⑯，而非继续查找。你可以使用for_each()来做这件事，不过在找到之后，继续做查找就是没有意义的了。

remove_if()⑰就有些不同了，因为这个函数会改变链表；所以，你就不能使用for_each()来实现这个功能。当函数(谓词)返回true⑱，对应元素将会移除，并且更新current->next⑲。当这些都做完，你就可以释放next指向节点的锁。当`std::unique_ptr<node>`的移动超出链表范围⑳，这个节点将被删除。这种情况下，你就不需要更新当前节点了，因为你只需要修改next所指向的下一个节点就可以。当函数(谓词)返回false，那么移动的操作就和之前一样了(21)。

那么，所有的互斥量中会有死锁或条件竞争吗？答案无疑是“否”，这里要看提供的函数(谓词)是否有良好的行为。迭代通常都是使用一种方式，都是从head节点开始，并且在释放当前节点锁之前，将下一个节点的互斥量锁住，所以这里就不可能会有不同线程有不同的上锁顺序。唯一可能出现条件竞争的地方就是在remove_if()⑳中删除已有节点的时候。因为，这个操作在解锁互斥量后进行(其导致的未定义行为，可对已上锁的互斥量进行破坏)。不过，在考虑一阵后，可以确定这的确是安全的，因为你还持有前一个节点(当前节点)的互斥锁，所以不会有新的线程尝试去获取你正在删除的那个节点的互斥锁。

这里并发概率有多大呢？细粒度锁要比单锁的并发概率大很多，那我们已经获得了吗？是的，你已经获取了：同一时间内，不同线程可以在不同节点上工作，无论是其使用for_each()对每一个节点进行处理，使用find_first_if()对数据进行查找，还是使用remove_if()删除一些元素。不过，因为互斥量必须按顺序上锁，那么线程就不能交叉进行工作。当一个线程耗费大量的时间对一个特殊节点进行处理，那么其他线程就必须等待这个处理完成。在完成后，其他线程才能到达这个节点。

##6.4 小结

本章开篇，我们讨论了设计并发数据结构的意义，以及给出了一些指导意见。然后，通过设计一些通用的数据结构(栈，队列，哈希表和单链表)，探究了在指导意见在实现这些数据结构的应用，并使用锁来保护数据和避免数据竞争。那么现在，你应该回看一下本章实现的那些数据结构，再回顾一下如何增加并发访问的几率，和哪里会存在潜在条件竞争。

在第7章中，我们将看一下如何避免锁完全锁定，使用底层原子操作来提供必要访问顺序约束的同时，遵循本章的指导意见。