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

####类的定义

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

####std::chrono::duration::Rep 类型

用来记录`dration`中时钟周期的数量。

**声明**<br>
```c++
typedef Rep rep;
```

`rep`类型用来记录`duration`对象内部的表示。

####std::chrono::duration::Period 类型

这个类型必须是一个`std::ratio`的特化实例，用来表示在继续时间中，1s所要记录的次数。例如，当`period`是`std::ratio<1, 50>`，`duration`变量的count()就会在N秒钟返回50N。

**声明**
```c++
typedef Period period;
```

####std::chrono::duration 默认构造函数

使用默认值构造`std::chrono::duration`实例

**声明**
```c++
constexpr duration() = default;
```

**效果**<br>
`duration`内部值(例如`rep`类型的值)都已初始化。

####std::chrono::duration 需要计数值的转换构造函数

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

**后验条件**
```c++
this->count()==static_cast<rep>(r)
```

####std::chrono::duration 需要另一个std::chrono::duration值的转化构造函数

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

####std::chrono::duration::count 成员函数

查询持续时长。

**声明**
```c++
constexpr rep count() const;
```

**返回**<br>
返回duration的内部值，其值类型和rep一样。

####std::chrono::duration::operator+ 加法操作符

这是一个空操作：只会返回*this的副本。

**声明**
```c++
constexpr duration operator+() const;
```

**返回**
`*this`

####std::chrono::duration::operator- 减法操作符

返回将内部值只为负数的*this副本。

**声明**
```c++
constexpr duration operator-() const;
```

**返回**
`duration(-this->count());`

####std::chrono::duration::operator++ 前置自加操作符

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

####std::chrono::duration::operator++ 后置自加操作符

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

####std::chrono::duration::operator-- 前置自减操作符

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

####std::chrono::duration::operator-- 前置自减操作符

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

####std::chrono::duration::operator+= 复合赋值操作符

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

####std::chrono::duration::operator-= 复合赋值操作符

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

####std::chrono::duration::operator*= 复合赋值操作符

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

####std::chrono::duration::operator/= 复合赋值操作符

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

####std::chrono::duration::operator%= 复合赋值操作符

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

####std::chrono::duration::operator%= 复合赋值操作符(重载)

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

####std::chrono::duration::zero 静态成员函数

返回一个内部值为0的duration对象。

**声明**
```c++
constexpr duration zero();
```

**返回**
```c++
duration(duration_values<rep>::zero());
```

####std::chrono::duration::min 静态成员函数

返回duration类实例化后能表示的最小值。

**声明**
```c++
constexpr duration min();
```

**返回**
```c++
duration(duration_values<rep>::min());
```

####std::chrono::duration::max 静态成员函数

返回duration类实例化后能表示的最大值。

**声明**
```c++
constexpr duration max();
```

**返回**
```c++
duration(duration_values<rep>::max());
```

####std::chrono::duration 等于比较操作符

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

####std::chrono::duration 不等于比较操作符

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

####std::chrono::duration 小于比较操作符

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

####std::chrono::duration 大于比较操作符

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

####std::chrono::duration 小于等于比较操作符

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

####std::chrono::duration 大于等于比较操作符

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

####std::chrono::duration_cast 非成员函数

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

####类型定义

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

####std::chrono::time_point 默认构造函数

构造time_point代表着，使用相关的Clock，记录从epoch到现在的时间；其内部计时使用Duration::zero()进行初始化。

**声明**
```c++
time_point();
```

**后验条件**<br>
对于使用默认构造函数构造出的time_point对象tp，`tp.time_since_epoch() == tp::duration::zero()`。

####std::chrono::time_point 需要时间长度的构造函数

构造time_point代表着，使用相关的Clock，记录从epoch到现在的时间。

**声明**
```c++
explicit time_point(const duration& d);
```

**后验条件**<br>
当有一个time_point对象tp，是通过duration d构造出来的(tp(d))，那么`tp.time_since_epoch() == d`。

####std::chrono::time_point 转换构造函数

构造time_point代表着，使用相关的Clock，记录从epoch到现在的时间。

**声明**
```c++
template <class Duration2>
time_point(const time_point<clock, Duration2>& t);
```

**要求**<br>
Duration2必须呢个隐式转换为Duration。


**效果**<br>
当`time_point(t.time_since_epoch())`存在，从t.time_since_epoch()中获取的返回值，可以隐式转换成Duration类型的对象，并且这个值可以存储在一个新的time_point对象中。

(扩展阅读：[as-if准则](http://stackoverflow.com/questions/15718262/what-exactly-is-the-as-if-rule))

####std::chrono::time_point::time_since_epoch 成员函数

返回当前time_point从epoch到现在的具体时长。

**声明**
```c++
duration time_since_epoch() const;
```

**返回**<br>
duration的值存储在*this中。

####std::chrono::time_point::operator+= 复合赋值函数

将指定的duration的值与原存储在指定的time_point对象中的duration相加，并将加后值存储在*this对象中。

**声明**
```c++
time_point& operator+=(const duration& d);
```

**效果**<br>
将d的值和duration对象的值相加，存储在*this中，就如同this->internal_duration += d;

**返回**
`*this`

####std::chrono::time_point::operator-= 复合赋值函数

将指定的duration的值与原存储在指定的time_point对象中的duration相减，并将加后值存储在*this对象中。

**声明**
```c++
time_point& operator-=(const duration& d);
```

**效果**<br>
将d的值和duration对象的值相减，存储在*this中，就如同this->internal_duration -= d;

**返回**
`*this`

####std::chrono::time_point::min 静态成员函数

获取time_point对象可能表示的最小值。

**声明**
```c++
static constexpr time_point min();
```

**返回**
```c++
time_point(time_point::duration::min()) (see 11.1.1.15)
```

####std::chrono::time_point::max 静态成员函数

获取time_point对象可能表示的最大值。

**声明**
```c++
static constexpr time_point max();
```

**返回**
```c++
time_point(time_point::duration::max()) (see 11.1.1.16)
```

###D.1.3 std::chrono::system_clock类

`std::chrono::system_clock`类提供给了从系统实时时钟上获取当前时间功能。可以调用`std::chrono::system_clock::now()`来获取当前的时间。`std::chrono::system_clock::time_point`也可以通过`std::chrono::system_clock::to_time_t()`和`std::chrono::system_clock::to_time_point()`函数返回值转换成time_t类型。系统时钟不稳定，所以`std::chrono::system_clock::now()`获取到的时间可能会早于之前的一次调用(比如，时钟被手动调整过或与外部时钟进行了同步)。

####类型定义

```c++
class system_clock
{
public:
  typedef unspecified-integral-type rep;
  typedef std::ratio<unspecified,unspecified> period;
  typedef std::chrono::duration<rep,period> duration;
  typedef std::chrono::time_point<system_clock> time_point;
  static const bool is_steady=unspecified;

  static time_point now() noexcept;

  static time_t to_time_t(const time_point& t) noexcept;
  static time_point from_time_t(time_t t) noexcept;
};
```

####std::chrono::system_clock::rep 类型定义

将时间周期数记录在一个duration值中

**声明**
```c++
typedef unspecified-integral-type rep;
```

####std::chrono::system_clock::period 类型定义

类型为`std::ratio`类型模板，通过在两个不同的duration或time_point间特化最小秒数(或将1秒分为好几份)。period指定了时钟的精度，而非时钟频率。

**声明**
```c++
typedef std::ratio<unspecified,unspecified> period;
```

####std::chrono::system_clock::duration 类型定义

类型为`std::ratio`类型模板，通过系统实时时钟获取两个时间点之间的时长。

**声明**
```c++
typedef std::chrono::duration<
   std::chrono::system_clock::rep,
   std::chrono::system_clock::period> duration;
```

####std::chrono::system_clock::time_point 类型定义

类型为`std::ratio`类型模板，通过系统实时时钟获取当前时间点的时间。

**声明**<br>
```c++
typedef std::chrono::time_point<std::chrono::system_clock> time_point;
```

####std::chrono::system_clock::now 静态成员函数

从系统实时时钟上获取当前的外部设备显示的时间。

**声明**
```c++
time_point now() noexcept;
```

**返回**<br>
time_point类型变量来代表当前系统实时时钟的时间。

**抛出**<br>
当错误发生，`std::system_error`异常将会抛出。

####std::chrono::system_clock:to_time_t 静态成员函数

将time_point类型值转化为time_t。

**声明**
```c++
time_t to_time_t(time_point const& t) noexcept;
```

**返回**<br>
通过对t进行舍入或截断精度，将其转化为一个time_t类型的值。

**抛出**<br>
当错误发生，`std::system_error`异常将会抛出。

####std::chrono::system_clock::from_time_t 静态成员函数

**声明**
```c++
time_point from_time_t(time_t const& t) noexcept;
```

**返回**<br>
time_point中的值与t中的值一样。

**抛出**<br>
当错误发生，`std::system_error`异常将会抛出。

###D.1.4 std::chrono::steady_clock类

`std::chrono::steady_clock`能访问系统稳定时钟。可以通过调用`std::chrono::steady_clock::now()`获取当前的时间。设备上显示的时间，与使用`std::chrono::steady_clock::now()`获取的时间没有固定的关系。稳定时钟是无法回调的，所以在`std::chrono::steady_clock::now()`两次调用后，第二次调用获取的时间必定等于或大于第一次获得的时间。时钟以固定的速率进行计时。

####类型定义

```c++
class steady_clock
{
public:
  typedef unspecified-integral-type rep;
  typedef std::ratio<
      unspecified,unspecified> period;
  typedef std::chrono::duration<rep,period> duration;
  typedef std::chrono::time_point<steady_clock>
      time_point;
  static const bool is_steady=true;

  static time_point now() noexcept;
};
```

####std::chrono::steady_clock::rep 类型定义

定义一个整型，用来保存duration的值。

**声明**
```c++
typedef unspecified-integral-type rep;
```

####std::chrono::steady_clock::period 类型定义

类型为`std::ratio`类型模板，通过在两个不同的duration或time_point间特化最小秒数(或将1秒分为好几份)。period指定了时钟的精度，而非时钟频率。

**声明**
```c++
typedef std::ratio<unspecified,unspecified> period;
```

####std::chrono::steady_clock::duration 类型定义

类型为`std::ratio`类型模板，通过系统实时时钟获取两个时间点之间的时长。

**声明**
```c++
typedef std::chrono::duration<
   std::chrono::system_clock::rep,
   std::chrono::system_clock::period> duration;
```

####std::chrono::steady_clock::time_point 类型定义

`std::chrono::time_point`类型实例，可以存储从系统稳定时钟返回的时间点。

**声明**
```c++
typedef std::chrono::time_point<std::chrono::steady_clock> time_point;
```

####std::chrono::steady_clock::now 静态成员函数

从系统稳定时钟获取当前时间。

**声明**
```c++
time_point now() noexcept;
```

**返回**<br>
time_point表示当前系统稳定时钟的时间。

**抛出**<br>
当遇到错误，会抛出`std::system_error`异常。

**同步**<br>
当先行调用过一次`std::chrono::steady_clock::now()`，那么下一次time_point获取的值，一定大于等于第一次获取的值。

###D.1.5 std::chrono::high_resolution_clock类定义

`td::chrono::high_resolution_clock`类能访问系统高精度时钟。和所有其他时钟一样，通过调用`std::chrono::high_resolution_clock::now()`来获取当前时间。`std::chrono::high_resolution_clock`可能是`std::chrono::system_clock`类或`std::chrono::steady_clock`类的别名，也可能就是独立的一个类。

通过`std::chrono::high_resolution_clock`具有所有标准库支持时钟中最高的精度，这就意味着使用
`std::chrono::high_resolution_clock::now()`要花掉一些时间。所以，当你再调用`std::chrono::high_resolution_clock::now()`的时候，需要注意函数本身的时间开销。

####类型定义

```c++
class high_resolution_clock
{
public:
  typedef unspecified-integral-type rep;
  typedef std::ratio<
      unspecified,unspecified> period;
  typedef std::chrono::duration<rep,period> duration;
  typedef std::chrono::time_point<
      unspecified> time_point;
  static const bool is_steady=unspecified;

  static time_point now() noexcept;
};
```

##D.2 <condition_variable>头文件

<condition_variable>头文件提供了条件变量的定义。其作为基本同步机制，允许被阻塞的线程在某些条件达成或超时时，解除阻塞继续执行。

####头文件内容
```c++
namespace std
{
  enum class cv_status { timeout, no_timeout };
  
  class condition_variable;
  class condition_variable_any;
}
```

###D.2.1 std::condition_variable类

`std::condition_variable`允许阻塞一个线程，直到条件达成。

`std::condition_variable`实例不支持CopyAssignable(拷贝赋值), CopyConstructible(拷贝构造), MoveAssignable(移动赋值)和 MoveConstructible(移动构造)。

####类型定义
```c++
class condition_variable
{
public:
  condition_variable();
  ~condition_variable();

  condition_variable(condition_variable const& ) = delete;
  condition_variable& operator=(condition_variable const& ) = delete;

  void notify_one() noexcept;
  void notify_all() noexcept;

  void wait(std::unique_lock<std::mutex>& lock);

  template <typename Predicate>
  void wait(std::unique_lock<std::mutex>& lock,Predicate pred);

  template <typename Clock, typename Duration>
  cv_status wait_until(
       std::unique_lock<std::mutex>& lock,
       const std::chrono::time_point<Clock, Duration>& absolute_time);

  template <typename Clock, typename Duration, typename Predicate>
  bool wait_until(
       std::unique_lock<std::mutex>& lock,
       const std::chrono::time_point<Clock, Duration>& absolute_time,
       Predicate pred);

  template <typename Rep, typename Period>
  cv_status wait_for(
       std::unique_lock<std::mutex>& lock,
       const std::chrono::duration<Rep, Period>& relative_time);

  template <typename Rep, typename Period, typename Predicate>
  bool wait_for(
       std::unique_lock<std::mutex>& lock,
       const std::chrono::duration<Rep, Period>& relative_time,
       Predicate pred);
};

void notify_all_at_thread_exit(condition_variable&,unique_lock<mutex>);
```

####std::condition_variable 默认构造函数

构造一个`std::condition_variable`对象。

**声明**
```c++
condition_variable();
```

**效果**<br>
构造一个新的`std::condition_variable`实例。

**抛出**<br>
当条件变量无法够早的时候，将会抛出一个`std::system_error`异常。

####std::condition_variable 析构函数

销毁一个`std::condition_variable`对象。

**声明**
```c++
~condition_variable();
```

**先决条件**<br>
之前没有使用*this总的wait(),wait_for()或wait_until()阻塞过线程。

**效果**<br>
销毁*this。

**抛出**<br>
无

####std::condition_variable::notify_one 成员函数

唤醒一个等待当前`std::condition_variable`实例的线程。

**声明**
```c++
void notify_one() noexcept;
```

**效果**<br>
唤醒一个等待*this的线程。如果没有线程在等待，那么调用没有任何效果。

**抛出**<br>
当效果没有达成，就会抛出`std::system_error`异常。

**同步**<br>
`std::condition_variable`实例中的notify_one(),notify_all(),wait(),wait_for()和wait_until()都是序列化函数(串行调用)。调用notify_one()或notify_all()只能唤醒正在等待中的线程。

####std::condition_variable::notify_all 成员函数

唤醒所有等待当前`std::condition_variable`实例的线程。

**声明**
```c++
void notify_all() noexcept;
```

**效果**<br>
唤醒所有等待*this的线程。如果没有线程在等待，那么调用没有任何效果。

**抛出**<br>
当效果没有达成，就会抛出`std::system_error`异常

**同步**<br>
`std::condition_variable`实例中的notify_one(),notify_all(),wait(),wait_for()和wait_until()都是序列化函数(串行调用)。调用notify_one()或notify_all()只能唤醒正在等待中的线程。

####std::condition_variable::wait 成员函数

通过`std::condition_variable`的notify_one()、notify_all()或伪唤醒结束等待。

**等待**
```c++
void wait(std::unique_lock<std::mutex>& lock);
```

**先决条件**<br>
当线程调用wait()即可获得锁的所有权,lock.owns_lock()必须为true。

**效果**<br>
自动解锁lock对象，对于线程等待线程，当其他线程调用notify_one()或notify_all()时被唤醒，亦或该线程处于伪唤醒状态。在wait()返回前，lock对象将会再次上锁。

**抛出**<br>
当效果没有达成的时候，将会抛出`std::system_error`异常。当lock对象在调用wait()阶段被解锁，那么当wait()退出的时候lock会再次上锁，即使函数是通过异常的方式退出。

**NOTE**:伪唤醒意味着一个线程调用wait()后，在没有其他线程调用notify_one()或notify_all()时，还处以苏醒状态。因此，建议对wait()进行重载，在可能的情况下使用一个谓词。否则，建议wait()使用循环检查与条件变量相关的谓词。

**同步**<br>
`std::condition_variable`实例中的notify_one(),notify_all(),wait(),wait_for()和wait_until()都是序列化函数(串行调用)。调用notify_one()或notify_all()只能唤醒正在等待中的线程。

####std::condition_variable::wait 需要一个谓词的成员函数重载

等待`std::condition_variable`上的notify_one()或notify_all()被调用，或谓词为true的情况，来唤醒线程。

**声明**
```c++
template<typename Predicate>
void wait(std::unique_lock<std::mutex>& lock,Predicate pred);
```

**先决条件**<br>
pred()谓词必须是合法的，并且需要返回一个值，这个值可以和bool互相转化。当线程调用wait()即可获得锁的所有权,lock.owns_lock()必须为true。

**效果**<br>
正如
```c++
while(!pred())
{
  wait(lock);
}
```

**抛出**<br>
pred中可以抛出任意异常，或者当效果没有达到的时候，抛出`std::system_error`异常。

**NOTE**:潜在的伪唤醒意味着不会指定pred调用的次数。通过lock进行上锁，pred经常会被互斥量引用所调用，并且函数必须返回(只能返回)一个值，在`(bool)pred()`评估后，返回true。

**同步**<br>
`std::condition_variable`实例中的notify_one(),notify_all(),wait(),wait_for()和wait_until()都是序列化函数(串行调用)。调用notify_one()或notify_all()只能唤醒正在等待中的线程。

####std::condition_variable::wait_for 成员函数

`std::condition_variable`在调用notify_one()、调用notify_all()、超时或线程伪唤醒时，结束等待。

**声明**
```c++
template<typename Rep,typename Period>
cv_status wait_for(
    std::unique_lock<std::mutex>& lock,
    std::chrono::duration<Rep,Period> const& relative_time);
```

**先决条件**<br>
当线程调用wait()即可获得锁的所有权,lock.owns_lock()必须为true。

**效果**<br>
当其他线程调用notify_one()或notify_all()函数时，或超出了relative_time的时间，亦或是线程被伪唤醒，则将lock对象自动解锁，并将阻塞线程唤醒。当wait_for()调用返回前，lock对象会再次上锁。

**返回**<br>
线程被notify_one()、notify_all()或伪唤醒唤醒时，会返回`std::cv_status::no_timeout`；反之，则返回`std::cv_status::timeout`。

**抛出**<br>
当效果没有达成的时候，会抛出`std::system_error`异常。当lock对象在调用wait_for()函数前解锁，那么lock对象会在wait_for()退出前再次上锁，即使函数是以异常的方式退出。

**NOTE**:伪唤醒意味着，一个线程在调用wait_for()的时候，即使没有其他线程调用notify_one()和notify_all()函数，也处于苏醒状态。因此，这里建议重载wait_for()函数，重载函数可以使用谓词。要不，则建议wait_for()使用循环的方式对与谓词相关的条件变量进行检查。在这样做的时候还需要小心，以确保超时部分依旧有效；wait_until()可能适合更多的情况。这样的话，线程阻塞的时间就要比指定的时间长了。在有这样可能性的地方，流逝的时间是由稳定时钟决定。

**同步**<br>
`std::condition_variable`实例中的notify_one(),notify_all(),wait(),wait_for()和wait_until()都是序列化函数(串行调用)。调用notify_one()或notify_all()只能唤醒正在等待中的线程。

####std::condition_variable::wait_for 需要一个谓词的成员函数重载

`std::condition_variable`在调用notify_one()、调用notify_all()、超时或线程伪唤醒时，结束等待。

**声明**
```c++
template<typename Rep,typename Period,typename Predicate>
bool wait_for(
    std::unique_lock<std::mutex>& lock,
    std::chrono::duration<Rep,Period> const& relative_time,
    Predicate pred);
```

**先决条件**<br>
pred()谓词必须是合法的，并且需要返回一个值，这个值可以和bool互相转化。当线程调用wait()即可获得锁的所有权,lock.owns_lock()必须为true。

**效果**<br>
等价于
```c++
internal_clock::time_point end=internal_clock::now()+relative_time;
while(!pred())
{
  std::chrono::duration<Rep,Period> remaining_time=
      end-internal_clock::now();
  if(wait_for(lock,remaining_time)==std::cv_status::timeout)
      return pred();
}
return true;
```

**返回**<br>
当pred()为true，则返回true；当超过relative_time并且pred()返回false时，返回false。

**NOTE**:潜在的伪唤醒意味着不会指定pred调用的次数。通过lock进行上锁，pred经常会被互斥量引用所调用，并且函数必须返回(只能返回)一个值，在`(bool)pred()`评估后返回true，或在指定时间relative_time内完成。线程阻塞的时间就要比指定的时间长了。在有这样可能性的地方，流逝的时间是由稳定时钟决定。

**抛出**<br>
当效果没有达成时，会抛出`std::system_error`异常或者由pred抛出任意异常。

**同步**<br>
`std::condition_variable`实例中的notify_one(),notify_all(),wait(),wait_for()和wait_until()都是序列化函数(串行调用)。调用notify_one()或notify_all()只能唤醒正在等待中的线程。

####std::condition_variable::wait_until 成员函数

`std::condition_variable`在调用notify_one()、调用notify_all()、指定时间内达成条件或线程伪唤醒时，结束等待。

**声明**
```c++
template<typename Clock,typename Duration>
cv_status wait_until(
    std::unique_lock<std::mutex>& lock,
    std::chrono::time_point<Clock,Duration> const& absolute_time);
```

**先决条件**<br>
当线程调用wait()即可获得锁的所有权,lock.owns_lock()必须为true。

**效果**<br>
当其他线程调用notify_one()或notify_all()函数，或Clock::now()返回一个大于或等于absolute_time的时间，亦或线程伪唤醒，lock都将自动解锁，并且唤醒阻塞的线程。在wait_until()返回之前lock对象会再次上锁。

**返回**<br>
线程被notify_one()、notify_all()或伪唤醒唤醒时，会返回`std::cv_status::no_timeout`；反之，则返回`std::cv_status::timeout`。

**抛出**<br>
当效果没有达成的时候，会抛出`std::system_error`异常。当lock对象在调用wait_for()函数前解锁，那么lock对象会在wait_for()退出前再次上锁，即使函数是以异常的方式退出。

**NOTE**:伪唤醒意味着一个线程调用wait()后，在没有其他线程调用notify_one()或notify_all()时，还处以苏醒状态。因此，这里建议重载wait_until()函数，重载函数可以使用谓词。要不，则建议wait_until()使用循环的方式对与谓词相关的条件变量进行检查。这里不保证线程会被阻塞多长时间，只有当函数返回false后(Clock::now()的返回值大于或等于absolute_time)，线程才能解除阻塞。

**同步**<br>
`std::condition_variable`实例中的notify_one(),notify_all(),wait(),wait_for()和wait_until()都是序列化函数(串行调用)。调用notify_one()或notify_all()只能唤醒正在等待中的线程。

####std::condition_variable::wait_until 需要一个谓词的成员函数重载

`std::condition_variable`在调用notify_one()、调用notify_all()、谓词返回true或指定时间内达到条件，结束等待。

**声明**
```c++
template<typename Clock,typename Duration,typename Predicate>
bool wait_until(
    std::unique_lock<std::mutex>& lock,
    std::chrono::time_point<Clock,Duration> const& absolute_time,
    Predicate pred);
```

**先决条件**<br>
pred()必须是合法的，并且其返回值能转换为bool值。当线程调用wait()即可获得锁的所有权,lock.owns_lock()必须为true。

**效果**<br>
等价于
```c++
while(!pred())
{
  if(wait_until(lock,absolute_time)==std::cv_status::timeout)
    return pred();
}
return true;
```

**返回**<br>
当调用pred()返回true时，返回true；当Clock::now()的时间大于或等于指定的时间absolute_time，并且pred()返回false时，返回false。

**NOTE**:潜在的伪唤醒意味着不会指定pred调用的次数。通过lock进行上锁，pred经常会被互斥量引用所调用，并且函数必须返回(只能返回)一个值，在`(bool)pred()`评估后返回true，或Clock::now()返回的时间大于或等于absolute_time。这里不保证调用线程将被阻塞的时长，只有当函数返回false后(Clock::now()返回一个等于或大于absolute_time的值)，线程接触阻塞。

**抛出**<br>
当效果没有达成时，会抛出`std::system_error`异常或者由pred抛出任意异常。

**同步**<br>
`std::condition_variable`实例中的notify_one(),notify_all(),wait(),wait_for()和wait_until()都是序列化函数(串行调用)。调用notify_one()或notify_all()只能唤醒正在等待中的线程。

####std::notify_all_at_thread_exit 非成员函数

当当前调用函数的线程退出时，等待`std::condition_variable`的所有线程将会被唤醒。

**声明**
```c++
void notify_all_at_thread_exit(
  condition_variable& cv,unique_lock<mutex> lk);
```

**先决条件**<br>
当线程调用wait()即可获得锁的所有权,lk.owns_lock()必须为true。lk.mutex()需要返回的值要与并发等待线程相关的任意cv中锁对象提供的wait(),wait_for()或wait_until()相同。

**效果**<br>
将lk的所有权转移到内部存储中，并且当有线程退出时，安排被提醒的cv类。这里的提醒等价于
```c++
lk.unlock();
cv.notify_all();
```

**抛出**<br>
当效果没有达成时，抛出`std::system_error`异常。

**NOTE**:在线程退出前，掌握着锁的所有权，所以这里要避免死锁发生。这里建议调用该函数的线程应该尽快退出，并且在该线程可以执行一些阻塞的操作。用户必须保证等地线程不会错误的将唤醒线程当做已退出的线程，特别是伪唤醒。可以通过等待线程上的谓词测试来实现这一功能，在互斥量保护的情况下，只有谓词返回true时线程才能被唤醒，并且在调用notify_all_at_thread_exit(std::condition_variable_any类中函数)前是不会释放锁。

###D.2.2 std::condition_variable_any类

`std::condition_variable_any`类允许线程等待某一条件为true的时候继续运行。不过`std::condition_variable`只能和`std::unique_lock<std::mutex>`一起使用，`std::condition_variable_any`可以和任意可上锁(Lockable)类型一起使用。

`std::condition_variable_any`实例不能进行拷贝赋值(CopyAssignable)、拷贝构造(CopyConstructible)、移动赋值(MoveAssignable)或移动构造(MoveConstructible)。

####类型定义
```c++
class condition_variable_any
{
public:
  condition_variable_any();
  ~condition_variable_any();

  condition_variable_any(
      condition_variable_any const& ) = delete;
  condition_variable_any& operator=(
      condition_variable_any const& ) = delete;

  void notify_one() noexcept;
  void notify_all() noexcept;

  template<typename Lockable>
  void wait(Lockable& lock);

  template <typename Lockable, typename Predicate>
  void wait(Lockable& lock, Predicate pred);

  template <typename Lockable, typename Clock,typename Duration>
  std::cv_status wait_until(
      Lockable& lock,
      const std::chrono::time_point<Clock, Duration>& absolute_time);

  template <
      typename Lockable, typename Clock,
      typename Duration, typename Predicate>
  bool wait_until(
      Lockable& lock,
      const std::chrono::time_point<Clock, Duration>& absolute_time,
      Predicate pred);

  template <typename Lockable, typename Rep, typename Period>
  std::cv_status wait_for(
      Lockable& lock,
      const std::chrono::duration<Rep, Period>& relative_time);

  template <
      typename Lockable, typename Rep,
      typename Period, typename Predicate>
  bool wait_for(
      Lockable& lock,
      const std::chrono::duration<Rep, Period>& relative_time,
      Predicate pred);
};
```

####std::condition_variable_any 默认构造函数

构造一个`std::condition_variable_any`对象。

**声明**
```c++
condition_variable_any();
```

**效果**<br>
构造一个新的`std::condition_variable_any`实例。

**抛出**<br>
当条件变量构造成功，将抛出`std::system_error`异常。

####std::condition_variable_any 析构函数

销毁`std::condition_variable_any`对象。

**声明**
```c++
~condition_variable_any();
```

**先决条件**<br>
之前没有使用*this总的wait(),wait_for()或wait_until()阻塞过线程。

**效果**<br>
销毁*this。

**抛出**<br>
无

####std::condition_variable_any::notify_one 成员函数

`std::condition_variable_any`唤醒一个等待该条件变量的线程。

**声明**
```c++
void notify_all() noexcept;
```

**效果**<br>
唤醒一个等待*this的线程。如果没有线程在等待，那么调用没有任何效果

**抛出**<br>
当效果没有达成，就会抛出std::system_error异常。

**同步**<br>
`std::condition_variable`实例中的notify_one(),notify_all(),wait(),wait_for()和wait_until()都是序列化函数(串行调用)。调用notify_one()或notify_all()只能唤醒正在等待中的线程。

####std::condition_variable_any::notify_all 成员函数

唤醒所有等待当前`std::condition_variable_any`实例的线程。

**声明**
```c++
void notify_all() noexcept;
```

**效果**<br>
唤醒所有等待*this的线程。如果没有线程在等待，那么调用没有任何效果

**抛出**<br>
当效果没有达成，就会抛出std::system_error异常。

**同步**<br>
`std::condition_variable`实例中的notify_one(),notify_all(),wait(),wait_for()和wait_until()都是序列化函数(串行调用)。调用notify_one()或notify_all()只能唤醒正在等待中的线程。

####std::condition_variable_any::wait 成员函数

通过`std::condition_variable_any`的notify_one()、notify_all()或伪唤醒结束等待。

**声明**
```c++
template<typename Lockable>
void wait(Lockable& lock);
```

**先决条件**<br>
Lockable类型需要能够上锁，lock对象拥有一个锁。

**效果**<br>
自动解锁lock对象，对于线程等待线程，当其他线程调用notify_one()或notify_all()时被唤醒，亦或该线程处于伪唤醒状态。在wait()返回前，lock对象将会再次上锁。

**抛出**<br>
当效果没有达成的时候，将会抛出`std::system_error`异常。当lock对象在调用wait()阶段被解锁，那么当wait()退出的时候lock会再次上锁，即使函数是通过异常的方式退出。

**NOTE**:伪唤醒意味着一个线程调用wait()后，在没有其他线程调用notify_one()或notify_all()时，还处以苏醒状态。因此，建议对wait()进行重载，在可能的情况下使用一个谓词。否则，建议wait()使用循环检查与条件变量相关的谓词。

**同步**<br>
std::condition_variable_any实例中的notify_one(),notify_all(),wait(),wait_for()和wait_until()都是序列化函数(串行调用)。调用notify_one()或notify_all()只能唤醒正在等待中的线程。

####std::condition_variable_any::wait 需要一个谓词的成员函数重载

等待`std::condition_variable_any`上的notify_one()或notify_all()被调用，或谓词为true的情况，来唤醒线程。

**声明**
```c++
template<typename Lockable,typename Predicate>
void wait(Lockable& lock,Predicate pred);
```

**先决条件**<br>
pred()谓词必须是合法的，并且需要返回一个值，这个值可以和bool互相转化。当线程调用wait()即可获得锁的所有权,lock.owns_lock()必须为true。

**效果**<br>
正如
```c++
while(!pred())
{
wait(lock);
}
```

**抛出**<br>
pred中可以抛出任意异常，或者当效果没有达到的时候，抛出`std::system_error`异常。

**NOTE**:潜在的伪唤醒意味着不会指定pred调用的次数。通过lock进行上锁，pred经常会被互斥量引用所调用，并且函数必须返回(只能返回)一个值，在`(bool)pred()`评估后，返回true。

**同步**<br>
`std::condition_variable_any`实例中的notify_one(),notify_all(),wait(),wait_for()和wait_until()都是序列化函数(串行调用)。调用notify_one()或notify_all()只能唤醒正在等待中的线程。

####std::condition_variable_any::wait_for 成员函数

`std::condition_variable_any`在调用notify_one()、调用notify_all()、超时或线程伪唤醒时，结束等待。

**声明**
```c++
template<typename Lockable,typename Rep,typename Period>
std::cv_status wait_for(
    Lockable& lock,
    std::chrono::duration<Rep,Period> const& relative_time);
```

**先决条件**<br>
当线程调用wait()即可获得锁的所有权,lock.owns_lock()必须为true。

**效果**<br>
当其他线程调用notify_one()或notify_all()函数时，或超出了relative_time的时间，亦或是线程被伪唤醒，则将lock对象自动解锁，并将阻塞线程唤醒。当wait_for()调用返回前，lock对象会再次上锁。

**返回**<br>
线程被notify_one()、notify_all()或伪唤醒唤醒时，会返回`std::cv_status::no_timeout`；反之，则返回std::cv_status::timeout。

**抛出**<br>
当效果没有达成的时候，会抛出`std::system_error`异常。当lock对象在调用wait_for()函数前解锁，那么lock对象会在wait_for()退出前再次上锁，即使函数是以异常的方式退出。

**NOTE**:伪唤醒意味着，一个线程在调用wait_for()的时候，即使没有其他线程调用notify_one()和notify_all()函数，也处于苏醒状态。因此，这里建议重载wait_for()函数，重载函数可以使用谓词。要不，则建议wait_for()使用循环的方式对与谓词相关的条件变量进行检查。在这样做的时候还需要小心，以确保超时部分依旧有效；wait_until()可能适合更多的情况。这样的话，线程阻塞的时间就要比指定的时间长了。在有这样可能性的地方，流逝的时间是由稳定时钟决定。

**同步**<br>
`std::condition_variable_any`实例中的notify_one(),notify_all(),wait(),wait_for()和wait_until()都是序列化函数(串行调用)。调用notify_one()或notify_all()只能唤醒正在等待中的线程。

####std::condition_variable_any::wait_for 需要一个谓词的成员函数重载

`std::condition_variable_any`在调用notify_one()、调用notify_all()、超时或线程伪唤醒时，结束等待。

**声明**
```c++
template<typename Lockable,typename Rep,
    typename Period, typename Predicate>
bool wait_for(
    Lockable& lock,
    std::chrono::duration<Rep,Period> const& relative_time,
    Predicate pred);
```

**先决条件**<br>
pred()谓词必须是合法的，并且需要返回一个值，这个值可以和bool互相转化。当线程调用wait()即可获得锁的所有权,lock.owns_lock()必须为true。

**效果**<br>
正如
```c++
internal_clock::time_point end=internal_clock::now()+relative_time;
while(!pred())
{
  std::chrono::duration<Rep,Period> remaining_time=
      end-internal_clock::now();
  if(wait_for(lock,remaining_time)==std::cv_status::timeout)
      return pred();
}
return true;
```

**返回**<br>
当pred()为true，则返回true；当超过relative_time并且pred()返回false时，返回false。

**NOTE**:
潜在的伪唤醒意味着不会指定pred调用的次数。通过lock进行上锁，pred经常会被互斥量引用所调用，并且函数必须返回(只能返回)一个值，在(bool)pred()评估后返回true，或在指定时间relative_time内完成。线程阻塞的时间就要比指定的时间长了。在有这样可能性的地方，流逝的时间是由稳定时钟决定。

**抛出**<br>
当效果没有达成时，会抛出`std::system_error`异常或者由pred抛出任意异常。

**同步**<br>
`std::condition_variable_any`实例中的notify_one(),notify_all(),wait(),wait_for()和wait_until()都是序列化函数(串行调用)。调用notify_one()或notify_all()只能唤醒正在等待中的线程。

####std::condition_variable_any::wait_until 成员函数

`std::condition_variable_any`在调用notify_one()、调用notify_all()、指定时间内达成条件或线程伪唤醒时，结束等待

**声明**
```c++
template<typename Lockable,typename Clock,typename Duration>
std::cv_status wait_until(
    Lockable& lock,
    std::chrono::time_point<Clock,Duration> const& absolute_time);
```

**先决条件**<br>
Lockable类型需要能够上锁，lock对象拥有一个锁。

**效果**<br>
当其他线程调用notify_one()或notify_all()函数，或Clock::now()返回一个大于或等于absolute_time的时间，亦或线程伪唤醒，lock都将自动解锁，并且唤醒阻塞的线程。在wait_until()返回之前lock对象会再次上锁。

**返回**<br>
线程被notify_one()、notify_all()或伪唤醒唤醒时，会返回std::cv_status::no_timeout；反之，则返回`std::cv_status::timeout`。

**抛出**<br>
当效果没有达成的时候，会抛出`std::system_error`异常。当lock对象在调用wait_for()函数前解锁，那么lock对象会在wait_for()退出前再次上锁，即使函数是以异常的方式退出。

**NOTE**:伪唤醒意味着一个线程调用wait()后，在没有其他线程调用notify_one()或notify_all()时，还处以苏醒状态。因此，这里建议重载wait_until()函数，重载函数可以使用谓词。要不，则建议wait_until()使用循环的方式对与谓词相关的条件变量进行检查。这里不保证线程会被阻塞多长时间，只有当函数返回false后(Clock::now()的返回值大于或等于absolute_time)，线程才能解除阻塞。

**同步**
`std::condition_variable_any`实例中的notify_one(),notify_all(),wait(),wait_for()和wait_until()都是序列化函数(串行调用)。调用notify_one()或notify_all()只能唤醒正在等待中的线程。

####std::condition_variable_any::wait_unti 需要一个谓词的成员函数重载

`std::condition_variable_any`在调用notify_one()、调用notify_all()、谓词返回true或指定时间内达到条件，结束等待。

**声明**
```c++
template<typename Lockable,typename Clock,
    typename Duration, typename Predicate>
bool wait_until(
    Lockable& lock,
    std::chrono::time_point<Clock,Duration> const& absolute_time,
    Predicate pred);
```

**先决条件**<br>
pred()必须是合法的，并且其返回值能转换为bool值。当线程调用wait()即可获得锁的所有权,lock.owns_lock()必须为true。

**效果**<br>
等价于
```c++
while(!pred())
{
  if(wait_until(lock,absolute_time)==std::cv_status::timeout)
    return pred();
}
return true;
```

**返回**<br>
当调用pred()返回true时，返回true；当Clock::now()的时间大于或等于指定的时间absolute_time，并且pred()返回false时，返回false。

**NOTE**：潜在的伪唤醒意味着不会指定pred调用的次数。通过lock进行上锁，pred经常会被互斥量引用所调用，并且函数必须返回(只能返回)一个值，在(bool)pred()评估后返回true，或Clock::now()返回的时间大于或等于absolute_time。这里不保证调用线程将被阻塞的时长，只有当函数返回false后(Clock::now()返回一个等于或大于absolute_time的值)，线程接触阻塞。

**抛出**<br>
当效果没有达成时，会抛出`std::system_error`异常或者由pred抛出任意异常。

**同步**<br>
`std::condition_variable_any`实例中的notify_one(),notify_all(),wait(),wait_for()和wait_until()都是序列化函数(串行调用)。调用notify_one()或notify_all()只能唤醒正在等待中的线程。
 
##D.3 <atomic>头文件

<atomic>头文件提供一组基础的原子类型，和提供对这些基本类型的操作，以及一个原子模板函数，用来接收用户定义的类型，以适用于某些标准。

####头文件内容
```c++
#define ATOMIC_BOOL_LOCK_FREE 参见详述
#define ATOMIC_CHAR_LOCK_FREE 参见详述
#define ATOMIC_SHORT_LOCK_FREE 参见详述
#define ATOMIC_INT_LOCK_FREE 参见详述
#define ATOMIC_LONG_LOCK_FREE 参见详述
#define ATOMIC_LLONG_LOCK_FREE 参见详述
#define ATOMIC_CHAR16_T_LOCK_FREE 参见详述
#define ATOMIC_CHAR32_T_LOCK_FREE 参见详述
#define ATOMIC_WCHAR_T_LOCK_FREE 参见详述
#define ATOMIC_POINTER_LOCK_FREE 参见详述

#define ATOMIC_VAR_INIT(value) 参见详述

namespace std
{
  enum memory_order;

  struct atomic_flag;
  参见类型定义详述 atomic_bool;
  参见类型定义详述 atomic_char;
  参见类型定义详述 atomic_char16_t;
  参见类型定义详述 atomic_char32_t;
  参见类型定义详述 atomic_schar;
  参见类型定义详述 atomic_uchar;
  参见类型定义详述 atomic_short;
  参见类型定义详述 atomic_ushort;
  参见类型定义详述 atomic_int;
  参见类型定义详述 atomic_uint;
  参见类型定义详述 atomic_long;
  参见类型定义详述 atomic_ulong;
  参见类型定义详述 atomic_llong;
  参见类型定义详述 atomic_ullong;
  参见类型定义详述 atomic_wchar_t;

  参见类型定义详述 atomic_int_least8_t;
  参见类型定义详述 atomic_uint_least8_t;
  参见类型定义详述 atomic_int_least16_t;
  参见类型定义详述 atomic_uint_least16_t;
  参见类型定义详述 atomic_int_least32_t;
  参见类型定义详述 atomic_uint_least32_t;
  参见类型定义详述 atomic_int_least64_t;
  参见类型定义详述 atomic_uint_least64_t;
  参见类型定义详述 atomic_int_fast8_t;
  参见类型定义详述 atomic_uint_fast8_t;
  参见类型定义详述 atomic_int_fast16_t;
  参见类型定义详述 atomic_uint_fast16_t;
  参见类型定义详述 atomic_int_fast32_t;
  参见类型定义详述 atomic_uint_fast32_t;
  参见类型定义详述 atomic_int_fast64_t;
  参见类型定义详述 atomic_uint_fast64_t;
  参见类型定义详述 atomic_int8_t;
  参见类型定义详述 atomic_uint8_t;
  参见类型定义详述 atomic_int16_t;
  参见类型定义详述 atomic_uint16_t;
  参见类型定义详述 atomic_int32_t;
  参见类型定义详述 atomic_uint32_t;
  参见类型定义详述 atomic_int64_t;
  参见类型定义详述 atomic_uint64_t;
  参见类型定义详述 atomic_intptr_t;
  参见类型定义详述 atomic_uintptr_t;
  参见类型定义详述 atomic_size_t;
  参见类型定义详述 atomic_ssize_t;
  参见类型定义详述 atomic_ptrdiff_t;
  参见类型定义详述 atomic_intmax_t;
  参见类型定义详述 atomic_uintmax_t;

  template<typename T>
  struct atomic;

  extern "C" void atomic_thread_fence(memory_order order);
  extern "C" void atomic_signal_fence(memory_order order);

  template<typename T>
  T kill_dependency(T);
}
```

###std::atomic_xxx类型定义

为了兼容新的C标准(C11)，C++支持定义原子整型类型。这些类型都与`std::atimic<T>`特化类相对应，或是用同一接口特化的一个基本类型。

**Table D.1 原子类型定义和与之相关的std::atmoic<>特化模板**

| std::atomic_itype 原子类型 | std::atomic<> 相关特化类 |
| ------------ | -------------- |
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
| atomic_wchar_t | std::atomic&lt;wchar_t> |
| atomic_char16_t | std::atomic&lt;char16_t> |
| atomic_char32_t | std::atomic&lt;char32_t> |

(译者注：该表与第5章中的表5.1几乎一致)

###D.3.2 ATOMIC_xxx_LOCK_FREE宏

这里的宏指定了原子类型与其内置类型是否是无锁的。

**宏定义**
```c++
#define ATOMIC_BOOL_LOCK_FREE 参见详述
#define ATOMIC_CHAR_LOCK_FREE参见详述
#define ATOMIC_SHORT_LOCK_FREE 参见详述
#define ATOMIC_INT_LOCK_FREE 参见详述
#define ATOMIC_LONG_LOCK_FREE 参见详述
#define ATOMIC_LLONG_LOCK_FREE 参见详述
#define ATOMIC_CHAR16_T_LOCK_FREE 参见详述
#define ATOMIC_CHAR32_T_LOCK_FREE 参见详述
#define ATOMIC_WCHAR_T_LOCK_FREE 参见详述
#define ATOMIC_POINTER_LOCK_FREE 参见详述
```

`ATOMIC_xxx_LOCK_FREE`的值无非就是0，1，2。0意味着，在对有无符号的相关原子类型操作是有锁的；1意味着，操作只对一些特定的类型上锁，而对没有指定的类型不上锁；2意味着，所有操作都是无锁的。例如，当`ATOMIC_INT_LOCK_FREE`是2的时候，在`std::atomic<int>`和`std::atomic<unsigned>`上的操作始终无锁。

宏`ATOMIC_POINTER_LOCK_FREE`描述了，对于特化的原子类型指针`std::atomic<T*>`操作的无锁特性。

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