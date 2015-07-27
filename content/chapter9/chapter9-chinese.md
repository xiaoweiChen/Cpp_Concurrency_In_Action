#第9章 高级线程管理

**本章主要内容**

- 线程池<br>
- 处理线程池中任务的依赖关系<br>
- 池中线程如何获取任务<br>
- 中断线程<br>

在之前的章节中，我们通过创建`std::thread`对象来对每一个线程进行管理。在一些地方，你已经看到这种方式是不可取的了，因为需要在线程的整个生命周期中对其进行管理，根据当前使用的硬件来决定恰当的线程数量，等等。理想情况是将代码划分为最小块，以便能并发执行，然后将他们传递给处理器和标准库，进行性能的并行优化。

另一种情况是，当你使用多线程来解决某个问题时，当某个条件达成的时候，就可以提前结束。这可能是因为结果已经确定好了，或者因为发生了错误，亦或者是用户显式的终止操作。无论是哪种原因，线程都需要发送“请停止”的请求，放弃给定的任务，清理，然后尽快停止运行。

在本章，我们将了解一下管理线程和任务的机制，从自动管理线程数量和自自动管理划分任务开始。

##9.1 线程池

在很多公司里面，雇员通常会在办公室度过他们的办公时光(偶尔也会外出访问客户或供应商)，或是参加贸易展会。虽然这些路程可能很有必要，并且可能需要很多人一起去，不过对于一些比较特别的雇员来说，这一去就是几个月，甚至是几年。公司要给每个雇员都配一辆车，这基本上是不可能的，不过公司可以提供一些共用车辆；这样就会有一定数量车，来让所有雇员使用。当一个员工要去异地旅游时，那么他就可以从共用车辆中预定一辆，并在返回公司的时候将车交还。如果有一天共用车辆没有闲置的，雇员就不得不将其旅程后延了。

线程池就是类似的一种方式。在大多数系统中，将每个任务制定给一个线程是不切实际的，不过可以利用现有的并发可能，进行并发。线程池就提供了这样的功能；提交到线程池中的任务将会被并发执行，提交的任务将会挂在任务队列上。队列中的每一个任务都会被池中的一个工作线程所获取，当任务执行完成后，再回到线程池中获取下一个任务。

创建一个线程池的时候，会遇到几个关键的设计问题，比如，可使用的线程数量，高效的任务分配方式，以及是否需要等待一个任务完成。在本节，我们将看到很多线程池的实现是如何解决这些问题的，让我们从最简单的线程池开始吧！

###9.1.1 最简单的线程池

作为最简单的线程池，其拥有固定数量的工作线程(通常工作线程的数量是与`std::thread::hardware_concurrency()`相同的)。当有工作需要完成，可以调用函数将任务挂在任务队列中。每个工作线程都会从任务队列上获取任务，然后执行这个任务，执行完成后再回来获取新的任务。在最简单的情况下，线程就不需要等待其他线程完成对应任务了。如果需要等待，那么就需要对同步进行管理。

下面清单中的代码就展示了一个最简单的线程池实现。

清单9.1 简单的线程池
```c++
class thread_pool
{
  std::atomic_bool done;
  thread_safe_queue<std::function<void()> > work_queue;  // 1
  std::vector<std::thread> threads;  // 2
  join_threads joiner;  // 3

  void worker_thread()
  {
    while(!done)  // 4
    {
      std::function<void()> task;
      if(work_queue.try_pop(task))  // 5
      {
        task();  // 6
      }
      else
      {
        std::this_thread::yield();  // 7
      }
    }
  }

public:
  thread_pool():
    done(false),joiner(threads)
  {
    unsigned const thread_count=std::thread::hardware_concurrency();  // 8

    try
    {
      for(unsigned i=0;i<thread_count;++i)
      {
        threads.push_back( 
          std::thread(&thread_pool::worker_thread,this));  // 9
      }
    }
    catch(...)
    {
      done=true;  // 10
      throw;
    }
  }

  ~thread_pool()
  {
    done=true;  // 11
  }

  template<typename FunctionType>
  void submit(FunctionType f)
  {
    work_queue.push(std::function<void()>(f));  // 12
  }
};
```

这个实现中有一组工作线程②，并且使用了一个线程安全队列(从第6章)①来管理任务队列。这种情况下，用户不用等待任务，并且这些函数不需要返回任何值，所以这里可以使用`std::function<void()>`来封装任务。submit()函数会对函数或可调用对象包装成一个`std::function<void()>`实例，并将其推入队列中⑫。

线程始于构造函数：使用`std::thread::hardware_concurrency()`来获取硬件支持多少个并发线程⑧，折现线程会在worker_thread()成员函数中执行⑨。

当有异常抛出，线程启动就会失败，所以需要保证任何已启动的线程都会停止，并且能在这种情况下清理的十分干净。当有异常抛出时，通过使用*try-catch*来设置done标识⑩，还有join_threads类的实例(来自于第8章)③用来汇聚所有线程。这里当然也需要析构函数：仅设置done标识⑪，并且join_threads可以确保所有线程在线程池销毁前，全部执行完成。注意成员声明的顺序很重要：done标识和worker_queue必须在threads数组之前声明，而这个数据必须在joiner前声明。这就能确保成员能以正确的顺序销毁；比如，所有线程都停止运行时，队列就可以安全的销毁了。

worker_thread函数本身就很简单：就是执行一个循环直到done标识被设置④，从任务队列上获取任务⑤，以及同时执行这些任务⑥。如果任务队列上没有任务，函数会调用`std::this_thread::yield()`进行休息⑦，并且给予其他线程向任务队列上推送任务的机会。

对于一些简单的情况，这样线程池就足以满足要求，特别是任务完全独立没有任何返回值，或需要执行一些阻塞操作的。不过在很多情况下，这样简单的线程池是完全不够用的，还有其他情况使用这样简单的线程池可能会出现问题，比如：死锁。同样，在简单例子中，使用`std::async`能提供更好的功能，如第8章中的那些例子一样。在本章中，我们将了解一下更加复杂的线程池实现，通过添加性能满足用户需求，或减少问题的发生几率。首先，从已经提交的任务开始说起。

###9.1.2 等待提交到线程池中的任务

在第8章中的例子中，在线程间的任务划分完成后，代码会显式生成新线程，而主线程通常就是等待新生成的线程结束，确保在返回调用之前，所有任务都完成了。使用线程池，就需要等待任务提交到线程池中，而非直接提交给单个线程。这与基于`std::async`的方法(第8章等待future的例子)类似，使用清单9.1中的简单线程池，这里必须使用第4章中提到的技术：条件变量和future。这就会增加代码的复杂度；不过，这要比直接对任务进行等待的方式好的多。

通过增加线程池的复杂度，你可以直接等待任务完成。可以使用submit()函数返回一个对任务描述的句柄，并用来等待任务的完成。任务句柄会用条件变量或future进行包装，这样使用线程池来简化代码。

一种特殊的情况就是，执行任务的线程需要返回一个结果到主线程上进行处理。你已经在本书中看到多这样的例子，比如parallel_accumulate()函数(第2章)。在这种情况下，需要使用future来对最终的结果进行转移。清单9.2就展示了对简单线程池的一些修改，这样就能等待任务完成，以及在在工作线程完成后，返回一个结果到等待线程中去不过`std::packaged_task<>`实例是不可拷贝的，仅是可移动的，所以这里不能再使用`std::function<>`来实现任务队列，因为`std::function<>`需要存储可复制构造的函数对象。或者，包装一个自定义函数，用来处理只可移动的类型。这就是一个带有函数操作符的类型擦除类。只需要处理那些没有函数和无返回的函数，所以这是一个简单的虚函数调用。

清单9.2 可等待任务的线程池
```c++
class function_wrapper
{
  struct impl_base {
    virtual void call()=0;
    virtual ~impl_base() {}
  };

  std::unique_ptr<impl_base> impl;
  template<typename F>
  struct impl_type: impl_base
  {
    F f;
    impl_type(F&& f_): f(std::move(f_)) {}
    void call() { f(); }
  };
public:
  template<typename F>
  function_wrapper(F&& f):
    impl(new impl_type<F>(std::move(f)))
  {}

  void operator()() { impl->call(); }

  function_wrapper() = default;

  function_wrapper(function_wrapper&& other):
    impl(std::move(other.impl))
  {}
 
  function_wrapper& operator=(function_wrapper&& other)
  {
    impl=std::move(other.impl);
    return *this;
  }

  function_wrapper(const function_wrapper&)=delete;
  function_wrapper(function_wrapper&)=delete;
  function_wrapper& operator=(const function_wrapper&)=delete;
};

class thread_pool
{
  thread_safe_queue<function_wrapper> work_queue;  // 使用function_wrapper，而非使用std::function

  void worker_thread()
  {
    while(!done)
    {
      function_wrapper task;
      if(work_queue.try_pop(task))
      {
        task();
      }
      else
      {
        std::this_thread::yield();
      }
    }
  }
public:
  template<typename FunctionType>
  std::future<typename std::result_of<FunctionType()>::type>  // 1
    submit(FunctionType f)
  {
    typedef typename std::result_of<FunctionType()>::type
      result_type;  // 2
    
    std::packaged_task<result_type()> task(std::move(f));  // 3
    std::future<result_type> res(task.get_future());  // 4
    work_queue.push(std::move(task));  // 5
    return res;  // 6
  }
  // 休息一下
};
```

首先，修改的是submit()函数①返回一个`std::future<>`保存任务的返回值，并且允许调用者等待任务完全结束。这就需要你知道提供函数f的返回类型，这就是使用`std::result_of<>`的原因：`std::result_of<FunctionType()>::type`是FunctionType类型的引用实例(如f)，并且没有参数。同样，在函数中可以对result_type typedef②使用`std::result_of<>`。

然后，将f包装入`std::packaged_task<result_type()>`③，因为f是一个无参数的函数或是可调用对象，能够返回result_type类型的实例。在向任务队列推送任务⑤和返回future⑥前，就可以从`std::packaged_task<>`中获取future了④。注意，要将任务推送到任务队列中时，只能使用`std::move()`，因为`std::packaged_task<>`是不拷贝的。为了对任务进行处理，队列里面存的就是function_wrapper对象，而非`std::function<void()>`对象。

现在线程池允许你等待任务，并且返回任务后的结果。下面的清单就展示了，如何让parallel_accumuate函数使用线程池。

清单9.3 parallel_accumulate使用一个可等待任务的线程池
```c++
template<typename Iterator,typename T>
T parallel_accumulate(Iterator first,Iterator last,T init)
{
  unsigned long const length=std::distance(first,last);
  
  if(!length)
    return init;

  unsigned long const block_size=25;
  unsigned long const num_blocks=(length+block_size-1)/block_size;  // 1

  std::vector<std::future<T> > futures(num_blocks-1);
  thread_pool pool;

  Iterator block_start=first;
  for(unsigned long i=0;i<(num_blocks-1);++i)
  {
    Iterator block_end=block_start;
    std::advance(block_end,block_size);
    futures[i]=pool.submit(accumulate_block<Iterator,T>());  // 2
    block_start=block_end;
  }
  T last_result=accumulate_block<Iterator,T>()(block_start,last);
  T result=init;
  for(unsigned long i=0;i<(num_blocks-1);++i)
  {
    result+=futures[i].get();
  }
  result += last_result;
  return result;
}
```

与清单8.4相比，有几个点需要注意一下。首先，工作量是依据使用的块数(num_blocks①)，而不是线程的数量。为了利用线程池的最大化可扩展性，需要将工作块划分为最小工作块，这是为了让工作并发起来。当线程池中线程不多时，每个线程将会处理多个工作块，不过随着硬件可用线程数量的增长，会有越来越多的工作块并发执行。

当你选择“因为能并发执行，最小工作块值的一试”时，就需要谨慎了。向线程池提交一个任务是有一定的开销的，让工作线程执行这个任务，并且将返回值保存在`std::future<>`中，对于太小的任务，这样的开销是不划算的。如果你的任务块太小，使用线程池的速度可能都不及单线程的速度。

假设，任务块的大小是合理的，那么就不用为这些事而担心：打包任务、获取future或存储之后要汇入的`std::thread`对象；在使用线程池的时候，这些都需要注意。之后，你说要做的就是调用submit()来提交你的任务②。

线程池也需要注意异常安全。任何异常都会通过submit()返回给future，并在获取future的结果时，抛出异常。如果函数因为异常退出，线程池的析构函数丢掉那些没有完成的任务，等待线程池中的工作线程完成工作。

在简单的例子中这个线程池工作的还算不错，因为这里的任务都是相互独立的。不过，当任务队列中的任务有依赖关系时，这个线程池就不能胜任工作了。

###9.1.3 等待依赖任务

以之前的快速排序算法为例，原理很简单：数据与中轴数据项比较，在中轴项两侧分为大于和小于的两个序列，然后再对这两组序列进行排序。这两组序列会递归排序，最后会整合成一个全排序序列。要将这个算法写成并发模式，需要保证递归调用能够使用硬件的并发能力。

回到第4章，我们第一次接触这个例子，我们使用`std::async`来执行每一层的调用，让标准库来选择，是在新线程上执行这个任务，还是当对应get()调用时，同步执行。工作起来很不错，因为诶一个任务都在其自己的线程上执行，或当需要的时候进行调用。

当我们回顾第8章时，使用了一个使用固定数量线程(根据硬件可用并发线程数)的结构体。在这样的情况下，使用了栈来挂起要排序的数据块。当每个线程在为一个数据块排序前，会向数据栈上添加一组要排序的数据，然后在对当前数据块排序结束后，直接对另一块进行排序。这里，等待其他线程完成排序，可能会造成死锁，因为这会消耗有限的线程。有一种情况很可能会出现，那就是所有线程都在等某一个数据块被排序，不过没有线程在做排序。我们通过拉取栈上数据块的线程，并对数据块进行排序，来解决这个问题；因为，这个已处理的特殊数据块，就是其他线程都在等待排序数据块。

如果只是用简单的线程池进行替换，例如第4章替换`std::async`的那个线程池。现在只有限制数量的线程，因为线程池中没有空闲的线程，线程会等待没有被安排的任务。因此，需要一个和第8章中类似的解决方案：当等待某个数据块完成时，去处理未完成的数据块。如果是使用线程池来管理任务列表和相关线程——要使用线程池的主要原因——你就不用再去访问任务列表了。可以对线程池做一些改动，让其来自动完成这些事情。

最简单的方法就是在thread_pool中添加一个新函数，来执行任务队列上的任务，并对线程池进行管理。高级线程的实现可能会在等待函数中添加逻辑，或等待其他函数来处理这个任务，优先的任务会让其他的任务进行等待。下面清单中的实现，就展示了一个新的run_pending_task()函数，对于快速排序的修改将会在清单9.5中展示。

清单9.4 run_pending_task()函数实现
```c++
void thread_pool::run_pending_task()
{
  function_wrapper task;
  if(work_queue.try_pop(task))
  {
    task();
  }
  else
  {
    std::this_thread::yield();
  }
}
```

run_pending_task()的实现去掉了在worker_thread()函数中的主循环。这个函数就是在任务队列中有任务的时候，执行任务；要是没有的话，就会让操作系统对线程重新进行分配。下面对快速排序算法的实现要比清单8.1中版本简单许多，这是因为所有线程管理逻辑都被移入线程池中了。

清单9.5 基于线程池的快速排序实现
```c++
template<typename T>
struct sorter  // 1
{
  thread_pool pool;  // 2

  std::list<T> do_sort(std::list<T>& chunk_data)
  {
    if(chunk_data.empty())
    {
      return chunk_data;
    }

    std::list<T> result;
    result.splice(result.begin(),chunk_data,chunk_data.begin());
    T const& partition_val=*result.begin();
    
    typename std::list<T>::iterator divide_point=
      std::partition(chunk_data.begin(),chunk_data.end(),
                     [&](T const& val){return val<partition_val;});

    std::list<T> new_lower_chunk;
    new_lower_chunk.splice(new_lower_chunk.end(),
                           chunk_data,chunk_data.begin(),
                           divide_point);

    std::future<std::list<T> > new_lower=  // 3
      pool.submit(std::bind(&sorter::do_sort,this,
                            std::move(new_lower_chunk)));

    std::list<T> new_higher(do_sort(chunk_data));

    result.splice(result.end(),new_higher);
    while(!new_lower.wait_for(std::chrono::seconds(0)) ==
      std::future_status::timeout)
    {
      pool.run_pending_task();  // 4
    }

    result.splice(result.begin(),new_lower.get());
    return result;
  }
};

template<typename T>
std::list<T> parallel_quick_sort(std::list<T> input)
{
  if(input.empty())
  {
    return input;
  }
  sorter<T> s;

  return s.do_sort(input);
}
```

和清单8.1相比，这里将实际工作放在sorter类模板的do_sort()成员函数中执行①，即使在这个例子中仅对thread_pool实例进行包装②。

你的线程和任务管理，就会减少向线程池中提交一个任务③，并且在线程等待的时候，执行任务队列上未完成的任务④。这里的实现要比清单8.1简单许多，这里需要显式的管理线程和栈上要排序的数据块。当有任务提交到线程池中，可以使用`std::bind()`绑定this指针到do_sort()上，绑定是为了让数据块进行排序。在这种情况下，需要对new_lower_chunk使用`std::move()`来将其传入函数，数据移动要比拷贝的方式开销少。

虽然，现在使用等待其他任务的方式，解决了死锁问题，这个线程池距离理想的线程池很远。首先，每次对submit()的调用和对run_pending_task()的调用，访问的都是同一个队列。在第8章中，我们了解过，当多线程去修改一组数据，就会对性能有所影响，所以需要来解决这个问题。

###9.1.4 避免队列中的任务竞争

线程每次调用线程池实例的submit()函数，都会推送一个任务到工作队列中。就像工作线程为了执行任务，从任务队列中获取任务。这就意味着随着处理器的增加，就会在任务队列上有很多的竞争。这会让性能下降；使用无锁队列会让任务没有明显的等待，但是乒乓缓存会消耗大量的时间。

一种为了避免乒乓缓存的方式，是为每个线程建立独立的任务队列。这样，每个线程就会将新任务放在自己的任务队列上，并且当线程上的任务队列没有任务时，回去全局的任务列表中取任务。下面列表中的实现，使用了一个thread_local变量，来保证每个线程都拥有自己的任务列表(如全局列表那样)。

清单9.6 线程池——线程具有本地任务队列
```c++
class thread_pool
{
  thread_safe_queue<function_wrapper> pool_work_queue;

  typedef std::queue<function_wrapper> local_queue_type;  // 1
  static thread_local std::unique_ptr<local_queue_type>
    local_work_queue;  // 2
  
  void worker_thread()
  {
    local_work_queue.reset(new local_queue_type);  // 3
    while(!done)
    {
      run_pending_task();
    }
  }

public:
  template<typename FunctionType>
  std::future<typename std::result_of<FunctionType()>::type>
    submit(FunctionType f)
  {
    typedef typename std::result_of<FunctionType()>::type result_type;

    std::packaged_task<result_type()> task(f);
    std::future<result_type> res(task.get_future());
    if(local_work_queue)  // 4
    {
      local_work_queue->push(std::move(task));
    }
    else
    {
      pool_work_queue.push(std::move(task));  // 5
    }
    return res;
  }

  void run_pending_task()
  {
    function_wrapper task;
    if(local_work_queue && !local_work_queue->empty())  // 6
    {
      task=std::move(local_work_queue->front());
      local_work_queue->pop();
      task();
    }
    else if(pool_work_queue.try_pop(task))  // 7
    {
      task();
    }
    else
    {
      std::this_thread::yield();
    }
  }
// rest as before
};
```

我们使用了`std::unique_ptr<>`指向线程本地的工作队列②，因为不希望非线程池中的线程也拥有一个任务队列；这个指针在worker_thread()中进行初始化③。`std:unique_ptr<>`的析构函数会保证在线程退出的时候，工作队列被销毁。

submit()会检查当前线程是否具有一个工作队列④。如果有，那就是线程池中的线程，并且可以将任务放入线程的本地队列中；否者，就像之前一样将这个任务放在线程池中的全局队列中⑤。

run_pending_task()⑥中的检查和之前类似，只是要对是否有本地任务队列进行检查。如果有，就会让其从队列中的第一个任务开始处理；注意这里的本地任务队列可以是一个普通的`std::queue<>`①，因为这个队列只能被一个线程所访问。如果本地线程上没有任务，那么就会从全局工作列表上获取任务⑦。

这样就能有效避免竞争，不过当任务分配不均时，造成的结果就是某个线程本地队列中有很多任务的同时，其他线程无所事事。例如，举一个快速排序的例子，只有一开始的数据块能在线程池上被处理，因为剩余部分会放在工作线程的本地队列上进行处理。这样使用违背使用线程池的初衷。

幸好，这个问题是有解的：在本地工作队列和全局工作队列上没有任务的时候，从别的线程队列中窃取任务。

###9.1.5 窃取任务

为了让没有任务的线程从能从其他线程的本地任务队列中获取任务，这里本地任务队里必须是可以访问的，这样才能让run_pending_tasks()窃取任务。这就需要每个线程在线程池队列上进行注册，或由线程池指定一个线程。同样，还需要保证在数据队列中的任务被适当的同步和保护，这样队列的不变量就不会被破坏。

实现一个无锁队列，让其拥有线程在其他线程窃取任务的时候，能够推送和弹出一个任务也是可能的；不过这个队列的实现就超出了本书的讨论范围。为了证明这种方法的可行性，还将使用一个互斥量来保护队列中的数据。我们希望任务窃取是一个不常见的现象，这样就会减少对互斥量的竞争，并且使得简单队列的开销最小。下面，实现了一个简单的基于锁的任务窃取队列。

清单9.7 基于锁的任务窃取队列
```c++
class work_stealing_queue
{
private:
  typedef function_wrapper data_type;
  std::deque<data_type> the_queue;  // 1
  mutable std::mutex the_mutex;

public:
  work_stealing_queue()
  {}

  work_stealing_queue(const work_stealing_queue& other)=delete;
  work_stealing_queue& operator=(
    const work_stealing_queue& other)=delete;

  void push(data_type data)  // 2
  {
    std::lock_guard<std::mutex> lock(the_mutex);
    the_queue.push_front(std::move(data));
  }

  bool empty() const
  {
    std::lock_guard<std::mutex> lock(the_mutex);
    return the_queue.empty();
  }

  bool try_pop(data_type& res)  // 3
  {
    std::lock_guard<std::mutex> lock(the_mutex);
    if(the_queue.empty())
    {
      return false;
    }

    res=std::move(the_queue.front());
    the_queue.pop_front();
    return true;
  }

  bool try_steal(data_type& res)  // 4
  {
    std::lock_guard<std::mutex> lock(the_mutex);
    if(the_queue.empty())
    {
      return false;
    }

    res=std::move(the_queue.back());
    the_queue.pop_back();
    return true;
  }
};
```

这个队列对`std::deque<fuction_wrapper>`进行了简单的包装①，这样就能通过一个互斥锁来对所有访问进行控制了。push()②和try_pop()③对队列的前端进行操作，而try_steal()④对队列的后端进行操作。

这就说明每个线程中的“队列”是一个后进先出的栈；最新推入的任务将会第一个执行。从魂村角度来看，这将对性能有所提升，因为任务相关的数据一直存于缓存中，要比提前将任务相关数据推送到栈上好。同样，这种方式能很好的映射到某个算法上，例如：快速排序。之前的实现中，每次调用do_sort()都会推送一个任务到栈上，并且等待这个任务执行完毕。通过对最新推入任务的处理，就可以保证在将当前所需数据块处理完成前，其他任务是否需要这些数据块，从而就可以减少活动任务的数量和栈的使用次数。try_steal()从队列末尾获取任务，是为了减少与try_pop()之间的竞争；可以使用在第6、7章中的所讨论的技术来让try_pop()和try_steal()并发执行。

OK，现在拥有了一个很不错的任务队列，并且支持窃取；那这个队列将如何在线程池中使用呢？这里展示一个简单的实现。

清单9.8 使用任务窃取的线程池
```c++
class thread_pool
{
  typedef function_wrapper task_type;

  std::atomic_bool done;
  thread_safe_queue<task_type> pool_work_queue;
  std::vector<std::unique_ptr<work_stealing_queue> > queues;  // 1
  std::vector<std::thread> threads;
  join_threads joiner;

  static thread_local work_stealing_queue* local_work_queue;  // 2
  static thread_local unsigned my_index;

  void worker_thread(unsigned my_index_)
  {
    my_index=my_index_;
    local_work_queue=queues[my_index].get();  // 3
    while(!done)
    {
      run_pending_task();
    }
  }

  bool pop_task_from_local_queue(task_type& task)
  {
    return local_work_queue && local_work_queue->try_pop(task);
  }

  bool pop_task_from_pool_queue(task_type& task)
  {
    return pool_work_queue.try_pop(task);
  }

  bool pop_task_from_other_thread_queue(task_type& task)  // 4
  {
    for(unsigned i=0;i<queues.size();++i)
    {
      unsigned const index=(my_index+i+1)%queues.size();  // 5
      if(queues[index]->try_steal(task))
      {
        return true;
      }
    }
    return false;
  }

public:
  thread_pool():
    done(false),joiner(threads)
  {
    unsigned const thread_count=std::thread::hardware_concurrency();

    try
    {
      for(unsigned i=0;i<thread_count;++i)
      {
        queues.push_back(std::unique_ptr<work_stealing_queue>(  // 6
                         new work_stealing_queue));
        threads.push_back(
          std::thread(&thread_pool::worker_thread,this,i));
      }
    }
    catch(...)
    {
      done=true;
      throw;
    }
  }

  ~thread_pool()
  {
    done=true;
  }

  template<typename FunctionType>
  std::future<typename std::result_of<FunctionType()>::type> submit(
    FunctionType f)
  { 
    typedef typename std::result_of<FunctionType()>::type result_type;
    std::packaged_task<result_type()> task(f);
    std::future<result_type> res(task.get_future());
    if(local_work_queue)
    {
      local_work_queue->push(std::move(task));
    }
    else
    {
      pool_work_queue.push(std::move(task));
    }
    return res;
  }

  void run_pending_task()
  {
    task_type task;
    if(pop_task_from_local_queue(task) ||  // 7
       pop_task_from_pool_queue(task) ||  // 8
       pop_task_from_other_thread_queue(task))  // 9
    {
      task();
    }
    else
    {
      std::this_thread::yield();
    }
  }
};
```

这段代码与清单9.6很相似。第一个不同在于，每个线程都有一个work_stealing_queue，而非只是普通的`std::queue<>`②。当每个线程被创建，就为每个线程创建了一个自己的工作队列，而非线程池只为自己构造一个任务队列⑥，每个线程自己的共工作队列将存在线程池的全局工作队列章①。列表中队列的序号，之后会传递给线程函数，然后使用序号来索引对应队列③。这就意味着线程池可以访问任意线程中的队列，为了给没有事情做的线程窃取任务。run_pending_task()将会从线程的任务队列中取出一个任务来执行⑦，或从线程池队列中获取一个任务⑧，亦或从其他线程的队列中获取一个任务⑨。

pop_task_from_other_thread_queue()④会遍历池中所有线程的任务队列，然后尝试从中窃取任务。为了避免美俄线程都尝试从列表中的第一个线程上窃取任务，每一个线程都会从下一个线程开始遍历，通过自身的线程序号来确定开始遍历的线程序号。

使用线程池有很多好处。当然，还有很多很多的方式能为某些特殊用法提升性能，不过这就留给读者作为课后作业吧。特别是还没有探究动态变换大小的线程池，即使线程被一些事情所阻塞的时候(例如：I/O或互斥锁)，程序都能保证CPU最优的使用率。

下面，我们将来看一看线程管理的“高级”用法——中断线程。

##9.2 中断线程

在很多情况下，使用信号来终止一个长时间运行的线程是合理的。这种线程的存在，可能是因为工作线程所在的线程池被销毁，或是用户显式的取消了这个任务，亦或其他各种原因。不管是什么原因，原理都一样：需要使用信号来让未结束线程停止运行。这里需要使用一种合适的方式让线程主动的停下来，而非让线程戛然而止。

你可能会给每种情况制定一个独立的机制，这样做的意义不大。不仅是因为用一个统一的机制会更容易在之后的场景中实现，而且这样写出来的中断代码就不用担心在哪里使用了。C++11标准没有提供这样一种机制，不过实现这样的一种机制也不是很困难。

让我们来了解一下应该如何实现这种机制，先来了解一下启动和中断一个线程的接口。

###9.2.1 启动和中断线程

先让我们看一下外部接口。需要从可中断线程上获取些什么？最基本的，需要和`std::thread`相同的接口，还要多加一个interrupt()函数：

```c++
class interruptible_thread
{
public:
  template<typename FunctionType>
  interruptible_thread(FunctionType f);
  void join();
  void detach();
  bool joinable() const;
  void interrupt();
};
```

在类内部，你可以使用`std::thread`来管理线程，并且使用一些自动以数据结构来处理中断。现在，从线程的角度能看到些什么呢？最基本的就是“我能用这个类来中断线程”——需要一个断点(*interruption point*)。为了使断点能够正常使用，且不添加多余的数据，就需要使用一个没有参数的函数：interruption_point()。这意味着中断数据结构可以访问一个thread_local变量，并在线程运行时，对变量进行设置，因此当有线程调用interruption_point()函数，就会去检查当前运行线程的数据结构。我们将在后面看到interruption_point()的具体实现。

thread_local标志是不能使用普通的`std::thread`管理线程的主要原因；其需要使用一种方法分配出一个可访问的interruptible_thread实例，就像新启动一个线程一样。在你使用已提供函数来做这件事情前，需要将interruptible_thread实例传递给`std::thread`的构造函数，创建一个能够执行的线程，就像下面的代码清单所实现。

清单9.9 interruptible_thread的基本实现
```c++
class interrupt_flag
{
public:
  void set();
  bool is_set() const;
};
thread_local interrupt_flag this_thread_interrupt_flag;  // 1

class interruptible_thread
{
  std::thread internal_thread;
  interrupt_flag* flag;
public:
  template<typename FunctionType>
  interruptible_thread(FunctionType f)
  {
    std::promise<interrupt_flag*> p;  // 2
    internal_thread=std::thread([f,&p]{  // 3
      p.set_value(&this_thread_interrupt_flag);
      f();  // 4
    });
    flag=p.get_future().get();  // 5
  }
  void interrupt()
  {
    if(flag)
    {
      flag->set();  // 6
    }
  }
};
```

提供函数f是包装了一个lambda函数③，线程将会持有f副本和本地promise变量p的引用②。在新线程中，lambda函数设置promise变量的值到this_thread_interrupt_flag(在thread_local①中声明)的地址中，为的是让线程能够调用提供函数的副本④。调用线程会等待与其future相关的promise就绪，并且将结果存入到flag成员变量中⑤。注意，即使lambda函数在新线程上执行，并且对本地变量p进行悬空引用，这都没有太大问题，因为在新线程返回之前，interruptible_thread构造函数会等待变量p，直到变量p不被引用。这里实现没有考虑去处理汇入线程，或分离线程。所以需要确保flag变量在线程退出或分离前，已经声明，这样就能避免悬空指针问题。

interrupt()函数相对简单：当需要一个线程去做中断时，需要一个合法指针作为一个中断标志，所以可以仅对标识进行设置⑥。这就是中断线程在做中断的时候，所要做的工作。

###9.2.2 检查线程是否中断

现在就可以设置中断标识了，不过不检查线程是否被中断，这么做的意义就不会那么大了。使用interruption_point()函数最简单的情况；你可以一个安全的地方调用这个函数，如果标识已经设置，那么就可以抛出一个thread_interrupted异常：

```c++
void interruption_point()
{
  if(this_thread_interrupt_flag.is_set())
  {
    throw thread_interrupted();
  }
}
```

在代码中可以在适当的地方使用这个函数：

```c++
void foo()
{
  while(!done)
  {
    interruption_point();
    process_next_item();
  }
}
```

虽然这样也能工作，但不理想。最好实在线程等待或阻塞的时候中断线程，因为这时的线程不能运行，也就不能调用interruption_point()函数！在线程等待的时候，什么方式才能去中断线程呢？

###9.2.3 中断等待——条件变量

OK，所以需要仔细选择中断的位置，并通过显示调用interruption_point()进行中断，不过在线程阻塞等待的时候，这种办法就显得苍白无力了，例如：等待条件变量的通知。这就需要一个新函数——interruptible_wait()——就可以运行各种需要等待的任务，并且你也可以知道如何中断等待。之前提过，可能会等待一个条件变量，所以就从它开始：你要如何做才能中断一个等待的条件变量呢？最简单的方式是，当设置中断标识时，需要要提醒条件变量，并在等待后立即设置中断点。为了让其工作，你需要提醒所有等待对应条件变量的线程，为了确保你所感谢兴趣的线程能够苏醒。伪苏醒是无论如何都要处理的，所以其他线程(非感兴趣线程)将会被当作伪苏醒处理——因为两者之间没什么区别。interrupt_flag结构需要存储一个指针指向一个条件变量，所以可以调用set()函数对其进行提醒。为条件变量实现的interruptible_wait()可能会看起来像下面清单中所示。

清单9.10 为`std::condition_variable`实现的interruptible_wait有问题版
```c++
void interruptible_wait(std::condition_variable& cv,
std::unique_lock<std::mutex>& lk)
{
  interruption_point();
  this_thread_interrupt_flag.set_condition_variable(cv);  // 1
  cv.wait(lk);  // 2
  this_thread_interrupt_flag.clear_condition_variable();  // 3
  interruption_point();
}
```

假设有函数能够设置和清除相关条件变量上的中断标志，这样的代码就是短小精悍的。其会检查中断，通过interrupt_flag为当前线程关联条件变量①，等待条件变量②，清理相关条件变量③，并且再次检查中断。如果线程在等待期间被条件变量所中断，中断线程将广播条件变量，并唤醒等待该条件变量的线程，所以这里就可以检查中断了。不幸的是，这里的代码有两个问题。第一个问题比较明显，如果想要线程安全：`std::condition_variable::wait()`可以抛出异常，所以这里会直接退出函数，而没有通过条件变量删除相关的中断标识。这个问题很容易修复，那就是在这个数据结构的析构函数中添加相关删除操作即可。

第二个问题就不大明显了，这段代码存在条件竞争。虽然，线程可以通过调用interruption_point()被中断，不过在调用wait()后，条件变量和相关中断标志就没有什么太大关系了，因为线程不是等待状态，所以不能通过条件变量的方式唤醒。需要确保线程不会在最后一次中断检查和调用wait()间被提醒。不对`std::condition_variable`的内部结构进行研究；不过，可以通过一种方法来解决这个问题：使用lk上的互斥量来对线程进行保护，这就需要将lk传递到set_condition_variable()函数中去。不幸的是，这将产生两个新问题：你需要传递一个互斥量的引用到一个不知道生命周期的线程中去(这个线程做中断操作)为该线程上锁(在调用interrupt()的时候)。这里可能会死锁，并且可能访问到一个已经销毁的互斥量，所以这种方法不可取。当不能完全确定能中断条件变量等待——在没有interruptible_wait()情况下也可以时，可能有些严格，那有没有其他选择呢？一个选择就是放置一个超时等待；使用wait_for()(而非wait())并带有一个简单的超时量(比如，1ms)。在线程被中断前，这算是给了线程一个等待的上限(以时钟刻度为基准)。如果你这样做了，等待线程将会看到更多因为超时而“伪”苏醒的线程，不过超时也不是很轻易的就帮助到我们。这样的一个实现放在下面的清单中，与interrupt_flag相关的实现。

清单9.11 为`std::condition_variable`在interruptible_wait中使用超时
```c++
class interrupt_flag
{
  std::atomic<bool> flag;
  std::condition_variable* thread_cond;
  std::mutex set_clear_mutex;

public:
  interrupt_flag():
    thread_cond(0)
  {}

  void set()
  {
    flag.store(true,std::memory_order_relaxed);
    std::lock_guard<std::mutex> lk(set_clear_mutex);
    if(thread_cond)
    {
      thread_cond->notify_all();
    }
  }

  bool is_set() const
  {
    return flag.load(std::memory_order_relaxed);
  }

  void set_condition_variable(std::condition_variable& cv)
  {
    std::lock_guard<std::mutex> lk(set_clear_mutex);
    thread_cond=&cv;
  }

  void clear_condition_variable()
  {
    std::lock_guard<std::mutex> lk(set_clear_mutex);
    thread_cond=0;
  }

  struct clear_cv_on_destruct
  {
    ~clear_cv_on_destruct()
    {
      this_thread_interrupt_flag.clear_condition_variable();
    }
  };
};

void interruptible_wait(std::condition_variable& cv,
  std::unique_lock<std::mutex>& lk)
{
  interruption_point();
  this_thread_interrupt_flag.set_condition_variable(cv);
  interrupt_flag::clear_cv_on_destruct guard;
  interruption_point();
  cv.wait_for(lk,std::chrono::milliseconds(1));
  interruption_point();
}
```

如果有谓词(相关函数)进行等待，然后1ms的超时将会完全在谓词循环中隐藏：

```c++
template<typename Predicate>
void interruptible_wait(std::condition_variable& cv,
                        std::unique_lock<std::mutex>& lk,
                        Predicate pred)
{
  interruption_point();
  this_thread_interrupt_flag.set_condition_variable(cv);
  interrupt_flag::clear_cv_on_destruct guard;
  while(!this_thread_interrupt_flag.is_set() && !pred())
  {
    cv.wait_for(lk,std::chrono::milliseconds(1));
  }
  interruption_point();
}
```

这会让谓词被检查的次数增加许多，不过对于简单调用wait()这套实现还是很好用的。超时变量很容易实现：通过制定时间，比如：1ms或更短。OK，对于`std::condition_variable`的等下，现在就需要小心对待了；那`std::condition_variable_any`呢？只能提供一样的功能，还是能做的更好？

###9.2.4 使用`std::condition_variable_any`中断等待

`std::condition_variable_any`与`std::condition_variable`的不同在于，`std::condition_variable_any`可以使用任意类型的锁，而不是仅有`std::unique_lock<std::mutex>`。其可以让事情做起来更加简单，并且`std::condition_variable_any`可以比`std::condition_variable`做的更好。因为其能与任意类型的锁一起工作，这样就可以设计自己的锁，去锁/解锁interrupt_flag的内部互斥量set_clear_mutex，并且锁也支持等待调用，就像下面的代码。

清单9.12 为`std::condition_variable_any`设计的interruptible_wait
```c++
class interrupt_flag
{
  std::atomic<bool> flag;
  std::condition_variable* thread_cond;
  std::condition_variable_any* thread_cond_any;
  std::mutex set_clear_mutex;

public:
  interrupt_flag(): 
    thread_cond(0),thread_cond_any(0)
  {}

  void set()
  {
    flag.store(true,std::memory_order_relaxed);
    std::lock_guard<std::mutex> lk(set_clear_mutex);
    if(thread_cond)
    {
      thread_cond->notify_all();
    }
    else if(thread_cond_any)
    {
      thread_cond_any->notify_all();
    }
  }

  template<typename Lockable>
  void wait(std::condition_variable_any& cv,Lockable& lk)
  {
    struct custom_lock
    {
      interrupt_flag* self;
      Lockable& lk;

      custom_lock(interrupt_flag* self_,
                  std::condition_variable_any& cond,
                  Lockable& lk_):
        self(self_),lk(lk_)
      {
        self->set_clear_mutex.lock();  // 1
        self->thread_cond_any=&cond;  // 2
      }

      void unlock()  // 3
      {
        lk.unlock();
        self->set_clear_mutex.unlock();
      }

      void lock()
      {
        std::lock(self->set_clear_mutex,lk);  // 4
      }

      ~custom_lock()
      {
        self->thread_cond_any=0;  // 5
        self->set_clear_mutex.unlock();
      }
    };
    custom_lock cl(this,cv,lk);
    interruption_point();
    cv.wait(cl);
    interruption_point();
  }
  // rest as before
};

template<typename Lockable>
void interruptible_wait(std::condition_variable_any& cv,
                        Lockable& lk)
{
  this_thread_interrupt_flag.wait(cv,lk);
}
```

你自定义的锁类型，在构造的时候，需要所锁住内部set_clear_mutex①，并且对thread_cond_any指针进行设置，并引用`std::condition_variable_any`传入锁的构造函数中②。Lockable引用将会在之后进行存储；其变量必须被锁住。仙子可以安心的检查中断，而不用担心竞争了。如果这时中断标志已经设置，那么标志一定是在锁住set_clear_mutex设置的。当条件变量调用你的锁类型的unlock()函数中的wait()时，就会对Lockable对象和set_clear_mutex进行解锁③。这就允许线程可以尝试中断其他线程获取set_clear_mutex的锁，以及在内部wait()调用之后，检查thread_cond_any指针。这些就是你在替换`std::condition_variable`所拥有的功能(不包括管理)。当wait()结束等待(因为等待，或因为伪苏醒)，线程将会调用lock()函数，这里依旧要求锁住内部set_clear_mutex，并且锁住Lockable对象④。现在，在wait()调用时，custom_lock的析构函数中⑤清理thread_cond_any指针(同样会解锁set_clear_mutex)之前，可以再次对中断进行检查。

###9.2.5 中断其他阻塞调用

这次轮到中断条件变量的等待了，不过其他阻塞情况，比如：互斥锁，等待future等等，该怎么办呢？通常情况下，可以使用`std::condition_variable`的超时选项，因为在实际运行中不可能很快的将条件变量的等待终止(不访问内部互斥量或future的话)。不过，在某些情况下，你知道知道你在等待什么，这样你就可以让循环在interruptible_wait()函数中运行。作为一个例子，这里有一个为`std::future<>`重载的interruptible_wait()实现：

```c++
template<typename T>
void interruptible_wait(std::future<T>& uf)
{
  while(!this_thread_interrupt_flag.is_set())
  {
    if(uf.wait_for(lk,std::chrono::milliseconds(1)==
       std::future_status::ready)
      break;
  }
  interruption_point();
}
```

这个等待会在中断标志被设置好的时候，或future准备就绪的时候停止，不过这个实现中每次等待future的时间只有1ms。这就意味着，在中断请求被确定前，平均等待的时间为0.5ms(这里假设存在一个高精度的时钟)。通常wait_for至少会等待一个时钟周期，所以如果时钟周期为15ms，那么结束等待的时间将会是15ms，而不是1ms。接受与不接受这种情况，都得视情况而定。如果这必要且时钟支持的话，可以持续削减超时时间。这种方式的确定就是，线程将会苏醒很多次来检查标志，并且这将增加线程切换的开销。

OK，我们已经了解如何使用interruption_point()和interruptible_wait()函数检查中断，当中断被检查出来了，要如何处理它呢？

###9.2.6 处理中断

###9.2.7 应用退出时中断后台任务

##9.3 小结