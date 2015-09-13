#附录D C++线程库参考

##D.1 <chrono>头文件

<chrono>头文件作为`time_point`的提供者，具有代表时间点的类，duration类和时钟类。每个时钟都有一个`is_steady`静态数据成员，这个成员用来表示该时钟是否是一个*稳定的*时钟(以匀速计时的时钟，且不可调节)。`std::chrono::steady_clock`是唯一个能保证稳定的时钟类。

头文件正文
```c++
namespace std
{
  namespace chrono
  {
    template<typename Rep,typename Period = ratio<1>>
    class duration;
    template<
        typename Clock,
        typename Duration = typename Clock::duration>
    class time_point;
    class system_clock;
    class steady_clock;
    typedef unspecified-clock-type high_resolution_clock;
  }
}
```

###D.1.1 std::chrono::duration类型模板

`std::chrono::duration`类模板可以用来表示时间。模板参数`Rep`和`Period`是用来存储持续时间的数据类型，`std::ratio`实例代表了时间的长度(几分之一秒)，其表示了在两次“时钟滴答”后的时间(时钟周期)。因此，`std::chrono::duration<int, std::milli>`即为，时间以毫秒数的形式存储到int类型中，而`std::chrono::duration<short, std::ratio<1,50>>`则是记录1/50秒的个数，并将个数存入short类型的变量中，还有`std::chrono::duration <long long, std::ratio<60,1>>`则是将分钟数存储到long long类型的变量中。

**类的定义**
```c++
template <class Rep, class Period=ratio<1> >
class duration
{
public:
  typedef Rep rep;
  typedef Period period;

  constexpr duration() = default;
  ~duration() = default;

  duration(const duration&) = default;
  duration& operator=(const duration&) = default;

  template <class Rep2>
  constexpr explicit duration(const Rep2& r);

  template <class Rep2, class Period2>
  constexpr duration(const duration<Rep2, Period2>& d);

  constexpr rep count() const;
  constexpr duration operator+() const;
  constexpr duration operator-() const;

  duration& operator++();
  duration operator++(int);
  duration& operator--();
  duration operator--(int);

  duration& operator+=(const duration& d);
  duration& operator-=(const duration& d);
  duration& operator*=(const rep& rhs);
  duration& operator/=(const rep& rhs);

  duration& operator%=(const rep& rhs);
  duration& operator%=(const duration& rhs);

  static constexpr duration zero();
  static constexpr duration min();
  static constexpr duration max();
};
template <class Rep1, class Period1, class Rep2, class Period2>
constexpr bool operator==(
    const duration<Rep1, Period1>& lhs,
    const duration<Rep2, Period2>& rhs);

template <class Rep1, class Period1, class Rep2, class Period2>
    constexpr bool operator!=(
    const duration<Rep1, Period1>& lhs,
    const duration<Rep2, Period2>& rhs);

template <class Rep1, class Period1, class Rep2, class Period2>
    constexpr bool operator<(
    const duration<Rep1, Period1>& lhs,
    const duration<Rep2, Period2>& rhs);

template <class Rep1, class Period1, class Rep2, class Period2>
    constexpr bool operator<=(
    const duration<Rep1, Period1>& lhs,
    const duration<Rep2, Period2>& rhs);

template <class Rep1, class Period1, class Rep2, class Period2>
    constexpr bool operator>(
    const duration<Rep1, Period1>& lhs,
    const duration<Rep2, Period2>& rhs);

template <class Rep1, class Period1, class Rep2, class Period2>
    constexpr bool operator>=(
    const duration<Rep1, Period1>& lhs,
    const duration<Rep2, Period2>& rhs);

template <class ToDuration, class Rep, class Period>
    constexpr ToDuration duration_cast(const duration<Rep, Period>& d);
```

**要求**<br>
`Rep`必须是内置数值类型，或是自定义的类数值类型。

`Period`必须是`std::ratio<>`实例。

**std::chrono::duration::Rep 类型**

用来记录`dration`中时钟周期的数量。

**声明**<br>
```c++
typedef Rep rep;
```

`rep`类型用来记录`duration`对象内部的表示。

**std::chrono::duration::Period 类型**

这个类型必须是一个`std::ratio`的特化实例，用来表示在继续时间中，1s所要记录的次数。例如，当`period`是`std::ratio<1, 50>`，`duration`变量的count()就会在N秒钟返回50N。

**声明**
```c++
typedef Period period;
```

**std::chrono::duration 默认构造函数**

使用默认值构造`std::chrono::duration`实例

**声明**
```c++
constexpr duration() = default;
```

**效果**<br>
`duration`内部值(例如`rep`类型的值)都已初始化。

**std::chrono::duration 需要计数值的转换构造函数**

通过给定的数值来构造`std::chrono::duration`实例。

**声明**
```c++
template <class Rep2>
constexpr explicit duration(const Rep2& r);
```

**效果**<br>
`duration`对象的内部值会使用`static_cast<rep>(r)`进行初始化。

**结果**<br>
当Rep2隐式转换为Rep，Rep是浮点类型或Rep2不是浮点类型，这个构造函数才能使用。

后验条件
```c++
this->count()==static_cast<rep>(r)
```

**std::chrono::duration 需要另一个std::chrono::duration值的转化构造函数**

通过另一个`std::chrono::duration`类实例中的计数值来构造一个`std::chrono::duration`类实例。

**声明**
```c++
template <class Rep2, class Period2>
constexpr duration(const duration<Rep2,Period2>& d);
```

**结果**<br>

duration对象的内部值通过`duration_cast<duration<Rep,Period>>(d).count()`初始化。

**要求**<br>
当Rep是一个浮点类或Rep2不是浮点类，且Period2是Period数的倍数(比如，ratio_divide<Period2,Period>::den==1)时，才能调用该重载。当一个较小的数据转换为一个较大的数据时，使用该构造函数就能避免数位截断和精度损失。

**后验条件**<br>
`this->count() == dutation_cast<duration<Rep, Period>>(d).count()`

**例子**
```c++
duration<int, ratio<1, 1000>> ms(5);  // 5毫秒
duration<int, ratio<1, 1>> s(ms);  // 错误：不能将ms当做s进行存储
duration<double, ratio<1,1>> s2(ms);  // 合法：s2.count() == 0.005
duration<int, ration<1, 1000000>> us<ms>;  // 合法:us.count() == 5000
```

**std::chrono::duration::count 成员函数**

查询持续时长。

**声明**
```c++
constexpr rep count() const;
```

**返回**<br>
返回duration的内部值，其值类型和rep一样。

**std::chrono::duration::operator+ 加法操作符**

这是一个空操作：只会返回*this的副本。

**声明**
```c++
constexpr duration operator+() const;
```

**返回**
`*this`

**std::chrono::duration::operator- 减法操作符**

返回将内部值只为负数的*this副本。

**声明**
```c++
constexpr duration operator-() const;
```

**返回**
`duration(-this->count());`

**std::chrono::duration::operator++ 前置自加操作符**

增加内部计数值。

**声明**
```c++
duration& operator++();
```

**结果**
```c++
++this->internal_count;
```

**返回**
`*this`

**std::chrono::duration::operator++ 后置自加操作符**

自加内部计数值，并且返回还没有增加前的*this。

**声明**
```c++
duration operator++(int);
```

**结果**
```c++
duration temp(*this);
++(*this);
return temp;
```

**std::chrono::duration::operator-- 前置自减操作符**

自减内部计数值

**声明**
```c++
duration& operator--();
```

**结果**
```c++
--this->internal_count;
```

**返回**
`*this`

**std::chrono::duration::operator-- 前置自减操作符**

自减内部计数值，并且返回还没有减少前的*this。

**声明**
```c++
duration operator--(int);
```

**结果**
```c++
duration temp(*this);
--(*this);
return temp;
```

**std::chrono::duration::operator+= 复合赋值操作符**

将其他duration对象中的内部值增加到现有duration对象当中。

**声明**
```c++
duration& operator+=(duration const& other);
```

**结果**
```c++
internal_count+=other.count();
```

**返回**
`*this`

**std::chrono::duration::operator-= 复合赋值操作符**

现有duration对象减去其他duration对象中的内部值。

**声明**
```c++
duration& operator-=(duration const& other);
```

**结果**
```c++
internal_count-=other.count();
```

**返回**
`*this`

**std::chrono::duration::operator*= 复合赋值操作符**

内部值乘以一个给定的值。

**声明**
```c++
duration& operator*=(rep const& rhs);
```

**结果**
```c++
internal_count*=rhs;
```

**返回**
`*this`

**std::chrono::duration::operator/= 复合赋值操作符**

内部值除以一个给定的值。

**声明**
```c++
duration& operator/=(rep const& rhs);
```

**结果**
```c++
internal_count/=rhs;
```

**返回**
`*this`

**std::chrono::duration::operator%= 复合赋值操作符**

内部值对一个给定的值求余。

**声明**
```c++
duration& operator%=(rep const& rhs);
```

**结果**
```c++
internal_count%=rhs;
```

**返回**
`*this`

**std::chrono::duration::operator%= 复合赋值操作符(重载)**

内部值对另一个duration类的内部值求余。

**声明**
```c++
duration& operator%=(duration const& rhs);
```

**结果**
```c++
internal_count%=rhs.count();
```

**返回**
`*this`

**std::chrono::duration::zero 静态成员函数**

返回一个内部值为0的duration对象。

**声明**
```c++
constexpr duration zero();
```

**返回**
```c++
duration(duration_values<rep>::zero());
```

**std::chrono::duration::min 静态成员函数**

返回duration类实例化后能表示的最小值。

**声明**
```c++
constexpr duration min();
```

**返回**
```c++
duration(duration_values<rep>::min());
```

**std::chrono::duration::max 静态成员函数**

返回duration类实例化后能表示的最大值。

**声明**
```c++
constexpr duration max();
```

**返回**
```c++
duration(duration_values<rep>::max());
```

**std::chrono::duration 等于比较操作符**

比较两个duration对象是否相等。

**声明**
```c++
template <class Rep1, class Period1, class Rep2, class Period2>
constexpr bool operator==(
const duration<Rep1, Period1>& lhs,
const duration<Rep2, Period2>& rhs);
```

**要求**<br>
`lhs`和`rhs`两种类型可以互相进行隐式转换。当两种类型无法进行隐式转换，或是可以互相转换的两个不同类型的duration类，则表达式不合理。

**结果**<br>
当`CommonDuration`和`std::common_type< duration< Rep1, Period1>, duration< Rep2, Period2>>::type`同类，那么`lhs==rhs`就会返回`CommonDuration(lhs).count()==CommonDuration(rhs).count()`。

**std::chrono::duration 不等于比较操作符**

比较两个duration对象是否不相等。

**声明**
```c++
template <class Rep1, class Period1, class Rep2, class Period2>
constexpr bool operator!=(
   const duration<Rep1, Period1>& lhs,
   const duration<Rep2, Period2>& rhs);
```

**要求**<br>
`lhs`和`rhs`两种类型可以互相进行隐式转换。当两种类型无法进行隐式转换，或是可以互相转换的两个不同类型的duration类，则表达式不合理。

**返回**
`!(lhs==rhs)`

**std::chrono::duration 小于比较操作符**

比较两个duration对象是否小于。

**声明**
```c++
template <class Rep1, class Period1, class Rep2, class Period2>
constexpr bool operator<(
   const duration<Rep1, Period1>& lhs,
   const duration<Rep2, Period2>& rhs);
```

**要求**<br>
`lhs`和`rhs`两种类型可以互相进行隐式转换。当两种类型无法进行隐式转换，或是可以互相转换的两个不同类型的duration类，则表达式不合理。

**结果**<br>
当`CommonDuration`和`std::common_type< duration< Rep1, Period1>, duration< Rep2, Period2>>::type`同类，那么`lhs<rhs`就会返回`CommonDuration(lhs).count()<CommonDuration(rhs).count()`。

**std::chrono::duration 大于比较操作符**

比较两个duration对象是否大于。

**声明**
```c++
template <class Rep1, class Period1, class Rep2, class Period2>
constexpr bool operator>(
   const duration<Rep1, Period1>& lhs,
   const duration<Rep2, Period2>& rhs);
```

**要求**<br>
`lhs`和`rhs`两种类型可以互相进行隐式转换。当两种类型无法进行隐式转换，或是可以互相转换的两个不同类型的duration类，则表达式不合理。

**返回**
`rhs<lhs`

**std::chrono::duration 小于等于比较操作符**

比较两个duration对象是否小于等于。

**声明**
```c++
template <class Rep1, class Period1, class Rep2, class Period2>
constexpr bool operator<=(
   const duration<Rep1, Period1>& lhs,
   const duration<Rep2, Period2>& rhs);
```

**要求**<br>
`lhs`和`rhs`两种类型可以互相进行隐式转换。当两种类型无法进行隐式转换，或是可以互相转换的两个不同类型的duration类，则表达式不合理。

**返回**
`!(rhs<lhs)`

**std::chrono::duration 大于等于比较操作符**

比较两个duration对象是否大于等于。

**声明**
```c++
template <class Rep1, class Period1, class Rep2, class Period2>
constexpr bool operator>=(
   const duration<Rep1, Period1>& lhs,
   const duration<Rep2, Period2>& rhs);
```

**要求**<br>
`lhs`和`rhs`两种类型可以互相进行隐式转换。当两种类型无法进行隐式转换，或是可以互相转换的两个不同类型的duration类，则表达式不合理。

**返回**
`!(lhs<rhs)`

**std::chrono::duration_cast 非成员函数**

显示将一个`std::chrono::duration`对象转化为另一个`std::chrono::duration`实例。

**声明**
```c++
template <class ToDuration, class Rep, class Period>
constexpr ToDuration duration_cast(const duration<Rep, Period>& d);
```

**要求**<br>
ToDuration必须是`std::chrono::duration`的实例。

**返回**<br>
duration类d转换为指定类型ToDuration。这种方式可以在不同尺寸和表示类型的转换中尽可能减少精度损失。

###D.1.2 std::chrono::time_point类型模板

`std::chrono::time_point`类型模板通过(特别的)时钟来表示某个时间点。这个时钟代表的是从epoch(1970-01-01 00:00:00 UTC，作为UNIX系列系统的特定时间戳)到现在的时间。模板参数Clock代表使用的使用(不同的使用必定有自己独特的类型)，而Duration模板参数使用来测量从epoch到现在的时间，并且这个参数的类型必须是`std::chrono::duration`类型。Duration默认存储Clock上的测量值。

**类型定义**
```c++
template <class Clock,class Duration = typename Clock::duration>
class time_point
{
public:
  typedef Clock clock;
  typedef Duration duration;
  typedef typename duration::rep rep;
  typedef typename duration::period period;
  
  time_point();
  explicit time_point(const duration& d);

  template <class Duration2>
  time_point(const time_point<clock, Duration2>& t);

  duration time_since_epoch() const;
  
  time_point& operator+=(const duration& d);
  time_point& operator-=(const duration& d);
  
  static constexpr time_point min();
  static constexpr time_point max();
};
```

**std::chrono::time_point 默认构造函数**

构造

**std::chrono::time_point 需要时间长度的构造函数**

**std::chrono::time_point 转换构造函数**

**std::chrono::time_point::time_since_epoch 成员函数**

**std::chrono::time_point::operator+= 复合赋值函数**

**std::chrono::time_point::operator-= 复合赋值函数**

**std::chrono::time_point::min 静态成员函数**

**std::chrono::time_point::max 静态成员函数**

###D.1.3 std::chrono::system_clock类

###D.1.4 std::chrono::steady_clock类

###D.1.5 std::chrono::high_resolution_clock类定义

##D.2 <condition_variable>头文件

###D.2.1 std::condition_variable类

###D.2.2 std::condition_variable_any类

##D.3 <atomic>头文件

###std::atomic_xxx类型定义

###D.3.2 ATOMIC_xxx_LOCK_FREE宏

###D.3.3 ATOMIC_VAR_INIT宏

###D.3.4 std::memory_order枚举类型

###D.3.5 std::atomic_thread_fence函数

###D.3.6 std::atomic_signal_fence函数

###D.3.7 std::atomic_flag类

###D.3.8 std::atomic类型模板

###D.3.9 std::atomic模板类型的特化

###D.3.10 特化std::atomic<integral-type>

##D.4 <future>头文件

###D.4.1 std::future类型模板

###D.4.2 std::shared_future类型模板

###D.4.3 std::packaged_task类型模板

###D.4.4 std::promise类型模板

###D.4.5 std::async函数模板

##D.5 <mutex>头文件

###D.5.1 std::mutex类

###D.5.2 std::recursive_mutex类

###D.5.3 std::timed_mutex类

###D.5.4 std::recursive_timed_mutex类

###D.5.5 std::lock_guard类型模板

###D.5.6 std::unique_lock类型模板

###D.5.7 std::lock函数模板

###D.5.8 std::try_lock函数模板

###D.5.9 std::once_flag类

###D.5.10 std::call_once函数模板

##D.6 <ratio>头文件

###D.6.1 std::ratio类型模板

###D.6.2 std::ratio_add模板别名

###D.6.3 std::ratio_subtract模板别名

###D.6.4 std::ratio_multiply模板别名

###D.6.5 std::ratio_divide模板别名

###D.6.6 std::ratio_equal类型模板

###D.6.7 std::ratio_not_equal类型模板

###D.6.8 std::ratio_less类型模板

###D.6.9 std::ratio_greater类型模板

###D.6.10 std::ratio_less_equal类型模板

###D.6.11 std::ratio_greater_equal类型模板

##D.7 <thread>头文件

###D.7.1 std::thread类

###D.7.2 this_thread命名空间