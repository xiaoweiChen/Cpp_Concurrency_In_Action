#附录D C++线程库参考

##D.1 &lt;chrono&gt;头文件

&lt;chrono&gt;头文件作为`time_point`的提供者，具有代表时间点的类，duration类和时钟类。每个时钟都有一个`is_steady`静态数据成员，这个成员用来表示该时钟是否是一个*稳定的*时钟(以匀速计时的时钟，且不可调节)。`std::chrono::steady_clock`是唯一个能保证稳定的时钟类。

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
template <class Rep2>;
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
template <class Rep2, class Period>
constexpr duration(const duration<Rep2,Period2>& d);
```

**结果**<br>
duration对象的内部值通过`duration_cast<duration<Rep,Period>>(d).count()`初始化。

**要求**<br>
当Rep是一个浮点类或Rep2不是浮点类，且Period2是Period数的倍数(比如，ratio_divide&lt;Period2,Period&gt;::den==1)时，才能调用该重载。当一个较小的数据转换为一个较大的数据时，使用该构造函数就能避免数位截断和精度损失。

**后验条件**<br>
`this->count() == dutation_cast&lt;duration<Rep, Period>>(d).count()`

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
`duration(--this->count());`

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
当`CommonDuration`和`std::common_type< duration< Rep1, Period1>, duration< Rep2, Period2>>::type`同类，那么`lhs&lt;rhs`就会返回`CommonDuration(lhs).count()&lt;CommonDuration(rhs).count()`。

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
将d的值和duration对象的值相加，存储在*this中，就如同this-&gt;internal_duration += d;

**返回**
`*this`

####std::chrono::time_point::operator-= 复合赋值函数

将指定的duration的值与原存储在指定的time_point对象中的duration相减，并将加后值存储在*this对象中。

**声明**
```c++
time_point& operator-=(const duration& d);
```

**效果**<br>
将d的值和duration对象的值相减，存储在*this中，就如同this-&gt;internal_duration -= d;

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
typedef std::chrono::time_point&lt;std::chrono::system_clock&gt; time_point;
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

##D.2 &lt;condition_variable&gt;头文件

&lt;condition_variable&gt;头文件提供了条件变量的定义。其作为基本同步机制，允许被阻塞的线程在某些条件达成或超时时，解除阻塞继续执行。

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
 
##D.3 &lt;atomic&gt;头文件

&lt;atomic&gt;头文件提供一组基础的原子类型，和提供对这些基本类型的操作，以及一个原子模板函数，用来接收用户定义的类型，以适用于某些标准。

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

为了兼容新的C标准(C11)，C++支持定义原子整型类型。这些类型都与`std::atimic<T>;`特化类相对应，或是用同一接口特化的一个基本类型。

**Table D.1 原子类型定义和与之相关的std::atmoic&lt;&gt;特化模板**

| std::atomic_itype 原子类型 | std::atomic&lt;&gt; 相关特化类 |
| ------------ | -------------- |
| atomic_char | std::atomic&lt;char&gt; |
| atomic_schar | std::atomic&lt;signed char&gt; |
| atomic_uchar | std::atomic&lt;unsigned char&gt; |
| atomic_int | std::atomic&lt;int&gt; |
| atomic_uint | std::atomic&lt;unsigned&gt; |
| atomic_short | std::atomic&lt;short&gt; |
| atomic_ushort | std::atomic&lt;unsigned short&gt; |
| atomic_long | std::atomic&lt;long&gt; |
| atomic_ulong | std::atomic&lt;unsigned long&gt; |
| atomic_llong | std::atomic&lt;long long&gt; |
| atomic_ullong | std::atomic&lt;unsigned long long&gt; |
| atomic_wchar_t | std::atomic&lt;wchar_t&gt; |
| atomic_char16_t | std::atomic&lt;char16_t&gt; |
| atomic_char32_t | std::atomic&lt;char32_t&gt; |

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

`ATOMIC_xxx_LOCK_FREE`的值无非就是0，1，2。0意味着，在对有无符号的相关原子类型操作是有锁的；1意味着，操作只对一些特定的类型上锁，而对没有指定的类型不上锁；2意味着，所有操作都是无锁的。例如，当`ATOMIC_INT_LOCK_FREE`是2的时候，在`std::atomic&lt;int&gt;`和`std::atomic&lt;unsigned&gt;`上的操作始终无锁。

宏`ATOMIC_POINTER_LOCK_FREE`描述了，对于特化的原子类型指针`std::atomic<T*>`操作的无锁特性。

###D.3.3 ATOMIC_VAR_INIT宏

`ATOMIC_VAR_INIT`宏可以通过一个特定的值来初始化一个原子变量。

**声明**
`#define ATOMIC_VAR_INIT(value)参见详述`

宏可以扩展成一系列符号，这个宏可以通过一个给定值，初始化一个标准原子类型，表达式如下所示：
```c++
std::atomic<type> x = ATOMIC_VAR_INIT(val);
```

给定值可以兼容与原子变量相关的非原子变量，例如：

```c++
std::atomic&lt;int> i = ATOMIC_VAR_INIT(42);
std::string s;
std::atomic&lt;std::string*> p = ATOMIC_VAR_INIT(&s);
```

这样初始化的变量是非原子的，并且在变量初始化之后，其他线程可以随意的访问该变量，这样可以避免条件竞争和未定义行为的发生。

###D.3.4 std::memory_order枚举类型

`std::memory_order`枚举类型用来表明原子操作的约束顺序。

**声明**
```c++
typedef enum memory_order
{
  memory_order_relaxed,memory_order_consume,
  memory_order_acquire,memory_order_release,
  memory_order_acq_rel,memory_order_seq_cst
} memory_order;
```

通过标记各种内存序变量来标记操作的顺序(详见第5章，在该章节中有对书序约束更加详尽的介绍)

####std::memory_order_relaxed

操作不受任何额外的限制。

####std::memory_order_release

对于指定位置上的内存可进行释放操作。因此，与获取操作读取同一内存位置所存储的值。

####std::memory_order_acquire

操作可以获取指定内存位置上的值。当需要存储的值通过释放操作写入时，是与存储操同步的。

####std::memory_order_acq_rel

操作必须是“读-改-写”操作，并且其行为需要在`std::memory_order_acquire`和`std::memory_order_release`序指定的内存位置上进行操作。

####std::memory_order_seq_cst

操作在全局序上都会受到约束。还有，当为存储操作时，其行为好比`std::memory_order_release`操作；当为加载操作时，其行为好比`std::memory_order_acquire`操作；并且，当其是一个“读-改-写”操作时，其行为和`std::memory_order_acquire`和`std::memory_order_release`类似。对于所有顺序来说，该顺序为默认序。

####std::memory_order_consume

对于指定位置的内存进行消耗操作(consume operation)。

(译者注：与memory_order_acquire类似)

###D.3.5 std::atomic_thread_fence函数

`std::atomic_thread_fence()`会在代码中插入“内存栅栏”，强制两个操作保持内存约束顺序。

**声明**
```c++
extern "C" void atomic_thread_fence(std::memory_order order);
```

**效果**<br>
插入栅栏的目的是为了保证内存序的约束性。

栅栏使用`std::memory_order_release`, `std::memory_order_acq_rel`, 或 `std::memory_order_seq_cst`内存序，会同步与一些内存位置上的获取操作进行同步，如果这些获取操作要获取一个已存储的值(通过原子操作进行的存储)，就会通过栅栏进行同步。

释放操作可对`std::memory_order_acquire`, `std::memory_order_acq_rel`, 或 `std::memory_order_seq_cst`进行栅栏同步，；当释放操作存储的值，在一个原子操作之前读取，那么就会通过栅栏进行同步。

**抛出**<br>
无

###D.3.6 std::atomic_signal_fence函数

`std::atomic_signal_fence()`会在代码中插入“内存栅栏”，强制两个操作保持内存约束顺序，并且在对应线程上执行信号处理操作。

**声明**
```c++
extern "C" void atomic_signal_fence(std::memory_order order);
```

**效果**<br>
根据需要的内存约束序插入一个栅栏。除非约束序应用于“操作和信号处理函数在同一线程”的情况下，否则，这个操作等价于`std::atomic_thread_fence(order)`操作。

**抛出**<br>
无

###D.3.7 std::atomic_flag类

`std::atomic_flag`类算是原子标识的骨架。在C++11标准下，只有这个数据类型可以保证是无锁的(当然，更多的原子类型在未来的实现中将采取无锁实现)。

对于一个`std::atomic_flag`来说，其状态不是set，就是clear。

**类型定义**
```c++
struct atomic_flag
{
  atomic_flag() noexcept = default;
  atomic_flag(const atomic_flag&) = delete;
  atomic_flag& operator=(const atomic_flag&) = delete;
  atomic_flag& operator=(const atomic_flag&) volatile = delete;

  bool test_and_set(memory_order = memory_order_seq_cst) volatile
    noexcept;
  bool test_and_set(memory_order = memory_order_seq_cst) noexcept;
  void clear(memory_order = memory_order_seq_cst) volatile noexcept;
  void clear(memory_order = memory_order_seq_cst) noexcept;
};

bool atomic_flag_test_and_set(volatile atomic_flag*) noexcept;
bool atomic_flag_test_and_set(atomic_flag*) noexcept;
bool atomic_flag_test_and_set_explicit(
  volatile atomic_flag*, memory_order) noexcept;
bool atomic_flag_test_and_set_explicit(
  atomic_flag*, memory_order) noexcept;
void atomic_flag_clear(volatile atomic_flag*) noexcept;
void atomic_flag_clear(atomic_flag*) noexcept;
void atomic_flag_clear_explicit(
  volatile atomic_flag*, memory_order) noexcept;
void atomic_flag_clear_explicit(
  atomic_flag*, memory_order) noexcept;

#define ATOMIC_FLAG_INIT unspecified
```

####std::atomic_flag 默认构造函数

这里未指定默认构造出来的`std::atomic_flag`实例是clear状态，还是set状态。因为对象存储过程是静态的，所以初始化必须是静态的。

**声明**
```c++
std::atomic_flag() noexcept = default;
```

**效果**<br>
构造一个新`std::atomic_flag`对象，不过未指明状态。(薛定谔的猫？)

**抛出**<br>
无

####std::atomic_flag 使用ATOMIC_FLAG_INIT进行初始化

`std::atomic_flag`实例可以使用`ATOMIC_FLAG_INIT`宏进行创建，这样构造出来的实例状态为clear。因为对象存储过程是静态的，所以初始化必须是静态的。

**声明**
```c++
#define ATOMIC_FLAG_INIT unspecified
```

**用法**
```c++
std::atomic_flag flag=ATOMIC_FLAG_INIT;
```

**效果**<br>
构造一个新`std::atomic_flag`对象，状态为clear。

**抛出**<br>
无

**NOTE**：
对于内存位置上的*this，这个操作属于“读-改-写”操作。

####std::atomic_flag::test_and_set 成员函数

自动设置实例状态标识，并且检查实例的状态标识是否已经设置。

**声明**
```c++
bool atomic_flag_test_and_set(volatile atomic_flag* flag) noexcept;
bool atomic_flag_test_and_set(atomic_flag* flag) noexcept;
```

**效果**
```c++
return flag->test_and_set();
```

####std::atomic_flag_test_and_set 非成员函数

自动设置原子变量的状态标识，并且检查原子变量的状态标识是否已经设置。

**声明**
```c++
bool atomic_flag_test_and_set_explicit(
    volatile atomic_flag* flag, memory_order order) noexcept;
bool atomic_flag_test_and_set_explicit(
    atomic_flag* flag, memory_order order) noexcept;
```

**效果**
```c++
return flag->test_and_set(order);
```

####std::atomic_flag_test_and_set_explicit 非成员函数

自动设置原子变量的状态标识，并且检查原子变量的状态标识是否已经设置。

**声明**
```c++
bool atomic_flag_test_and_set_explicit(
    volatile atomic_flag* flag, memory_order order) noexcept;
bool atomic_flag_test_and_set_explicit(
    atomic_flag* flag, memory_order order) noexcept;
```

**效果**
```c++
return flag->test_and_set(order);
```

####std::atomic_flag::clear 成员函数

自动清除原子变量的状态标识。

**声明**
```c++
void clear(memory_order order = memory_order_seq_cst) volatile noexcept;
void clear(memory_order order = memory_order_seq_cst) noexcept;
```

**先决条件**<br>
支持`std::memory_order_relaxed`,`std::memory_order_release`和`std::memory_order_seq_cst`中任意一个。


**效果**<br>
自动清除变量状态标识。

**抛出**<br>
无

**NOTE**:对于内存位置上的*this，这个操作属于“写”操作(存储操作)。


####std::atomic_flag_clear 非成员函数

自动清除原子变量的状态标识。

**声明**
```c++
void atomic_flag_clear(volatile atomic_flag* flag) noexcept;
void atomic_flag_clear(atomic_flag* flag) noexcept;
```

**效果**
```c++
flag->clear();
```

####std::atomic_flag_clear_explicit 非成员函数

自动清除原子变量的状态标识。

**声明**
```c++
void atomic_flag_clear_explicit(
    volatile atomic_flag* flag, memory_order order) noexcept;
void atomic_flag_clear_explicit(
    atomic_flag* flag, memory_order order) noexcept;
```

**效果**
```c++
return flag->clear(order);
```

###D.3.8 std::atomic类型模板

`std::atomic`提供了对任意类型的原子操作的包装，以满足下面的需求。

模板参数BaseType必须满足下面的条件。

- 具有简单的默认构造函数<br>
- 具有简单的拷贝赋值操作<br>
- 具有简单的析构函数<br>
- 可以进行位比较<br>

这就意味着`std::atomic&lt;some-simple-struct&gt;`会和使用`std::atomic<some-built-in-type>`一样简单；不过对于`std::atomic<std::string>`就不同了。

除了主模板，对于内置整型和指针的特化，模板也支持类似x++这样的操作。

`std::atomic`实例是不支持`CopyConstructible`(拷贝构造)和`CopyAssignable`(拷贝赋值)，原因你懂得，因为这样原子操作就无法执行。

**类型定义**
```c++
template<typename BaseType>
struct atomic
{
  atomic() noexcept = default;
  constexpr atomic(BaseType) noexcept;
  BaseType operator=(BaseType) volatile noexcept;
  BaseType operator=(BaseType) noexcept;

  atomic(const atomic&) = delete;
  atomic& operator=(const atomic&) = delete;
  atomic& operator=(const atomic&) volatile = delete;

  bool is_lock_free() const volatile noexcept;
  bool is_lock_free() const noexcept;

  void store(BaseType,memory_order = memory_order_seq_cst)
      volatile noexcept;
  void store(BaseType,memory_order = memory_order_seq_cst) noexcept;
  BaseType load(memory_order = memory_order_seq_cst)
      const volatile noexcept;
  BaseType load(memory_order = memory_order_seq_cst) const noexcept;
  BaseType exchange(BaseType,memory_order = memory_order_seq_cst)
      volatile noexcept;
  BaseType exchange(BaseType,memory_order = memory_order_seq_cst)
      noexcept;

  bool compare_exchange_strong(
      BaseType & old_value, BaseType new_value,
      memory_order order = memory_order_seq_cst) volatile noexcept;
  bool compare_exchange_strong(
      BaseType & old_value, BaseType new_value,
      memory_order order = memory_order_seq_cst) noexcept;
  bool compare_exchange_strong(
      BaseType & old_value, BaseType new_value,
      memory_order success_order,
      memory_order failure_order) volatile noexcept;
  bool compare_exchange_strong(
      BaseType & old_value, BaseType new_value,
      memory_order success_order,
      memory_order failure_order) noexcept;
  bool compare_exchange_weak(
      BaseType & old_value, BaseType new_value,
      memory_order order = memory_order_seq_cst)
      volatile noexcept;
  bool compare_exchange_weak(
      BaseType & old_value, BaseType new_value,
      memory_order order = memory_order_seq_cst) noexcept;
  bool compare_exchange_weak(
      BaseType & old_value, BaseType new_value,
      memory_order success_order,
      memory_order failure_order) volatile noexcept;
  bool compare_exchange_weak(
      BaseType & old_value, BaseType new_value,
      memory_order success_order,
      memory_order failure_order) noexcept;
      operator BaseType () const volatile noexcept;
      operator BaseType () const noexcept;
};

template<typename BaseType>
bool atomic_is_lock_free(volatile const atomic<BaseType>*) noexcept;
template<typename BaseType>
bool atomic_is_lock_free(const atomic<BaseType>*) noexcept;
template<typename BaseType>
void atomic_init(volatile atomic<BaseType>*, void*) noexcept;
template<typename BaseType>
void atomic_init(atomic<BaseType>*, void*) noexcept;
template<typename BaseType>
BaseType atomic_exchange(volatile atomic<BaseType>*, memory_order)
  noexcept;
template<typename BaseType>
BaseType atomic_exchange(atomic<BaseType>*, memory_order) noexcept;
template<typename BaseType>
BaseType atomic_exchange_explicit(
  volatile atomic<BaseType>*, memory_order) noexcept;
template<typename BaseType>
BaseType atomic_exchange_explicit(
  atomic<BaseType>*, memory_order) noexcept;
template<typename BaseType>
void atomic_store(volatile atomic<BaseType>*, BaseType) noexcept;
template<typename BaseType>
void atomic_store(atomic<BaseType>*, BaseType) noexcept;
template<typename BaseType>
void atomic_store_explicit(
  volatile atomic<BaseType>*, BaseType, memory_order) noexcept;
template<typename BaseType>
void atomic_store_explicit(
  atomic<BaseType>*, BaseType, memory_order) noexcept;
template<typename BaseType>
BaseType atomic_load(volatile const atomic<BaseType>*) noexcept;
template<typename BaseType>
BaseType atomic_load(const atomic<BaseType>*) noexcept;
template<typename BaseType>
BaseType atomic_load_explicit(
  volatile const atomic<BaseType>*, memory_order) noexcept;
template<typename BaseType>
BaseType atomic_load_explicit(
  const atomic<BaseType>*, memory_order) noexcept;
template<typename BaseType>
bool atomic_compare_exchange_strong(
  volatile atomic<BaseType>*,BaseType * old_value,
  BaseType new_value) noexcept;
template<typename BaseType>
bool atomic_compare_exchange_strong(
  atomic<BaseType>*,BaseType * old_value,
  BaseType new_value) noexcept;
template<typename BaseType>
bool atomic_compare_exchange_strong_explicit(
  volatile atomic<BaseType>*,BaseType * old_value,
  BaseType new_value, memory_order success_order,
  memory_order failure_order) noexcept;
template<typename BaseType>
bool atomic_compare_exchange_strong_explicit(
  atomic<BaseType>*,BaseType * old_value,
  BaseType new_value, memory_order success_order,
  memory_order failure_order) noexcept;
template<typename BaseType>
bool atomic_compare_exchange_weak(
  volatile atomic<BaseType>*,BaseType * old_value,BaseType new_value)
  noexcept;
template<typename BaseType>
bool atomic_compare_exchange_weak(
  atomic<BaseType>*,BaseType * old_value,BaseType new_value) noexcept;
template<typename BaseType>
bool atomic_compare_exchange_weak_explicit(
  volatile atomic<BaseType>*,BaseType * old_value,
  BaseType new_value, memory_order success_order,
  memory_order failure_order) noexcept;
template<typename BaseType>
bool atomic_compare_exchange_weak_explicit(
  atomic<BaseType>*,BaseType * old_value,
  BaseType new_value, memory_order success_order,
  memory_order failure_order) noexcept;
```

**NOTE**:虽然非成员函数通过模板的方式指定，不过他们只作为从在函数提供，并且对于这些函数，不能显示的指定模板的参数。

####std::atomic 构造函数

使用默认初始值，构造一个`std::atomic`实例。

**声明**
```c++
atomic() noexcept;
```

**效果**<br>
使用默认初始值，构造一个新`std::atomic`实例。因对象是静态存储的，所以初始化过程也是静态的。

**NOTE**:当`std::atomic`实例以非静态方式初始化的，那么其值就是不可估计的。

**抛出**<br>
无

####std::atomic_init 非成员函数

`std::atomic<BaseType>`实例提供的值，可非原子的进行存储。

**声明**
```c++
template<typename BaseType>
void atomic_init(atomic<BaseType> volatile* p, BaseType v) noexcept;
template<typename BaseType>
void atomic_init(atomic<BaseType>* p, BaseType v) noexcept;
```

**效果**<br>
将值v以非原子存储的方式，存储在*p中。调用`atomic<BaseType>`实例中的atomic_init()，这里需要实例不是默认构造出来的，或者在构造出来的时候被执行了某些操作，否则将会引发未定义行为。

**NOTE**:因为存储是非原子的，对对象指针p任意的并发访问(即使是原子操作)都会引发数据竞争。

**抛出**<br>
无

####std::atomic 转换构造函数

使用提供的BaseType值去构造一个`std::atomic`实例。

**声明**
```c++
constexpr atomic(BaseType b) noexcept;
```

**效果**<br>
通过b值构造一个新的`std::atomic`对象。因对象是静态存储的，所以初始化过程也是静态的。

**抛出**<br>
无

####std::atomic 转换赋值操作

在*this存储一个新值。

**声明**
```c++
BaseType operator=(BaseType b) volatile noexcept;
BaseType operator=(BaseType b) noexcept;
```

**效果**
```c++
return this->store(b);
```

####std::atomic::is_lock_free 成员函数

确定对于*this是否是无锁操作。

**声明**
```c++
bool is_lock_free() const volatile noexcept;
bool is_lock_free() const noexcept;
```

**返回**<br>
当操作是无锁操作，那么就返回true，否则返回false。

**抛出**<br>
无

####std::atomic_is_lock_free 非成员函数

确定对于*this是否是无锁操作。

**声明**
```c++
template<typename BaseType>
bool atomic_is_lock_free(volatile const atomic<BaseType>* p) noexcept;
template<typename BaseType>
bool atomic_is_lock_free(const atomic<BaseType>* p) noexcept;
```

**效果**
```c++
return p->is_lock_free();
```

####std::atomic::load 成员函数

原子的加载`std::atomic`实例当前的值

**声明**
```c++
BaseType load(memory_order order = memory_order_seq_cst)
    const volatile noexcept;
BaseType load(memory_order order = memory_order_seq_cst) const noexcept;
```

**先决条件**<br>
支持`std::memory_order_relaxed`、`std::memory_order_acquire`、`std::memory_order_consume`或`std::memory_order_seq_cst`内存序。

**效果**<br>
原子的加载已存储到*this上的值。

**返回**<br>
返回存储在*this上的值。

**抛出**<br>
无

**NOTE**:是对于*this内存地址原子加载的操作。

####std::atomic_load 非成员函数

原子的加载`std::atomic`实例当前的值。

**声明**
```c++
template<typename BaseType>
BaseType atomic_load(volatile const atomic<BaseType>* p) noexcept;
template<typename BaseType>
BaseType atomic_load(const atomic<BaseType>* p) noexcept;
```

**效果**
```c++
return p->load();
```

####std::atomic_load_explicit 非成员函数

原子的加载`std::atomic`实例当前的值。

**声明**
```c++
template<typename BaseType>
BaseType atomic_load_explicit(
    volatile const atomic<BaseType>* p, memory_order order) noexcept;
template<typename BaseType>
BaseType atomic_load_explicit(
    const atomic<BaseType>* p, memory_order order) noexcept;
```

**效果**
```c++
return p->load(order);
```

####std::atomic::operator BastType转换操作

加载存储在*this中的值。

**声明**
```c++
operator BaseType() const volatile noexcept;
operator BaseType() const noexcept;
```

**效果**
```c++
return this->load();
```

####std::atomic::store 成员函数

以原子操作的方式存储一个新值到`atomic<BaseType>`实例中。

**声明**
```c++
void store(BaseType new_value,memory_order order = memory_order_seq_cst)
    volatile noexcept;
void store(BaseType new_value,memory_order order = memory_order_seq_cst)
    noexcept;
```

**先决条件**<br>
支持`std::memory_order_relaxed`、`std::memory_order_release`或`std::memory_order_seq_cst`内存序。

**效果**<br>
将new_value原子的存储到*this中。

**抛出**<br>
无

**NOTE**:是对于*this内存地址原子加载的操作。

####std::atomic_store 非成员函数

以原子操作的方式存储一个新值到`atomic&lt;BaseType&gt;`实例中。

**声明**
```c++
template<typename BaseType>
void atomic_store(volatile atomic<BaseType>* p, BaseType new_value)
    noexcept;
template<typename BaseType>
void atomic_store(atomic<BaseType>* p, BaseType new_value) noexcept;
```

**效果**
```c++
p->store(new_value);
```

####std::atomic_explicit 非成员函数

以原子操作的方式存储一个新值到`atomic&lt;BaseType&gt;`实例中。

**声明**
```c++
template<typename BaseType>
void atomic_store_explicit(
    volatile atomic<BaseType>* p, BaseType new_value, memory_order order)
    noexcept;
template<typename BaseType>
void atomic_store_explicit(
    atomic<BaseType>* p, BaseType new_value, memory_order order) noexcept;
```

**效果**
```c++
p->store(new_value,order);
```

####std::atomic::exchange 成员函数

原子的存储一个新值，并读取旧值。

**声明**
```c++
BaseType exchange(
    BaseType new_value,
    memory_order order = memory_order_seq_cst)
    volatile noexcept;
```

**效果**<br>
原子的将new_value存储在*this中，并且取出*this中已经存储的值。

**返回**<br>
返回*this之前的值。

**抛出**<br>
无

**NOTE**:这是对*this内存地址的原子“读-改-写”操作。

####std::atomic_exchange 非成员函数

原子的存储一个新值到`atomic<BaseType>`实例中，并且读取旧值。

**声明**
```c++
template<typename BaseType>
BaseType atomic_exchange(volatile atomic<BaseType>* p, BaseType new_value)
    noexcept;
template<typename BaseType>
BaseType atomic_exchange(atomic<BaseType>* p, BaseType new_value) noexcept;
```

**效果**
```c++
return p->exchange(new_value);
```

####std::atomic_exchange_explicit 非成员函数

原子的存储一个新值到`atomic<BaseType>`实例中，并且读取旧值。

**声明**
```c++
template<typename BaseType>
BaseType atomic_exchange_explicit(
    volatile atomic<BaseType>* p, BaseType new_value, memory_order order)
    noexcept;
template<typename BaseType>
BaseType atomic_exchange_explicit(
    atomic<BaseType>* p, BaseType new_value, memory_order order) noexcept;
```

**效果**
```c++
return p->exchange(new_value,order);
```

####std::atomic::compare_exchange_strong 成员函数

当期望值和新值一样时，将新值存储到实例中。如果不相等，那么就实用新值更新期望值。

**声明**
```c++
bool compare_exchange_strong(
    BaseType& expected,BaseType new_value,
    memory_order order = std::memory_order_seq_cst) volatile noexcept;
bool compare_exchange_strong(
    BaseType& expected,BaseType new_value,
    memory_order order = std::memory_order_seq_cst) noexcept;
bool compare_exchange_strong(
    BaseType& expected,BaseType new_value,
    memory_order success_order,memory_order failure_order)
    volatile noexcept;
bool compare_exchange_strong(
    BaseType& expected,BaseType new_value,
    memory_order success_order,memory_order failure_order) noexcept;
```

**先决条件**<br>
failure_order不能是`std::memory_order_release`或`std::memory_order_acq_rel`内存序。

**效果**<br>
将存储在*this中的expected值与new_value值进行逐位对比，当相等时间new_value存储在*this中；否则，更新expected的值。

**返回**<br>
当new_value的值与*this中已经存在的值相同，就返回true；否则，返回false。

**抛出**<br>
无

**NOTE**:在success_order==order和failure_order==order的情况下，三个参数的重载函数与四个参数的重载函数等价。除非，order是`std::memory_order_acq_rel`时，failure_order是`std::memory_order_acquire`，且当order是`std::memory_order_release`时，failure_order是`std::memory_order_relaxed`。

**NOTE**:当返回true和success_order内存序时，是对*this内存地址的原子“读-改-写”操作；反之，这是对*this内存地址的原子加载操作(failure_order)。

####std::atomic_compare_exchange_strong 非成员函数

当期望值和新值一样时，将新值存储到实例中。如果不相等，那么就实用新值更新期望值。

**声明**
```c++
template<typename BaseType>
bool atomic_compare_exchange_strong(
    volatile atomic<BaseType>* p,BaseType * old_value,BaseType new_value)
    noexcept;
template<typename BaseType>
bool atomic_compare_exchange_strong(  
    atomic<BaseType>* p,BaseType * old_value,BaseType new_value) noexcept;
```

**效果**
```c++
return p->compare_exchange_strong(*old_value,new_value);
```

####std::atomic_compare_exchange_strong_explicit 非成员函数

当期望值和新值一样时，将新值存储到实例中。如果不相等，那么就实用新值更新期望值。

**声明**
```c++
template<typename BaseType>
bool atomic_compare_exchange_strong_explicit(
    volatile atomic<BaseType>* p,BaseType * old_value,
    BaseType new_value, memory_order success_order,
    memory_order failure_order) noexcept;
template<typename BaseType>
bool atomic_compare_exchange_strong_explicit(
    atomic<BaseType>* p,BaseType * old_value,
    BaseType new_value, memory_order success_order,
    memory_order failure_order) noexcept;
```

**效果**<br>
```c++
return p->compare_exchange_strong(
    *old_value,new_value,success_order,failure_order) noexcept;
```

####std::atomic::compare_exchange_weak 成员函数

原子的比较新值和期望值，如果相等，那么存储新值并且进行原子化更新。当两值不相等，或更新未进行，那期望值会更新为新值。

**声明**
```c++
bool compare_exchange_weak(
    BaseType& expected,BaseType new_value,
    memory_order order = std::memory_order_seq_cst) volatile noexcept;
bool compare_exchange_weak(
    BaseType& expected,BaseType new_value,
    memory_order order = std::memory_order_seq_cst) noexcept;
bool compare_exchange_weak(
    BaseType& expected,BaseType new_value,
    memory_order success_order,memory_order failure_order)
    volatile noexcept;
bool compare_exchange_weak(
    BaseType& expected,BaseType new_value,
    memory_order success_order,memory_order failure_order) noexcept;
```

**先决条件**<br>
failure_order不能是`std::memory_order_release`或`std::memory_order_acq_rel`内存序。

**效果**<br>
将存储在*this中的expected值与new_value值进行逐位对比，当相等时间new_value存储在*this中；否则，更新expected的值。

**返回**<br>
当new_value的值与*this中已经存在的值相同，就返回true；否则，返回false。

**抛出**<br>
无

**NOTE**:在success_order==order和failure_order==order的情况下，三个参数的重载函数与四个参数的重载函数等价。除非，order是`std::memory_order_acq_rel`时，failure_order是`std::memory_order_acquire`，且当order是`std::memory_order_release`时，failure_order是`std::memory_order_relaxed`。

**NOTE**:当返回true和success_order内存序时，是对*this内存地址的原子“读-改-写”操作；反之，这是对*this内存地址的原子加载操作(failure_order)。

####std::atomic_compare_exchange_weak 非成员函数

原子的比较新值和期望值，如果相等，那么存储新值并且进行原子化更新。当两值不相等，或更新未进行，那期望值会更新为新值。

**声明**
```c++
template<typename BaseType>
bool atomic_compare_exchange_weak(
    volatile atomic<BaseType>* p,BaseType * old_value,BaseType new_value)
    noexcept;
template<typename BaseType>
bool atomic_compare_exchange_weak(
    atomic<BaseType>* p,BaseType * old_value,BaseType new_value) noexcept;
```

**效果**
```c++
return p->compare_exchange_weak(*old_value,new_value);
```

####std::atomic_compare_exchange_weak_explicit 非成员函数

原子的比较新值和期望值，如果相等，那么存储新值并且进行原子化更新。当两值不相等，或更新未进行，那期望值会更新为新值。

**声明**
```c++
template<typename BaseType>
bool atomic_compare_exchange_weak_explicit(
    volatile atomic<BaseType>* p,BaseType * old_value,
    BaseType new_value, memory_order success_order,
    memory_order failure_order) noexcept;
template<typename BaseType>
bool atomic_compare_exchange_weak_explicit(
    atomic<BaseType>* p,BaseType * old_value,
    BaseType new_value, memory_order success_order,
    memory_order failure_order) noexcept;
```

**效果**
```c++
return p->compare_exchange_weak(
   *old_value,new_value,success_order,failure_order);
```

###D.3.9 std::atomic模板类型的特化

`std::atomic`类模板的特化类型有整型和指针类型。对于整型来说，特化模板提供原子加减，以及位域操作(主模板未提供)。对于指针类型来说，特化模板提供原子指针的运算(主模板未提供)。

特化模板提供如下整型：
```c++
std::atomic<bool>
std::atomic<char>
std::atomic<signed char>
std::atomic<unsigned char>
std::atomic<short>
std::atomic<unsigned short>
std::atomic<int>
std::atomic<unsigned>
std::atomic<long>
std::atomic<unsigned long>
std::atomic<long long>
std::atomic<unsigned long long>
std::atomic<wchar_t>
std::atomic<char16_t>
std::atomic<char32_t&gt;
```

`std::atomic<T*>`原子指针，可以使用以上的类型作为T。

###D.3.10 特化std::atomic&lt;integral-type&gt;

`std::atomic&lt;integral-type&gt;`是为每一个基础整型提供的`std::atomic`类模板，其中提供了一套完整的整型操作。

下面的特化模板也适用于`std::atomic<>`类模板：

```c++
std::atomic<char>
std::atomic<signed char>
std::atomic<unsigned char>
std::atomic<short>
std::atomic<unsigned short>
std::atomic<int>
std::atomic<unsigned>
std::atomic<long>
std::atomic<unsigned long>
std::atomic<long long>
std::atomic<unsigned long long>
std::atomic<wchar_t>
std::atomic<char16_t>
std::atomic<char32_t>
```

因为原子操作只能执行其中一个，所以特化模板的实例不可`CopyConstructible`(拷贝构造)和`CopyAssignable`(拷贝赋值)。

**类型定义**
```c++
template<>
struct atomic<integral-type>
{
  atomic() noexcept = default;
  constexpr atomic(integral-type) noexcept;
  bool operator=(integral-type) volatile noexcept;

  atomic(const atomic&) = delete;
  atomic& operator=(const atomic&) = delete;
  atomic& operator=(const atomic&) volatile = delete;

  bool is_lock_free() const volatile noexcept;
  bool is_lock_free() const noexcept;

  void store(integral-type,memory_order = memory_order_seq_cst)
      volatile noexcept;
  void store(integral-type,memory_order = memory_order_seq_cst) noexcept;
  integral-type load(memory_order = memory_order_seq_cst)
      const volatile noexcept;
  integral-type load(memory_order = memory_order_seq_cst) const noexcept;
  integral-type exchange(
      integral-type,memory_order = memory_order_seq_cst)
      volatile noexcept;
 integral-type exchange(
      integral-type,memory_order = memory_order_seq_cst) noexcept;

  bool compare_exchange_strong(
      integral-type & old_value,integral-type new_value,
      memory_order order = memory_order_seq_cst) volatile noexcept;
  bool compare_exchange_strong(
      integral-type & old_value,integral-type new_value,
      memory_order order = memory_order_seq_cst) noexcept;
  bool compare_exchange_strong(
      integral-type & old_value,integral-type new_value,
      memory_order success_order,memory_order failure_order)
      volatile noexcept;
  bool compare_exchange_strong(
      integral-type & old_value,integral-type new_value,
      memory_order success_order,memory_order failure_order) noexcept;
  bool compare_exchange_weak(
      integral-type & old_value,integral-type new_value,
      memory_order order = memory_order_seq_cst) volatile noexcept;
  bool compare_exchange_weak(
      integral-type & old_value,integral-type new_value,
      memory_order order = memory_order_seq_cst) noexcept;
  bool compare_exchange_weak(
      integral-type & old_value,integral-type new_value,
      memory_order success_order,memory_order failure_order)
      volatile noexcept;
  bool compare_exchange_weak(
      integral-type & old_value,integral-type new_value,
      memory_order success_order,memory_order failure_order) noexcept;

  operator integral-type() const volatile noexcept;
  operator integral-type() const noexcept;

  integral-type fetch_add(
      integral-type,memory_order = memory_order_seq_cst)
      volatile noexcept;
  integral-type fetch_add(
      integral-type,memory_order = memory_order_seq_cst) noexcept;
  integral-type fetch_sub(
      integral-type,memory_order = memory_order_seq_cst)
      volatile noexcept;
  integral-type fetch_sub(
      integral-type,memory_order = memory_order_seq_cst) noexcept;
  integral-type fetch_and(
      integral-type,memory_order = memory_order_seq_cst)
      volatile noexcept;
  integral-type fetch_and(
      integral-type,memory_order = memory_order_seq_cst) noexcept;
  integral-type fetch_or(
      integral-type,memory_order = memory_order_seq_cst)
      volatile noexcept;
  integral-type fetch_or(
      integral-type,memory_order = memory_order_seq_cst) noexcept;
  integral-type fetch_xor(
      integral-type,memory_order = memory_order_seq_cst)
      volatile noexcept;
  integral-type fetch_xor(
      integral-type,memory_order = memory_order_seq_cst) noexcept;

  integral-type operator++() volatile noexcept;
  integral-type operator++() noexcept;
  integral-type operator++(int) volatile noexcept;
  integral-type operator++(int) noexcept;
  integral-type operator--() volatile noexcept;
  integral-type operator--() noexcept;
  integral-type operator--(int) volatile noexcept;
  integral-type operator--(int) noexcept;
  integral-type operator+=(integral-type) volatile noexcept;
  integral-type operator+=(integral-type) noexcept;
  integral-type operator-=(integral-type) volatile noexcept;
  integral-type operator-=(integral-type) noexcept;
  integral-type operator&=(integral-type) volatile noexcept;
  integral-type operator&=(integral-type) noexcept;
  integral-type operator|=(integral-type) volatile noexcept;
  integral-type operator|=(integral-type) noexcept;
  integral-type operator^=(integral-type) volatile noexcept;
  integral-type operator^=(integral-type) noexcept;
};

bool atomic_is_lock_free(volatile const atomic<integral-type>*) noexcept;
bool atomic_is_lock_free(const atomic<integral-type>*) noexcept;
void atomic_init(volatile atomic<integral-type>*,integral-type) noexcept;
void atomic_init(atomic<integral-type>*,integral-type) noexcept;
integral-type atomic_exchange(
    volatile atomic<integral-type>*,integral-type) noexcept;
integral-type atomic_exchange(
    atomic<integral-type>*,integral-type) noexcept;
integral-type atomic_exchange_explicit(
    volatile atomic<integral-type>*,integral-type, memory_order) noexcept;
integral-type atomic_exchange_explicit(
    atomic<integral-type>*,integral-type, memory_order) noexcept;
void atomic_store(volatile atomic<integral-type>*,integral-type) noexcept;
void atomic_store(atomic<integral-type>*,integral-type) noexcept;
void atomic_store_explicit(
    volatile atomic<integral-type>*,integral-type, memory_order) noexcept;
void atomic_store_explicit(
    atomic<integral-type>*,integral-type, memory_order) noexcept;
integral-type atomic_load(volatile const atomic<integral-type>*) noexcept;
integral-type atomic_load(const atomic<integral-type>*) noexcept;
integral-type atomic_load_explicit(
    volatile const atomic<integral-type>*,memory_order) noexcept;
integral-type atomic_load_explicit(
    const atomic<integral-type>*,memory_order) noexcept;
bool atomic_compare_exchange_strong(
    volatile atomic<integral-type>*,
    integral-type * old_value,integral-type new_value) noexcept;
bool atomic_compare_exchange_strong(
    atomic<integral-type>*,
    integral-type * old_value,integral-type new_value) noexcept;
bool atomic_compare_exchange_strong_explicit(
    volatile atomic<integral-type>*,
    integral-type * old_value,integral-type new_value,
    memory_order success_order,memory_order failure_order) noexcept;
bool atomic_compare_exchange_strong_explicit(
    atomic<integral-type>*,
    integral-type * old_value,integral-type new_value,
    memory_order success_order,memory_order failure_order) noexcept;
bool atomic_compare_exchange_weak(
    volatile atomic<integral-type>*,
    integral-type * old_value,integral-type new_value) noexcept;
bool atomic_compare_exchange_weak(
    atomic<integral-type>*,
    integral-type * old_value,integral-type new_value) noexcept;
bool atomic_compare_exchange_weak_explicit(
    volatile atomic<integral-type>*,
    integral-type * old_value,integral-type new_value,
    memory_order success_order,memory_order failure_order) noexcept;
bool atomic_compare_exchange_weak_explicit(
    atomic<integral-type>*,
    integral-type * old_value,integral-type new_value,
    memory_order success_order,memory_order failure_order) noexcept;

integral-type atomic_fetch_add(
    volatile atomic<integral-type>*,integral-type) noexcept;
integral-type atomic_fetch_add(
    atomic<integral-type>*,integral-type) noexcept;
integral-type atomic_fetch_add_explicit(
    volatile atomic<integral-type>*,integral-type, memory_order) noexcept;
integral-type atomic_fetch_add_explicit(
    atomic<integral-type>*,integral-type, memory_order) noexcept;
integral-type atomic_fetch_sub(
    volatile atomic<integral-type>*,integral-type) noexcept;
integral-type atomic_fetch_sub(
    atomic<integral-type>*,integral-type) noexcept;
integral-type atomic_fetch_sub_explicit(
    volatile atomic<integral-type>*,integral-type, memory_order) noexcept;
integral-type atomic_fetch_sub_explicit(
    atomic<integral-type>*,integral-type, memory_order) noexcept;
integral-type atomic_fetch_and(
    volatile atomic<integral-type>*,integral-type) noexcept;
integral-type atomic_fetch_and(
    atomic<integral-type>*,integral-type) noexcept;
integral-type atomic_fetch_and_explicit(
    volatile atomic<integral-type>*,integral-type, memory_order) noexcept;
integral-type atomic_fetch_and_explicit(
    atomic<integral-type>*,integral-type, memory_order) noexcept;
integral-type atomic_fetch_or(
    volatile atomic<integral-type>*,integral-type) noexcept;
integral-type atomic_fetch_or(
    atomic<integral-type>*,integral-type) noexcept;
integral-type atomic_fetch_or_explicit(
    volatile atomic<integral-type>*,integral-type, memory_order) noexcept;
integral-type atomic_fetch_or_explicit(
    atomic<integral-type>*,integral-type, memory_order) noexcept;
integral-type atomic_fetch_xor(
    volatile atomic<integral-type>*,integral-type) noexcept;
integral-type atomic_fetch_xor(
    atomic<integral-type>*,integral-type) noexcept;
integral-type atomic_fetch_xor_explicit(
    volatile atomic<integral-type>*,integral-type, memory_order) noexcept;
integral-type atomic_fetch_xor_explicit(
    atomic<integral-type>*,integral-type, memory_order) noexcept;
```

这些操作在主模板中也有提供(见D.3.8)。

####std::atomic&lt;integral-type&gt;::fetch_add 成员函数

原子的加载一个值，然后使用与提供i相加的结果，替换掉原值。

**声明**
```c++
integral-type fetch_add(
    integral-type i,memory_order order = memory_order_seq_cst)
    volatile noexcept;
integral-type fetch_add(
    integral-type i,memory_order order = memory_order_seq_cst) noexcept;
```

**效果**<br>
原子的查询*this中的值，将old-value+i的和存回*this。

**返回**<br>
返回*this之前存储的值。

**抛出**<br>
无

**NOTE**:对于*this的内存地址来说，这是一个“读-改-写”操作。

####std::atomic_fetch_add 非成员函数

从`atomic<integral-type>`实例中原子的读取一个值，并且将其与给定i值相加，替换原值。

**声明**
```c++
integral-type atomic_fetch_add(
    volatile atomic<integral-type>* p, integral-type i) noexcept;
integral-type atomic_fetch_add(
    atomic<integral-type>* p, integral-type i) noexcept;
```

**效果**
```c++
return p->fetch_add(i);
```

####std::atomic_fetch_add_explicit 非成员函数

从`atomic<integral-type>`实例中原子的读取一个值，并且将其与给定i值相加，替换原值。

**声明**
```c++
integral-type atomic_fetch_add_explicit(
    volatile atomic<integral-type>* p, integral-type i,
    memory_order order) noexcept;
integral-type atomic_fetch_add_explicit(
    atomic<integral-type>* p, integral-type i, memory_order order)
    noexcept;
```

**效果**
```c++
return p->fetch_add(i,order);
```

####std::atomic&lt;integral-type&gt;::fetch_sub 成员函数

原子的加载一个值，然后使用与提供i相减的结果，替换掉原值。

**声明**
```c++
integral-type fetch_sub(
    integral-type i,memory_order order = memory_order_seq_cst)
    volatile noexcept;
integral-type fetch_sub(
    integral-type i,memory_order order = memory_order_seq_cst) noexcept;
```

**效果**<br>
原子的查询*this中的值，将old-value-i的和存回*this。

**返回**<br>
返回*this之前存储的值。

**抛出**<br>
无

**NOTE**:对于*this的内存地址来说，这是一个“读-改-写”操作。

####std::atomic_fetch_sub 非成员函数

从`atomic<integral-type>`实例中原子的读取一个值，并且将其与给定i值相减，替换原值。

**声明**
```c++
integral-type atomic_fetch_sub(
    volatile atomic<integral-type>* p, integral-type i) noexcept;
integral-type atomic_fetch_sub(
    atomic<integral-type>* p, integral-type i) noexcept;
```

**效果**
```c++
return p->fetch_sub(i);
```

####std::atomic_fetch_sub_explicit 非成员函数

从`atomic<integral-type>`实例中原子的读取一个值，并且将其与给定i值相减，替换原值。

**声明**
```c++
integral-type atomic_fetch_sub_explicit(
    volatile atomic<integral-type>* p, integral-type i,
    memory_order order) noexcept;
integral-type atomic_fetch_sub_explicit(
    atomic<integral-type>* p, integral-type i, memory_order order)
    noexcept;
```

**效果**
```c++
return p->fetch_sub(i,order);
```

####std::atomic&lt;integral-type&gt;::fetch_and 成员函数

从`atomic<integral-type>`实例中原子的读取一个值，并且将其与给定i值进行位与操作后，替换原值。

**声明**
```c++
integral-type fetch_and(
    integral-type i,memory_order order = memory_order_seq_cst)
    volatile noexcept;
integral-type fetch_and(
    integral-type i,memory_order order = memory_order_seq_cst) noexcept;
```

**效果**<br>
原子的查询*this中的值，将old-value&i的和存回*this。

**返回**<br>
返回*this之前存储的值。

**抛出**<br>
无

**NOTE**:对于*this的内存地址来说，这是一个“读-改-写”操作。

####std::atomic_fetch_and 非成员函数

从`atomic<integral-type>`实例中原子的读取一个值，并且将其与给定i值进行位与操作后，替换原值。

**声明**
```c++
integral-type atomic_fetch_and(
    volatile atomic<integral-type>* p, integral-type i) noexcept;
integral-type atomic_fetch_and(
    atomic<integral-type>* p, integral-type i) noexcept;
```

**效果**
```c++
return p->fetch_and(i);
```

####std::atomic_fetch_and_explicit 非成员函数

从`atomic<integral-type>`实例中原子的读取一个值，并且将其与给定i值进行位与操作后，替换原值。

**声明**
```c++
integral-type atomic_fetch_and_explicit(
    volatile atomic<integral-type>* p, integral-type i,
    memory_order order) noexcept;
integral-type atomic_fetch_and_explicit(
    atomic<integral-type>* p, integral-type i, memory_order order)
    noexcept;
```

**效果**
```c++
return p->fetch_and(i,order);
```

####std::atomic&lt;integral-type&gt;::fetch_or 成员函数

从`atomic<integral-type>`实例中原子的读取一个值，并且将其与给定i值进行位或操作后，替换原值。

**声明**
```c++
integral-type fetch_or(
    integral-type i,memory_order order = memory_order_seq_cst)
    volatile noexcept;
integral-type fetch_or(
    integral-type i,memory_order order = memory_order_seq_cst) noexcept;
```

**效果**<br>
原子的查询*this中的值，将old-value|i的和存回*this。

**返回**<br>
返回*this之前存储的值。

**抛出**<br>
无

**NOTE**:对于*this的内存地址来说，这是一个“读-改-写”操作。

####std::atomic_fetch_or 非成员函数

从`atomic<integral-type>`实例中原子的读取一个值，并且将其与给定i值进行位或操作后，替换原值。

**声明**
```c++
integral-type atomic_fetch_or(
    volatile atomic<integral-type>* p, integral-type i) noexcept;
integral-type atomic_fetch_or(
    atomic<integral-type>* p, integral-type i) noexcept;
```

**效果**
```c++
return p->fetch_or(i);
```

####std::atomic_fetch_or_explicit 非成员函数

从`atomic<integral-type>`实例中原子的读取一个值，并且将其与给定i值进行位或操作后，替换原值。

**声明**
```c++
integral-type atomic_fetch_or_explicit(
    volatile atomic<integral-type>* p, integral-type i,
    memory_order order) noexcept;
integral-type atomic_fetch_or_explicit(
    atomic<integral-type>* p, integral-type i, memory_order order)
    noexcept;
```

**效果**
```c++
return p->fetch_or(i,order);
```

####std::atomic&lt;integral-type&gt;::fetch_xor 成员函数

从`atomic<integral-type>`实例中原子的读取一个值，并且将其与给定i值进行位亦或操作后，替换原值。

**声明**
```c++
integral-type fetch_xor(
    integral-type i,memory_order order = memory_order_seq_cst)
    volatile noexcept;
integral-type fetch_xor(
    integral-type i,memory_order order = memory_order_seq_cst) noexcept;
```

**效果**<br>
原子的查询*this中的值，将old-value^i的和存回*this。

**返回**<br>
返回*this之前存储的值。

**抛出**<br>
无

**NOTE**:对于*this的内存地址来说，这是一个“读-改-写”操作。

####std::atomic_fetch_xor 非成员函数

从`atomic<integral-type>`实例中原子的读取一个值，并且将其与给定i值进行位异或操作后，替换原值。

**声明**
```c++
integral-type atomic_fetch_xor_explicit(
    volatile atomic<integral-type>* p, integral-type i,
    memory_order order) noexcept;
integral-type atomic_fetch_xor_explicit(
    atomic<integral-type>* p, integral-type i, memory_order order)
    noexcept;
```

**效果**
```c++
return p->fetch_xor(i,order);
```

####std::atomic_fetch_xor_explicit 非成员函数

从`atomic<integral-type>`实例中原子的读取一个值，并且将其与给定i值进行位异或操作后，替换原值。

**声明**
```c++
integral-type atomic_fetch_xor_explicit(
    volatile atomic<integral-type>* p, integral-type i,
    memory_order order) noexcept;
integral-type atomic_fetch_xor_explicit(
    atomic<integral-type>* p, integral-type i, memory_order order)
    noexcept;
```

**效果**
```c++
return p->fetch_xor(i,order);
```

####std::atomic&lt;integral-type&gt;::operator++ 前置递增操作

原子的将*this中存储的值加1，并返回新值。

**声明**
```c++
integral-type operator++() volatile noexcept;
integral-type operator++() noexcept;
```

**效果**
```c++
return this->fetch_add(1) + 1;
```

####std::atomic&lt;integral-type&gt;::operator++ 后置递增操作

原子的将*this中存储的值加1，并返回旧值。

**声明**
```c++
integral-type operator++() volatile noexcept;
integral-type operator++() noexcept;
```

**效果**
```c++
return this->fetch_add(1);
```

####std::atomic&lt;integral-type&gt;::operator-- 前置递减操作

原子的将*this中存储的值减1，并返回新值。

**声明**
```c++
integral-type operator--() volatile noexcept;
integral-type operator--() noexcept;
```

**效果**
```c++
return this->fetch_add(1) - 1;
```

####std::atomic&lt;integral-type&gt;::operator-- 后置递减操作

原子的将*this中存储的值减1，并返回旧值。

**声明**
```c++
integral-type operator--() volatile noexcept;
integral-type operator--() noexcept;
```

**效果**
```c++
return this->fetch_add(1);
```

####std::atomic&lt;integral-type&gt;::operator+= 复合赋值操作

原子的将给定值与*this中的值相加，并返回新值。

**声明**
```c++
integral-type operator+=(integral-type i) volatile noexcept;
integral-type operator+=(integral-type i) noexcept;
```

**效果**
```c++
return this->fetch_add(i) + i;
```

####std::atomic&lt;integral-type&gt;::operator-= 复合赋值操作

原子的将给定值与*this中的值相减，并返回新值。

**声明**
```c++
integral-type operator-=(integral-type i) volatile noexcept;
integral-type operator-=(integral-type i) noexcept;
```

**效果**
```c++
return this->fetch_sub(i,std::memory_order_seq_cst) – i;
```

####std::atomic&lt;integral-type&gt;::operator&= 复合赋值操作

原子的将给定值与*this中的值相与，并返回新值。

**声明**
```c++
integral-type operator&=(integral-type i) volatile noexcept;
integral-type operator&=(integral-type i) noexcept;
```

**效果**
```c++
return this->fetch_and(i) & i;
```

####std::atomic&lt;integral-type&gt;::operator|= 复合赋值操作

原子的将给定值与*this中的值相或，并返回新值。

**声明**
```c++
integral-type operator|=(integral-type i) volatile noexcept;
integral-type operator|=(integral-type i) noexcept;
```

**效果**
```c++
return this->fetch_or(i,std::memory_order_seq_cst) | i;
```

####std::atomic&lt;integral-type&gt;::operator^= 复合赋值操作

原子的将给定值与*this中的值相亦或，并返回新值。

**声明**
```c++
integral-type operator^=(integral-type i) volatile noexcept;
integral-type operator^=(integral-type i) noexcept;
```

**效果**
```c++
return this->fetch_xor(i,std::memory_order_seq_cst) ^ i;
```

####std::atomic&lt;T*&gt; 局部特化

`std::atomic<T*>`为`std::atomic`特化了指针类型原子变量，并提供了一系列相关操作。

`std::atomic<T*>`是CopyConstructible(拷贝构造)和CopyAssignable(拷贝赋值)的，因为操作是原子的，在同一时间只能执行一个操作。

**类型定义**
```c++
template<typename T>
struct atomic<T*>
{
  atomic() noexcept = default;
  constexpr atomic(T*) noexcept;
  bool operator=(T*) volatile;
  bool operator=(T*);

  atomic(const atomic&) = delete;
  atomic& operator=(const atomic&) = delete;
  atomic& operator=(const atomic&) volatile = delete;

  bool is_lock_free() const volatile noexcept;
  bool is_lock_free() const noexcept;
  void store(T*,memory_order = memory_order_seq_cst) volatile noexcept;
  void store(T*,memory_order = memory_order_seq_cst) noexcept;
  T* load(memory_order = memory_order_seq_cst) const volatile noexcept;
  T* load(memory_order = memory_order_seq_cst) const noexcept;
  T* exchange(T*,memory_order = memory_order_seq_cst) volatile noexcept;
  T* exchange(T*,memory_order = memory_order_seq_cst) noexcept;

  bool compare_exchange_strong(
      T* & old_value, T* new_value,
      memory_order order = memory_order_seq_cst) volatile noexcept;
  bool compare_exchange_strong(
      T* & old_value, T* new_value,
      memory_order order = memory_order_seq_cst) noexcept;
  bool compare_exchange_strong(
      T* & old_value, T* new_value,
      memory_order success_order,memory_order failure_order)  
      volatile noexcept;
  bool compare_exchange_strong(
      T* & old_value, T* new_value,
      memory_order success_order,memory_order failure_order) noexcept;
  bool compare_exchange_weak(
      T* & old_value, T* new_value,
      memory_order order = memory_order_seq_cst) volatile noexcept;
  bool compare_exchange_weak(
      T* & old_value, T* new_value,
      memory_order order = memory_order_seq_cst) noexcept;
  bool compare_exchange_weak(
      T* & old_value, T* new_value,
      memory_order success_order,memory_order failure_order)
      volatile noexcept;
  bool compare_exchange_weak(
      T* & old_value, T* new_value,
      memory_order success_order,memory_order failure_order) noexcept;

  operator T*() const volatile noexcept;
  operator T*() const noexcept;

  T* fetch_add(
      ptrdiff_t,memory_order = memory_order_seq_cst) volatile noexcept;
  T* fetch_add(
      ptrdiff_t,memory_order = memory_order_seq_cst) noexcept;
  T* fetch_sub(
      ptrdiff_t,memory_order = memory_order_seq_cst) volatile noexcept;
  T* fetch_sub(
      ptrdiff_t,memory_order = memory_order_seq_cst) noexcept;

  T* operator++() volatile noexcept;
  T* operator++() noexcept;
  T* operator++(int) volatile noexcept;
  T* operator++(int) noexcept;
  T* operator--() volatile noexcept;
  T* operator--() noexcept;
  T* operator--(int) volatile noexcept;
  T* operator--(int) noexcept;

  T* operator+=(ptrdiff_t) volatile noexcept;
  T* operator+=(ptrdiff_t) noexcept;
  T* operator-=(ptrdiff_t) volatile noexcept;
  T* operator-=(ptrdiff_t) noexcept;
};

bool atomic_is_lock_free(volatile const atomic<T*>*) noexcept;
bool atomic_is_lock_free(const atomic<T*>*) noexcept;
void atomic_init(volatile atomic<T*>*, T*) noexcept;
void atomic_init(atomic<T*>*, T*) noexcept;
T* atomic_exchange(volatile atomic<T*>*, T*) noexcept;
T* atomic_exchange(atomic<T*>*, T*) noexcept;
T* atomic_exchange_explicit(volatile atomic<T*>*, T*, memory_order)
  noexcept;
T* atomic_exchange_explicit(atomic<T*>*, T*, memory_order) noexcept;
void atomic_store(volatile atomic<T*>*, T*) noexcept;
void atomic_store(atomic<T*>*, T*) noexcept;
void atomic_store_explicit(volatile atomic<T*>*, T*, memory_order)
  noexcept;
void atomic_store_explicit(atomic<T*>*, T*, memory_order) noexcept;
T* atomic_load(volatile const atomic<T*>*) noexcept;
T* atomic_load(const atomic<T*>*) noexcept;
T* atomic_load_explicit(volatile const atomic<T*>*, memory_order) noexcept;
T* atomic_load_explicit(const atomic<T*>*, memory_order) noexcept;
bool atomic_compare_exchange_strong(
  volatile atomic<T*>*,T* * old_value,T* new_value) noexcept;
bool atomic_compare_exchange_strong(
  volatile atomic<T*>*,T* * old_value,T* new_value) noexcept;
bool atomic_compare_exchange_strong_explicit(
  atomic<T*>*,T* * old_value,T* new_value,
  memory_order success_order,memory_order failure_order) noexcept;
bool atomic_compare_exchange_strong_explicit(
  atomic<T*>*,T* * old_value,T* new_value,
  memory_order success_order,memory_order failure_order) noexcept;
bool atomic_compare_exchange_weak(
  volatile atomic<T*>*,T* * old_value,T* new_value) noexcept;
bool atomic_compare_exchange_weak(
  atomic<T*>*,T* * old_value,T* new_value) noexcept;
bool atomic_compare_exchange_weak_explicit(
  volatile atomic<T*>*,T* * old_value, T* new_value,
  memory_order success_order,memory_order failure_order) noexcept;
bool atomic_compare_exchange_weak_explicit(
  atomic<T*>*,T* * old_value, T* new_value,
  memory_order success_order,memory_order failure_order) noexcept;

T* atomic_fetch_add(volatile atomic<T*>*, ptrdiff_t) noexcept;
T* atomic_fetch_add(atomic<T*>*, ptrdiff_t) noexcept;
T* atomic_fetch_add_explicit(
  volatile atomic<T*>*, ptrdiff_t, memory_order) noexcept;
T* atomic_fetch_add_explicit(
  atomic<T*>*, ptrdiff_t, memory_order) noexcept;
T* atomic_fetch_sub(volatile atomic<T*>*, ptrdiff_t) noexcept;
T* atomic_fetch_sub(atomic<T*>*, ptrdiff_t) noexcept;
T* atomic_fetch_sub_explicit(
  volatile atomic<T*>*, ptrdiff_t, memory_order) noexcept;
T* atomic_fetch_sub_explicit(
  atomic<T*>*, ptrdiff_t, memory_order) noexcept;
```

在主模板中也提供了一些相同的操作(可见11.3.8节)。

####std::atomic&lt;T*&gt;::fetch_add 成员函数

原子的加载一个值，然后使用与提供i相加(使用标准指针运算规则)的结果，替换掉原值。

**声明**
```c++
T* fetch_add(
    ptrdiff_t i,memory_order order = memory_order_seq_cst)
    volatile noexcept;
T* fetch_add(
    ptrdiff_t i,memory_order order = memory_order_seq_cst) noexcept;
```

**效果**<br>
原子的查询*this中的值，将old-value+i的和存回*this。

**返回**<br>
返回*this之前存储的值。

**抛出**<br>
无

**NOTE**:对于*this的内存地址来说，这是一个“读-改-写”操作。

####std::atomic_fetch_add 非成员函数

从`atomic<T*>`实例中原子的读取一个值，并且将其与给定i值进行位相加操作(使用标准指针运算规则)后，替换原值。

**声明**
```c++
T* atomic_fetch_add(volatile atomic<T*>* p, ptrdiff_t i) noexcept;
T* atomic_fetch_add(atomic<T*>* p, ptrdiff_t i) noexcept;
```

**效果**
```c++
return p->fetch_add(i);
```

####std::atomic_fetch_add_explicit 非成员函数

从`atomic<T*>`实例中原子的读取一个值，并且将其与给定i值进行位相加操作(使用标准指针运算规则)后，替换原值。

**声明**
```c++
T* atomic_fetch_add_explicit(
     volatile atomic<T*>* p, ptrdiff_t i,memory_order order) noexcept;
T* atomic_fetch_add_explicit(
     atomic<T*>* p, ptrdiff_t i, memory_order order) noexcept;
```

**效果**
```c++
return p->fetch_add(i,order);
```

####std::atomic&lt;T*&gt;::fetch_sub 成员函数

原子的加载一个值，然后使用与提供i相减(使用标准指针运算规则)的结果，替换掉原值。

**声明**
```c++
T* fetch_sub(
    ptrdiff_t i,memory_order order = memory_order_seq_cst)
    volatile noexcept;
T* fetch_sub(
    ptrdiff_t i,memory_order order = memory_order_seq_cst) noexcept;
```

**效果**<br>
原子的查询*this中的值，将old-value-i的和存回*this。

**返回**<br>
返回*this之前存储的值。

**抛出**<br>
无

**NOTE**:对于*this的内存地址来说，这是一个“读-改-写”操作。

####std::atomic_fetch_sub 非成员函数

从`atomic<T*>`实例中原子的读取一个值，并且将其与给定i值进行位相减操作(使用标准指针运算规则)后，替换原值。

**声明**
```c++
T* atomic_fetch_sub(volatile atomic<T*>* p, ptrdiff_t i) noexcept;
T* atomic_fetch_sub(atomic<T*>* p, ptrdiff_t i) noexcept;
```

**效果**
```c++
return p->fetch_sub(i);
```

####std::atomic_fetch_sub_explicit 非成员函数

从`atomic<T*>`实例中原子的读取一个值，并且将其与给定i值进行位相减操作(使用标准指针运算规则)后，替换原值。

**声明**
```c++
T* atomic_fetch_sub_explicit(
     volatile atomic<T*>* p, ptrdiff_t i,memory_order order) noexcept;
T* atomic_fetch_sub_explicit(
     atomic<T*>* p, ptrdiff_t i, memory_order order) noexcept;
```

**效果**
```c++
return p->fetch_sub(i,order);
```

####std::atomic&lt;T*&gt;::operator++ 前置递增操作

原子的将*this中存储的值加1(使用标准指针运算规则)，并返回新值。

**声明**
```c++
T* operator++() volatile noexcept;
T* operator++() noexcept;
```

**效果**
```c++
return this->fetch_add(1) + 1;
```

####std::atomic&lt;T*&gt;::operator++ 后置递增操作

原子的将*this中存储的值加1(使用标准指针运算规则)，并返回旧值。

**声明**
```c++
T* operator++() volatile noexcept;
T* operator++() noexcept;
```

**效果**
```c++
return this->fetch_add(1);
```

####std::atomic&lt;T*&gt;::operator-- 前置递减操作

原子的将*this中存储的值减1(使用标准指针运算规则)，并返回新值。

**声明**
```c++
T* operator--() volatile noexcept;
T* operator--() noexcept;
```

**效果**
```c++
return this->fetch_sub(1) - 1;
```

####std::atomic&lt;T*&gt;::operator-- 后置递减操作

原子的将*this中存储的值减1(使用标准指针运算规则)，并返回旧值。

**声明**
```c++
T* operator--() volatile noexcept;
T* operator--() noexcept;
```

**效果**
```c++
return this->fetch_sub(1);
```

####std::atomic&lt;T*&gt;::operator+= 复合赋值操作

原子的将*this中存储的值与给定值相加(使用标准指针运算规则)，并返回新值。

**声明**
```c++
T* operator+=(ptrdiff_t i) volatile noexcept;
T* operator+=(ptrdiff_t i) noexcept;
```

**效果**
```c++
return this->fetch_add(i) + i;
```

####std::atomic&lt;T*&gt;::operator-= 复合赋值操作


原子的将*this中存储的值与给定值相减(使用标准指针运算规则)，并返回新值。

**声明**
```c++
T* operator+=(ptrdiff_t i) volatile noexcept;
T* operator+=(ptrdiff_t i) noexcept;
```

**效果**
```c++
return this->fetch_add(i) - i;
```

##D.4 &lt;future&gt;头文件

`<future>`头文件提供处理异步结果(在其他线程上执行额结果)的工具。

**头文件内容**
```c++
namespace std
{
  enum class future_status {
      ready, timeout, deferred };

  enum class future_errc
  {
    broken_promise,
    future_already_retrieved,
    promise_already_satisfied,
    no_state
  };

  class future_error;

  const error_category& future_category();

  error_code make_error_code(future_errc e);
  error_condition make_error_condition(future_errc e);

  template<typename ResultType>
  class future;

  template<typename ResultType>
  class shared_future;

  template<typename ResultType>
  class promise;

  template<typename FunctionSignature>
  class packaged_task; // no definition provided

  template<typename ResultType,typename ... Args>
  class packaged_task<ResultType (Args...)>;

  enum class launch {
    async, deferred
  };

  template<typename FunctionType,typename ... Args>
  future<result_of<FunctionType(Args...)>::type>
  async(FunctionType&& func,Args&& ... args);

  template<typename FunctionType,typename ... Args>
  future<result_of<FunctionType(Args...)>::type>
  async(std::launch policy,FunctionType&& func,Args&& ... args);
}
```

###D.4.1 std::future类型模板

`std::future`类型模板是为了等待其他线程上的异步结果。其和`std::promise`，`std::packaged_task`类型模板，还有`std::async`函数模板，都是为异步结果准备的工具。只有`std::future`实例可以在任意时间引用异步结果。

`std::future`实例是MoveConstructible(移动构造)和MoveAssignable(移动赋值)，不过不能CopyConstructible(拷贝构造)和CopyAssignable(拷贝赋值)。

**类型声明**
```c++
template<typename ResultType>
class future
{
public:
  future() noexcept;
  future(future&&) noexcept;
  future& operator=(future&&) noexcept;
  ~future();
  
  future(future const&) = delete;
  future& operator=(future const&) = delete;

  shared_future<ResultType> share();

  bool valid() const noexcept;
  
  see description get();
 
  void wait();

  template<typename Rep,typename Period>
  future_status wait_for(
      std::chrono::duration<Rep,Period> const& relative_time);

  template<typename Clock,typename Duration>
  future_status wait_until(
      std::chrono::time_point<Clock,Duration> const& absolute_time);
};
```

####std::future 默认构造函数

不使用异步结果构造一个`std::future`对象。

**声明**
```c++
future() noexcept;
```

**效果**<br>
构造一个新的`std::future`实例。

**后置条件**<br>
valid()返回false。

**抛出**<br>
无

####std::future 移动构造函数

使用另外一个对象，构造一个`std::future`对象，将相关异步结果的所有权转移给新`std::future`对象。

**声明**
```c++
future(future&& other) noexcept;
```

**效果**<br>
使用已有对象构造一个新的`std::future`对象。

**后置条件**<br>
已有对象中的异步结果，将于新的对象相关联。然后，解除已有对象和异步之间的关系。`this->valid()`返回的结果与之前已有对象`other.valid()`返回的结果相同。在调用该构造函数后，`other.valid()`将返回false。

**抛出**<br>
无

####std::future 移动赋值操作

将已有`std::future`对象中异步结果的所有权，转移到另一对象当中。

**声明**
```c++
future(future&& other) noexcept;
```

**效果**<br>
在两个`std::future`实例中转移异步结果的状态。

**后置条件**<br>
当执行完赋值操作后，`*this.other`就与异步结果没有关系了。异步状态(如果有的话)在释放后与`*this`相关，并且在最后一次引用后，销毁该状态。`this->valid()`返回的结果与之前已有对象`other.valid()`返回的结果相同。在调用该构造函数后，`other.valid()`将返回false。

**抛出**<br>
无

####std::future 析构函数

销毁一个`std::future`对象。

**声明**
```c++
~future();
```

**效果**<br>
销毁`*this`。如果这是最后一次引用与`*this`相关的异步结果，之后就会将该异步结果销毁。

**抛出**<br>
无

####std::future::share 成员函数

构造一个新`std::shared_future`实例，并且将`*this`异步结果的所有权转移到新的`std::shared_future`实例中。

**声明**
```c++
shared_future<ResultType> share();
```

**效果**<br>
如同 shared_future<ResultType>(std::move(*this))。

**后置条件**<br>
当调用share()成员函数，与`*this`相关的异步结果将与新构造的`std::shared_future`实例相关。`this->valid()`将返回false。

**抛出**<br>
无

####std::future::valid 成员函数

检查`std::future`实例是否与一个异步结果相关联。

**声明**
```c++
bool valid() const noexcept;
```

**返回**<br>
当与异步结果相关时，返回true，否则返回false。

**抛出**<br>
无

####std::future::wait 成员函数

如果与`*this`相关的状态包含延迟函数，将调用该函数。否则，会等待`std::future`实例中的异步结果准备就绪。

**声明**
```c++
void wait();
```

**先决条件**<br>
`this->valid()`将会返回true。

**效果**<br>
当相关状态包含延迟函数，调用延迟函数，并保存返回的结果，或将抛出的异常保存成为异步结果。否则，会阻塞到`*this`准备就绪。

**抛出**<br>
无

####std::future::wait_for 成员函数

等待`std::future`实例上相关异步结果准备就绪，或超过某个给定的时间。

**声明**
```c++
template<typename Rep,typename Period>
future_status wait_for(
    std::chrono::duration<Rep,Period> const& relative_time);
```

**先决条件**<br>
`this->valid()`将会返回true。

**效果**<br>
如果与`*this`相关的异步结果包含一个`std::async`调用的延迟函数(还未执行)，那么就不阻塞立即返回。否则将阻塞实例，直到与`*this`相关异步结果准备就绪，或超过给定的relative_time时长。

**返回**<br>
当与`*this`相关的异步结果包含一个`std::async`调用的延迟函数(还未执行)，返回`std::future_status::deferred`；当与`*this`相关的异步结果准备就绪，返回`std::future_status::ready`；当给定时间超过relative_time时，返回`std::future_status::timeout`。

**NOTE**:线程阻塞的时间可能超多给定的时长。时长尽可能由一个稳定的时钟决定。

**抛出**<br>
无

####std::future::wait_until 成员函数

等待`std::future`实例上相关异步结果准备就绪，或超过某个给定的时间。

**声明**
```c++
template<typename Clock,typename Duration>
future_status wait_until(
  std::chrono::time_point<Clock,Duration> const& absolute_time);
```

**先决条件**<br>
this->valid()将返回true。

**效果**<br>
如果与`*this`相关的异步结果包含一个`std::async`调用的延迟函数(还未执行)，那么就不阻塞立即返回。否则将阻塞实例，直到与`*this`相关异步结果准备就绪，或`Clock::now()`返回的时间大于等于absolute_time。

**返回**<br>
当与`*this`相关的异步结果包含一个`std::async`调用的延迟函数(还未执行)，返回`std::future_status::deferred`；当与`*this`相关的异步结果准备就绪，返回`std::future_status::ready`；`Clock::now()`返回的时间大于等于absolute_time，返回`std::future_status::timeout`。

**NOTE**:这里不保证调用线程会被阻塞多久，只有函数返回`std::future_status::timeout`，然后`Clock::now()`返回的时间大于等于absolute_time的时候，线程才会解除阻塞。

**抛出**<br>
无

####std::future::get 成员函数

当相关状态包含一个`std::async`调用的延迟函数，调用该延迟函数，并返回结果；否则，等待与`std::future`实例相关的异步结果准备就绪，之后返回存储的值或异常。

**声明**
```c++
void future<void>::get();
R& future<R&>::get();
R future<R>::get();
```

**先决条件**<br>
this->valid()将返回true。

**效果**<br>
如果*this相关状态包含一个延期函数，那么调用这个函数并返回结果，或将抛出的异常进行传播。

否则，线程就要被阻塞，直到与*this相关的异步结果就绪。当结果存储了一个异常，那么就就会将存储异常抛出。否则，将会返回存储值。

**返回**<br>
当相关状态包含一个延期函数，那么这个延期函数的结果将被返回。否则，当ResultType为void时，就会按照常规调用返回。如果ResultType是R&(R类型的引用)，存储的引用值将会被返回。否则，存储的值将会返回。

**抛出**<br>
异常由延期函数，或存储在异步结果中的异常(如果有的话)抛出。

**后置条件**
```c++
this->valid()==false
```

###D.4.2 std::shared_future类型模板

`std::shared_future`类型模板是为了等待其他线程上的异步结果。其和`std::promise`，`std::packaged_task`类型模板，还有`std::async`函数模板，都是为异步结果准备的工具。多个`std::shared_future`实例可以引用同一个异步结果。

`std::shared_future`实例是CopyConstructible(拷贝构造)和CopyAssignable(拷贝赋值)。你也可以同ResultType的`std::future`类型对象，移动构造一个`std::shared_future`类型对象。

访问给定`std::shared_future`实例是非同步的。因此，当有多个线程访问同一个`std::shared_future`实例，且无任何外围同步操作时，这样的访问是不安全的。不过访问关联状态时是同步的，所以多个线程访问多个独立的`std::shared_future`实例，且没有外围同步操作的时候，是安全的。

**类型定义**
```c++
template<typename ResultType>
class shared_future
{
public:
  shared_future() noexcept;
  shared_future(future<ResultType>&&) noexcept;
  
  shared_future(shared_future&&) noexcept;
  shared_future(shared_future const&);
  shared_future& operator=(shared_future const&);
  shared_future& operator=(shared_future&&) noexcept;
  ~shared_future();

  bool valid() const noexcept;

  see description get() const;

  void wait() const;

  template<typename Rep,typename Period>
  future_status wait_for(
     std::chrono::duration<Rep,Period> const& relative_time) const;

  template<typename Clock,typename Duration>
  future_status wait_until(
     std::chrono::time_point<Clock,Duration> const& absolute_time)
    const;
};
```

####std::shared_future 默认构造函数

不使用关联异步结果，构造一个`std::shared_future`对象。

**声明**
```c++
shared_future() noexcept;
```

**效果**<br>
构造一个新的`std::shared_future`实例。

**后置条件**<br>
当新实例构建完成后，调用valid()将返回false。

**抛出**<br>
无

####std::shared_future 移动构造函数

以一个已创建`std::shared_future`对象为准，构造`std::shared_future`实例，并将使用`std::shared_future`对象关联的异步结果的所有权转移到新的实例中。

**声明**
```c++
shared_future(shared_future&& other) noexcept;
```

**效果**<br>
构造一个新`std::shared_future`实例。

**后置条件**<br>
将other对象中关联异步结果的所有权转移到新对象中，这样other对象就没有与之相关联的异步结果了。

**抛出**<br>
无

####std::shared_future 移动对应std::future对象的构造函数

以一个已创建`std::future`对象为准，构造`std::shared_future`实例，并将使用`std::shared_future`对象关联的异步结果的所有权转移到新的实例中。

**声明**
```c++
shared_future(std::future<ResultType>&& other) noexcept;
```

**效果**<br>
构造一个`std::shared_future`对象。

**后置条件**<br>
将other对象中关联异步结果的所有权转移到新对象中，这样other对象就没有与之相关联的异步结果了。

**抛出**<br>
无

####std::shared_future 拷贝构造函数

以一个已创建`std::future`对象为准，构造`std::shared_future`实例，并将使用`std::shared_future`对象关联的异步结果(如果有的话)拷贝到新创建对象当中，两个对象共享该异步结果。

**声明**
```c++
shared_future(shared_future const& other);
```

**效果**<br>
构造一个`std::shared_future`对象。

**后置条件**<br>
将other对象中关联异步结果拷贝到新对象中，与other共享关联的异步结果。

**抛出**<br>
无

####std::shared_future 析构函数

销毁一个`std::shared_future`对象。

**声明**
```c++
~shared_future();
```

**效果**<br>
将`*this`销毁。如果`*this`关联的异步结果与`std::promise`或`std::packaged_task`不再有关联，那么该函数将会切断`std::shared_future`实例与异步结果的联系，并销毁异步结果。

**抛出**<br>
无

####std::shared_future::valid 成员函数

检查`std::shared_future`实例是否与一个异步结果相关联。

**声明**
```c++
bool valid() const noexcept;
```

**返回**<br>
当与异步结果相关时，返回true，否则返回false。

**抛出**<br>
无

####std::shared_future::wait 成员函数

当*this关联状态包含一个延期函数，那么调用这个函数。否则，等待直到与`std::shared_future`实例相关的异步结果就绪为止。

**声明**
```c++
void wait() const;
```

**先决条件**
this->valid()将返回true。

**效果**<br>
当有多个线程调用`std::shared_future`实例上的get()和wait()时，实例会序列化的共享同一关联状态。如果关联状态包括一个延期函数，那么第一个调用get()或wait()时就会调用延期函数，并且存储返回值，或将抛出异常以异步结果的方式保存下来。

**抛出**<br>
无

####std::shared_future::wait_for 成员函数

等待`std::shared_future`实例上相关异步结果准备就绪，或超过某个给定的时间。

**声明**
```c++
template<typename Rep,typename Period>
future_status wait_for(
    std::chrono::duration<Rep,Period> const& relative_time) const;
```

**先决条件**<br>
`this->valid()`将会返回true。

**效果**<br>
如果与`*this`相关的异步结果包含一个`std::async`调用的延期函数(还未执行)，那么就不阻塞立即返回。否则将阻塞实例，直到与`*this`相关异步结果准备就绪，或超过给定的relative_time时长。

**返回**<br>
当与`*this`相关的异步结果包含一个`std::async`调用的延迟函数(还未执行)，返回`std::future_status::deferred`；当与`*this`相关的异步结果准备就绪，返回`std::future_status::ready`；当给定时间超过relative_time时，返回`std::future_status::timeout`。

**NOTE**:线程阻塞的时间可能超多给定的时长。时长尽可能由一个稳定的时钟决定。

**抛出**<br>
无

####std::shared_future::wait_until 成员函数

等待`std::future`实例上相关异步结果准备就绪，或超过某个给定的时间。

**声明**
```c++
template<typename Clock,typename Duration>
future_status wait_until(
  std::chrono::time_point<Clock,Duration> const& absolute_time) const;
```

**先决条件**<br>
this->valid()将返回true。

**效果**<br>
如果与`*this`相关的异步结果包含一个`std::async`调用的延迟函数(还未执行)，那么就不阻塞立即返回。否则将阻塞实例，直到与`*this`相关异步结果准备就绪，或`Clock::now()`返回的时间大于等于absolute_time。

**返回**<br>
当与`*this`相关的异步结果包含一个`std::async`调用的延迟函数(还未执行)，返回`std::future_status::deferred`；当与`*this`相关的异步结果准备就绪，返回`std::future_status::ready`；`Clock::now()`返回的时间大于等于absolute_time，返回`std::future_status::timeout`。

**NOTE**:这里不保证调用线程会被阻塞多久，只有函数返回`std::future_status::timeout`，然后`Clock::now()`返回的时间大于等于absolute_time的时候，线程才会解除阻塞。

**抛出**<br>
无

####std::shared_future::get 成员函数

当相关状态包含一个`std::async`调用的延迟函数，调用该延迟函数，并返回结果；否则，等待与`std::shared_future`实例相关的异步结果准备就绪，之后返回存储的值或异常。

**声明**
```c++
void shared_future<void>::get() const;
R& shared_future<R&>::get() const;
R const& shared_future<R>::get() const;
```

**先决条件**<br>
this->valid()将返回true。

**效果**<br>
当有多个线程调用`std::shared_future`实例上的get()和wait()时，实例会序列化的共享同一关联状态。如果关联状态包括一个延期函数，那么第一个调用get()或wait()时就会调用延期函数，并且存储返回值，或将抛出异常以异步结果的方式保存下来。

阻塞会知道*this关联的异步结果就绪后解除。当异步结果存储了一个一行，那么就会抛出这个异常。否则，返回存储的值。

**返回**<br>
当ResultType为void时，就会按照常规调用返回。如果ResultType是R&(R类型的引用)，存储的引用值将会被返回。否则，返回存储值的const引用。

**抛出**<br>
抛出存储的异常(如果有的话)。

###D.4.3 std::packaged_task类型模板

`std::packaged_task`类型模板可打包一个函数或其他可调用对象，所以当函数通过`std::packaged_task`实例被调用时，结果将会作为异步结果。这个结果可以通过检索`std::future`实例来查找。

`std::packaged_task`实例是可以MoveConstructible(移动构造)和MoveAssignable(移动赋值)，不过不能CopyConstructible(拷贝构造)和CopyAssignable(拷贝赋值)。

**类型定义**
```c++
template<typename FunctionType>
class packaged_task; // undefined

template<typename ResultType,typename... ArgTypes>
class packaged_task<ResultType(ArgTypes...)>
{
public:
  packaged_task() noexcept;
  packaged_task(packaged_task&&) noexcept;
  ~packaged_task();

  packaged_task& operator=(packaged_task&&) noexcept;

  packaged_task(packaged_task const&) = delete;
  packaged_task& operator=(packaged_task const&) = delete;

  void swap(packaged_task&) noexcept;

  template<typename Callable>
  explicit packaged_task(Callable&& func);

  template<typename Callable,typename Allocator>
  packaged_task(std::allocator_arg_t, const Allocator&,Callable&&);

  bool valid() const noexcept;
  std::future<ResultType> get_future();
  void operator()(ArgTypes...);
  void make_ready_at_thread_exit(ArgTypes...);
  void reset();
};
```

####std::packaged_task 默认构造函数

构造一个`std::packaged_task`对象。

**声明**
```c++
packaged_task() noexcept;
```

**效果**<br>
不使用关联任务或异步结果来构造一个`std::packaged_task`对象。

**抛出**<br>
无

####std::packaged_task 通过可调用对象构造

使用关联任务和异步结果，构造一个`std::packaged_task`对象。

**声明**
```c++
template<typename Callable>
packaged_task(Callable&& func);
```

**先决条件**<br>
表达式`func(args...)`必须是合法的，并且在`args...`中的args-i参数，必须是`ArgTypes...`中ArgTypes-i类型的一个值。且返回值必须可转换为ResultType。

**效果**<br>
使用ResultType类型的关联异步结果，构造一个`std::packaged_task`对象，异步结果是未就绪的，并且Callable类型相关的任务是对func的一个拷贝。

**抛出**<br>
当构造函数无法为异步结果分配出内存时，会抛出`std::bad_alloc`类型的异常。其他异常会在使用Callable类型的拷贝或移动构造过程中抛出。

####std::packaged_task 通过有分配器的可调用对象构造

使用关联任务和异步结果，构造一个`std::packaged_task`对象。使用以提供的分配器为关联任务和异步结果分配内存。

**声明**
```c++
template<typename Allocator,typename Callable>
packaged_task(
    std::allocator_arg_t, Allocator const& alloc,Callable&& func);
```

**先决条件**<br>
表达式`func(args...)`必须是合法的，并且在`args...`中的args-i参数，必须是`ArgTypes...`中ArgTypes-i类型的一个值。且返回值必须可转换为ResultType。

**效果**<br>
使用ResultType类型的关联异步结果，构造一个`std::packaged_task`对象，异步结果是未就绪的，并且Callable类型相关的任务是对func的一个拷贝。异步结果和任务的内存通过内存分配器alloc进行分配，或进行拷贝。

**抛出**<br>
当构造函数无法为异步结果分配出内存时，会抛出`std::bad_alloc`类型的异常。其他异常会在使用Callable类型的拷贝或移动构造过程中抛出。

####std::packaged_task 移动构造函数

通过一个`std::packaged_task`对象构建另一个，将与已存在的`std::packaged_task`相关的异步结果和任务的所有权转移到新构建的对象当中。

**声明**
```c++
packaged_task(packaged_task&& other) noexcept;
```

**效果**<br>
构建一个新的`std::packaged_task`实例。

**后置条件**<br>
通过other构建新的`std::packaged_task`对象。在新对象构建完成后，other与其之前相关联的异步结果就没有任何关系了。

**抛出**<br>
无

####std::packaged_task 移动赋值操作

将一个`std::packaged_task`对象相关的异步结果的所有权转移到另外一个。

**声明**
```c++
packaged_task& operator=(packaged_task&& other) noexcept;
```

**效果**<br>
将other相关异步结果和任务的所有权转移到`*this`中，并且切断异步结果和任务与other对象的关联，如同`std::packaged_task(other).swap(*this)`。

**后置条件**<br>
与other相关的异步结果与任务移动转移，使*this.other无关联的异步结果。

**返回**
```c++
*this
```

**抛出**<br>
无

####std::packaged_task::swap 成员函数

将两个`std::packaged_task`对象所关联的异步结果的所有权进行交换。

**声明**
```c++
void swap(packaged_task& other) noexcept;
```

**效果**<br>
将other和*this关联的异步结果与任务进行交换。

**后置条件**<br>
将与other关联的异步结果和任务，通过调用swap的方式，与*this相交换。

**抛出**<br>
无

####std::packaged_task 析构函数

销毁一个`std::packaged_task`对象。

**声明**
```c++
~packaged_task();
```

**效果**<br>
将`*this`销毁。如果`*this`有关联的异步结果，并且结果不是一个已存储的任务或异常，那么异步结果状态将会变为就绪，伴随就绪的是一个`std::future_error`异常和错误码`std::future_errc::broken_promise`。

**抛出**<br>
无

####std::packaged_task::get_future 成员函数

在*this相关异步结果中，检索一个`std::future`实例。

**声明**
```c++
std::future<ResultType> get_future();
```

**先决条件**<br>
*this具有关联异步结果。

**返回**<br>
一个与*this关联异构结果相关的一个`std::future`实例。

**抛出**<br>
如果一个`std::future`已经通过get_future()获取了异步结果，在抛出`std::future_error`异常时，错误码是`std::future_errc::future_already_retrieved`

####std::packaged_task::reset 成员函数

将一个`std::packaged_task`对实例与一个新的异步结果相关联。

**声明**
```c++
void reset();
```

**先决条件**<br>
*this具有关联的异步任务。

**效果**<br>
如同`*this=packaged_task(std::move(f))`，f是*this中已存储的关联任务。

**抛出**<br>
如果内存不足以分配给新的异构结果，那么将会抛出`std::bad_alloc`类异常。

####std::packaged_task::valid 成员函数

检查*this中是都具有关联任务和异步结果。

**声明**
```c++
bool valid() const noexcept;
```

**返回**<br>
当*this具有相关任务和异步结构，返回true；否则，返回false。

**抛出**<br>
无

####std::packaged_task::operator() 函数调用操作

调用一个`std::packaged_task`实例中的相关任务，并且存储返回值，或将异常存储到异常结果当中。

**声明**
```c++
void operator()(ArgTypes... args);
```

**先决条件**<br>
*this具有相关任务。

**效果**<br>
像`INVOKE(func,args...)`那要调用相关的函数func。如果返回征程，那么将会存储到*this相关的异步结果中。当返回结果是一个异常，将这个异常存储到*this相关的异步结果中。

**后置条件**<br>
*this相关联的异步结果状态为就绪，并且存储了一个值或异常。所有阻塞线程，在等待到异步结果的时候被解除阻塞。

**抛出**<br>
当异步结果已经存储了一个值或异常，那么将抛出一个`std::future_error`异常，错误码为`std::future_errc::promise_already_satisfied`。

**同步**<br>
`std::future<ResultType>::get()`或`std::shared_future<ResultType>::get()`的成功调用，代表同步操作的成功，函数将会检索异步结果中的值或异常。

####std::packaged_task::make_ready_at_thread_exit 成员函数

调用一个`std::packaged_task`实例中的相关任务，并且存储返回值，或将异常存储到异常结果当中，直到线程退出时，将相关异步结果的状态置为就绪。

**声明**
```c++
void make_ready_at_thread_exit(ArgTypes... args);
```

**先决条件**<br>
*this具有相关任务。

**效果**<br>
像`INVOKE(func,args...)`那要调用相关的函数func。如果返回征程，那么将会存储到`*this`相关的异步结果中。当返回结果是一个异常，将这个异常存储到`*this`相关的异步结果中。当当前线程退出的时候，可调配相关异步状态为就绪。

**后置条件**<br>
*this的异步结果中已经存储了一个值或一个异常，不过在当前线程退出的时候，这个结果都是非就绪的。当当前线程退出时，阻塞等待异步结果的线程将会被解除阻塞。

**抛出**<br>
当异步结果已经存储了一个值或异常，那么将抛出一个`std::future_error`异常，错误码为`std::future_errc::promise_already_satisfied`。当无关联异步状态时，抛出`std::future_error`异常，错误码为`std::future_errc::no_state`。

**同步**<br>
`std::future<ResultType>::get()`或`std::shared_future<ResultType>::get()`在线程上的成功调用，代表同步操作的成功，函数将会检索异步结果中的值或异常。

###D.4.4 std::promise类型模板

`std::promise`类型模板提供设置异步结果的方法，这样其他线程就可以通过`std::future`实例来索引该结果。

ResultType模板参数，该类型可以存储异步结果。

`std::promise`实例中的异步结果与某个`srd::future`实例相关联，并且可以通过调用get_future()成员函数来获取这个`srd::future`实例。ResultType类型的异步结果，可以通过set_value()成员函数对存储值进行设置，或者使用set_exception()将对应异常设置进异步结果中。

`std::promise`实例是可以MoveConstructible(移动构造)和MoveAssignable(移动赋值)，但是不能CopyConstructible(拷贝构造)和CopyAssignable(拷贝赋值)。

**类型定义**
```c++
template<typename ResultType>
class promise
{
public:
  promise();
  promise(promise&&) noexcept;
  ~promise();
  promise& operator=(promise&&) noexcept;

  template<typename Allocator>
  promise(std::allocator_arg_t, Allocator const&);

  promise(promise const&) = delete;
  promise& operator=(promise const&) = delete;

  void swap(promise& ) noexcept;
  
  std::future<ResultType> get_future();

  void set_value(see description);
  void set_exception(std::exception_ptr p);
};
```

####std::promise 默认构造函数

构造一个`std::promise`对象。

**声明**
```c++
promise();
```

**效果**<br>
使用ResultType类型的相关异步结果来构造`std::promise`实例，不过异步结果并未就绪。

**抛出**<br>
当没有足够内存为异步结果进行分配，那么将抛出`std::bad_alloc`型异常。

####std::promise 带分配器的构造函数

构造一个`std::promise`对象，使用提供的分配器来为相关异步结果分配内存。

**声明**
```c++
template<typename Allocator>
promise(std::allocator_arg_t, Allocator const& alloc);
```

**效果**<br>
使用ResultType类型的相关异步结果来构造`std::promise`实例，不过异步结果并未就绪。异步结果的内存由alloc分配器来分配。

**抛出**<br>
当分配器为异步结果分配内存时，如有抛出异常，就为该函数抛出的异常。

####std::promise 移动构造函数

通过另一个已存在对象，构造一个`std::promise`对象。将已存在对象中的相关异步结果的所有权转移到新创建的`std::promise`对象当中。

**声明**
```c++
promise(promise&& other) noexcept;
```

**效果**<br>
构造一个`std::promise`实例。

**后置条件**<br>
当使用other来构造一个新的实例，那么other中相关异构结果的所有权将转移到新创建的对象上。之后，other将无关联异步结果。

**抛出**<br>
无

####std::promise 移动赋值操作符

在两个`std::promise`实例中转移异步结果的所有权。

**声明**
```c++
promise& operator=(promise&& other) noexcept;
```

**效果**<br>
在other和`*this`之间进行异步结果所有权的转移。当`*this`已经有关联的异步结果，那么该异步结果的状态将会为就绪态，且伴随一个`std::future_error`类型异常，错误码为`std::future_errc::broken_promise`。

**后置条件**<br>
将other中关联的异步结果转移到*this当中。other中将无关联异步结果。

**返回**<br>
```c++
*this
```

**抛出**<br>
无

####std::promise::swap 成员函数

将两个`std::promise`实例中的关联异步结果进行交换。

**声明**
```c++
void swap(promise& other);
```

**效果**<br>
交换other和*this当中的关联异步结果。

**后置条件**<br>
当swap使用other时，other中的异步结果就会与*this中关联异步结果相交换。二者返回来亦然。

**抛出**<br>
无

####std::promise 析构函数

销毁`std::promise`对象。

**声明**
```c++
~promise();
```

**效果**<br>
销毁`*this`。当`*this`具有关联的异步结果，并且结果中没有存储值或异常，那么结果将会置为就绪，伴随一个`std::future_error`异常，错误码为`std::future_errc::broken_promise`。

**抛出**<br>
无

####std::promise::get_future 成员函数

通过*this关联的异步结果，检索出所要的`std::future`实例。

**声明**
```c++
std::future<ResultType> get_future();
```

**先决条件**<br>
*this具有关联异步结果。

**返回**<br>
与*this关联异步结果关联的`std::future`实例。

**抛出**<br>
当`std::future`已经通过get_future()获取过了，将会抛出一个`std::future_error`类型异常，伴随的错误码为`std::future_errc::future_already_retrieved`。

####std::promise::set_value 成员函数

存储一个值到与*this关联的异步结果中。

**声明**
```c++
void promise<void>::set_value();
void promise<R&>::set_value(R& r);
void promise<R>::set_value(R const& r);
void promise<R>::set_value(R&& r);
```

**先决条件**<br>
*this具有关联异步结果。

**效果**<br>
当ResultType不是void型，就存储r到*this相关的异步结果当中。

**后置条件**<br>
*this相关的异步结果的状态为就绪，且将值存入。任意等待异步结果的阻塞线程将解除阻塞。

**抛出**<br>
当异步结果已经存有一个值或一个异常，那么将抛出`std::future_error`型异常，伴随错误码为`std::future_errc::promise_already_satisfied`。r的拷贝构造或移动构造抛出的异常，即为本函数抛出的异常。

**同步**<br>
并发调用set_value()和set_exception()的线程将被序列化。要想成功的调用set_exception()，需要在之前调用`std::future<Result-Type>::get()`或`std::shared_future<ResultType>::get()`，这两个函数将会查找已存储的异常。

####std::promise::set_value_at_thread_exit 成员函数

存储一个值到与*this关联的异步结果中，到线程退出时，异步结果的状态会被设置为就绪。

**声明**
```c++
void promise<void>::set_value_at_thread_exit();
void promise<R&>::set_value_at_thread_exit(R& r);
void promise<R>::set_value_at_thread_exit(R const& r);
void promise<R>::set_value_at_thread_exit(R&& r);
```

**先决条件**<br>
*this具有关联异步结果。

**效果**<br>
当ResultType不是void型，就存储r到*this相关的异步结果当中。标记异步结果为“已存储值”。当前线程退出时，会安排相关异步结果的状态为就绪。

**后置条件**<br>
将值存入*this相关的异步结果，且直到当前线程退出时，异步结果状态被置为就绪。任何等待异步结果的阻塞线程将解除阻塞。

**抛出**<br>
当异步结果已经存有一个值或一个异常，那么将抛出`std::future_error`型异常，伴随错误码为`std::future_errc::promise_already_satisfied`。r的拷贝构造或移动构造抛出的异常，即为本函数抛出的异常。

**同步**<br>
并发调用set_value(), set_value_at_thread_exit(), set_exception()和set_exception_at_thread_exit()的线程将被序列化。要想成功的调用set_exception()，需要在之前调用`std::future<Result-Type>::get()`或`std::shared_future<ResultType>::get()`，这两个函数将会查找已存储的异常。

####std::promise::set_exception 成员函数

存储一个异常到与*this关联的异步结果中。

**声明**
```c++
void set_exception(std::exception_ptr e);
```

**先决条件**<br>
*this具有关联异步结果。(bool)e为true。

**效果**<br>
将e存储到*this相关的异步结果中。

**后置条件**<br>
在存储异常后，*this相关的异步结果的状态将置为继续。任何等待异步结果的阻塞线程将解除阻塞。

**抛出**<br>
当异步结果已经存有一个值或一个异常，那么将抛出`std::future_error`型异常，伴随错误码为`std::future_errc::promise_already_satisfied`。

**同步**<br>
并发调用set_value()和set_exception()的线程将被序列化。要想成功的调用set_exception()，需要在之前调用`std::future<Result-Type>::get()`或`std::shared_future<ResultType>::get()`，这两个函数将会查找已存储的异常。

####std::promise::set_exception_at_thread_exit 成员函数

存储一个异常到与*this关联的异步结果中，知道当前线程退出，异步结果被置为就绪。

**声明**
```c++
void set_exception_at_thread_exit(std::exception_ptr e);
```

**先决条件**<br>
*this具有关联异步结果。(bool)e为true。

**效果**<br>
将e存储到*this相关的异步结果中。标记异步结果为“已存储值”。当前线程退出时，会安排相关异步结果的状态为就绪。

**后置条件**<br>
将值存入*this相关的异步结果，且直到当前线程退出时，异步结果状态被置为就绪。任何等待异步结果的阻塞线程将解除阻塞。

**抛出**<br>
当异步结果已经存有一个值或一个异常，那么将抛出`std::future_error`型异常，伴随错误码为`std::future_errc::promise_already_satisfied`。

**同步**<br>
并发调用set_value(), set_value_at_thread_exit(), set_exception()和set_exception_at_thread_exit()的线程将被序列化。要想成功的调用set_exception()，需要在之前调用`std::future<Result-Type>::get()`或`std::shared_future<ResultType>::get()`，这两个函数将会查找已存储的异常。

###D.4.5 std::async函数模板

`std::async`能够简单的使用可用的硬件并行来运行自身包含的异步任务。当调用`std::async`返回一个包含任务结果的`std::future`对象。根据投放策略，任务在其所在线程上是异步运行的，当有线程调用了这个future对象的wait()和get()成员函数，则该任务会同步运行。

**声明**
```c++
enum class launch
{
  async,deferred
};

template<typename Callable,typename ... Args>
future<result_of<Callable(Args...)>::type>
async(Callable&& func,Args&& ... args);

template<typename Callable,typename ... Args>
future<result_of<Callable(Args...)>::type>
async(launch policy,Callable&& func,Args&& ... args);
```

**先决条件**<br>
表达式`INVOKE(func,args)`能都为func提供合法的值和args。Callable和Args的所有成员都可MoveConstructible(可移动构造)。

**效果**<br>
在内部存储中拷贝构造`func`和`arg...`(分别使用fff和xyz...进行表示)。

当policy是`std::launch::async`,运行`INVOKE(fff,xyz...)`在所在线程上。当这个线程完成时，返回的`std::future`状态将会为就绪态，并且之后会返回对应的值或异常(由调用函数抛出)。析构函数会等待返回的`std::future`相关异步状态为就绪时，才解除阻塞。

当policy是`std::launch::deferred`，fff和xyx...都会作为延期函数调用，存储在返回的`std::future`。首次调用future的wait()或get()成员函数，将会共享相关状态，之后执行的`INVOKE(fff,xyz...)`与调用wait()或get()函数的线程同步执行。

执行`INVOKE(fff,xyz...)`后，在调用`std::future`的成员函数get()时，就会有值返回或有异常抛出。

当policy是`std::launch::async | std::launch::deferred`或是policy参数被省略，其行为如同已指定的`std::launch::async`或`std::launch::deferred`。具体实现将会通过逐渐递增的方式(call-by-call basis)最大化利用可用的硬件并行，并避免超限分配的问题。

在所有的情况下，`std::async`调用都会直接返回。

**同步**<br>
完成函数调用的先行条件是，需要通过调用`std::future`和`std::shared_future`实例的wait(),get(),wait_for()或wait_until()，返回的对象与`std::async`返回的`std::future`对象关联的状态相同才算成功。就`std::launch::async`这个policy来说，在完成线程上的函数前，也需要先行对上面的函数调用后，成功的返回才行。

**抛出**<br>
当内部存储无法分配所需的空间，将抛出`std::bad_alloc`类型异常；否则，当效果没有达到，或任何异常在构造fff和xyz...发生时，抛出`std::future_error`异常。

##D.5 &lt;mutex&gt;头文件

`<mutex>`头文件提供互斥工具：互斥类型，锁类型和函数，还有确保操作只执行一次的机制。

**头文件内容**
```c++
namespace std
{
  class mutex;
  class recursive_mutex;
  class timed_mutex;
  class recursive_timed_mutex;

  struct adopt_lock_t;
  struct defer_lock_t;
  struct try_to_lock_t;

  constexpr adopt_lock_t adopt_lock{};
  constexpr defer_lock_t defer_lock{};
  constexpr try_to_lock_t try_to_lock{};

  template<typename LockableType>
  class lock_guard;

  template<typename LockableType>
  class unique_lock;

  template<typename LockableType1,typename... LockableType2>
  void lock(LockableType1& m1,LockableType2& m2...);

  template<typename LockableType1,typename... LockableType2>
  int try_lock(LockableType1& m1,LockableType2& m2...);

  struct once_flag;

  template<typename Callable,typename... Args>
  void call_once(once_flag& flag,Callable func,Args args...);
}
```

###D.5.1 std::mutex类

`std::mutex`类型为线程提供基本的互斥和同步工具，这些工具可以用来保护共享数据。互斥量可以用来保护数据，互斥量上锁必须要调用lok()或try_lock()。当有一个线程获取已经获取了锁，那么其他线程想要在获取锁的时候，会在尝试或取锁的时候失败(调用try_lock())或阻塞(调用lock())，具体酌情而定。当线程完成对共享数据的访问，之后就必须调用unlock()对锁进行释放，并且允许其他线程来访问这个共享数据。

`std::mutex`符合Lockable的需求。

**类型定义**
```c++
class mutex
{
public:
  mutex(mutex const&)=delete;
  mutex& operator=(mutex const&)=delete;

  constexpr mutex() noexcept;
  ~mutex();

  void lock();
  void unlock();
  bool try_lock();
};
```

####std::mutex 默认构造函数

构造一个`std::mutex`对象。

**声明**
```c++
constexpr mutex() noexcept;
```

**效果**<br>
构造一个`std::mutex`实例。

**后置条件**<br>
新构造的`std::mutex`对象是未锁的。

**抛出**<br>
无

####std::mutex 析构函数

销毁一个`std::mutex`对象。

**声明**
```c++
~mutex();
```

**先决条件**<br>
*this必须是未锁的。

**效果**<br>
销毁*this。

**抛出**<br>
无

####std::mutex::lock 成员函数

为当前线程获取`std::mutex`上的锁。

**声明**
```c++
void lock();
```

**先决条件**<br>
*this上必须没有持有一个锁。

**效果**<br>
阻塞当前线程，知道*this获取锁。

**后置条件**<br>
*this被调用线程锁住。

**抛出**<br>
当有错误产生，抛出`std::system_error`类型异常。

####std::mutex::try_lock 成员函数

尝试为当前线程获取`std::mutex`上的锁。

**声明**
```c++
bool try_lock();
```

**先决条件**<br>
*this上必须没有持有一个锁。

**效果**<br>
尝试为当前线程*this获取上的锁，失败时当前线程不会被阻塞。

**返回**<br>
当调用线程获取锁时，返回true。

**后置条件**<br>
当*this被调用线程锁住，则返回true。

**抛出**
无

**NOTE** 该函数在获取锁时，可能失败(并返回false)，即使没有其他线程持有*this上的锁。

####std::mutex::unlock 成员函数

释放当前线程获取的`std::mutex`锁。

**声明**
```c++
void unlock();
```

**先决条件**<br>
*this上必须持有一个锁。

**效果**<be>
释放当前线程获取到`*this`上的锁。任意等待获取`*this`上的线程，会在该函数调用后解除阻塞。

**后置条件**<br>
调用线程不在拥有*this上的锁。

**抛出**<br>
无

###D.5.2 std::recursive_mutex类

`std::recursive_mutex`类型为线程提供基本的互斥和同步工具，可以用来保护共享数据。互斥量可以用来保护数据，互斥量上锁必须要调用lok()或try_lock()。当有一个线程获取已经获取了锁，那么其他线程想要在获取锁的时候，会在尝试或取锁的时候失败(调用try_lock())或阻塞(调用lock())，具体酌情而定。当线程完成对共享数据的访问，之后就必须调用unlock()对锁进行释放，并且允许其他线程来访问这个共享数据。

这个互斥量是可递归的，所以一个线程获取`std::recursive_mutex`后可以在之后继续使用lock()或try_lock()来增加锁的计数。只有当线程调用unlock释放锁，其他线程才可能用lock()或try_lock()获取锁。

`std::recursive_mutex`符合Lockable的需求。

**类型定义**
```c++
class recursive_mutex
{
public:
  recursive_mutex(recursive_mutex const&)=delete;
  recursive_mutex& operator=(recursive_mutex const&)=delete;

  recursive_mutex() noexcept;
 ~recursive_mutex();

  void lock();
  void unlock();
  bool try_lock() noexcept;
};
```

####std::recursive_mutex 默认构造函数

构造一个`std::recursive_mutex`对象。

**声明**
```c++
recursive_mutex() noexcept;
```

**效果**<br>
构造一个`std::recursive_mutex`实例。

**后置条件**<br>
新构造的`std::recursive_mutex`对象是未锁的。

**抛出**<br>
当无法创建一个新的`std::recursive_mutex`时，抛出`std::system_error`异常。

####std::recursive_mutex 析构函数

销毁一个`std::recursive_mutex`对象。

**声明**
```c++
~recursive_mutex();
```

**先决条件**<br>
*this必须是未锁的。

**效果**<br>
销毁*this。

**抛出**<br>
无

####std::recursive_mutex::lock 成员函数

为当前线程获取`std::recursive_mutex`上的锁。

**声明**
```c++
void lock();
```

**效果**<br>
阻塞线程，直到获取*this上的锁。

**先决条件**<br>
调用线程锁住*this上的锁。当调用已经持有一个*this的锁时，锁的计数会增加1。

**抛出**<br>
当有错误产生，将抛出`std::system_error`异常。

####std::recursive_mutex::try_lock 成员函数

尝试为当前线程获取`std::recursive_mutex`上的锁。

**声明**
```c++
bool try_lock() noexcept;
```

**效果**<br>
尝试为当前线程*this获取上的锁，失败时当前线程不会被阻塞。

**返回**<br>
当调用线程获取锁时，返回true；否则，返回false。

**后置条件**<br>
当*this被调用线程锁住，则返回true。

**抛出**
无

**NOTE** 该函数在获取锁时，当函数返回true时，`*this`上对锁的计数会加一。如果当前线程还未获取`*this`上的锁，那么该函数在获取锁时，可能失败(并返回false)，即使没有其他线程持有`*this`上的锁。

####std::recursive_mutex::unlock 成员函数

释放当前线程获取的`std::recursive_mutex`锁。

**声明**
```c++
void unlock();
```

**先决条件**<br>
*this上必须持有一个锁。

**效果**<be>
释放当前线程获取到`*this`上的锁。如果这是`*this`在当前线程上最后一个锁，那么任意等待获取`*this`上的线程，会在该函数调用后解除其中一个线程的阻塞。

**后置条件**<br>
`*this`上锁的计数会在该函数调用后减一。

**抛出**<br>
无

###D.5.3 std::timed_mutex类

`std::timed_mutex`类型在`std::mutex`基本互斥和同步工具的基础上，让锁支持超时。互斥量可以用来保护数据，互斥量上锁必须要调用lok(),try_lock_for(),或try_lock_until()。当有一个线程获取已经获取了锁，那么其他线程想要在获取锁的时候，会在尝试或取锁的时候失败(调用try_lock())或阻塞(调用lock())，或直到想要获取锁可以获取，亦或想要获取的锁超时(调用try_lock_for()或try_lock_until())。在线程调用unlock()对锁进行释放，其他线程才能获取这个锁被获取(不管是调用的哪个函数)。

`std::timed_mutex`符合TimedLockable的需求。

**类型定义**
```c++
class timed_mutex
{
public:
  timed_mutex(timed_mutex const&)=delete;
  timed_mutex& operator=(timed_mutex const&)=delete;

  timed_mutex();
  ~timed_mutex();

  void lock();
  void unlock();
  bool try_lock();

  template<typename Rep,typename Period>
  bool try_lock_for(
      std::chrono::duration<Rep,Period> const& relative_time);

  template<typename Clock,typename Duration>
  bool try_lock_until(
      std::chrono::time_point<Clock,Duration> const& absolute_time);
};
```

####std::timed_mutex 默认构造函数

构造一个`std::timed_mutex`对象。

**声明**
```c++
timed_mutex();
```

**效果**<br>
构造一个`std::timed_mutex`实例。

**后置条件**<br>
新构造一个未上锁的`std::timed_mutex`对象。

**抛出**<br>
当无法创建出新的`std::timed_mutex`实例时，抛出`std::system_error`类型异常。

####std::timed_mutex 析构函数

销毁一个`std::timed_mutex`对象。

**声明**
```c++
~timed_mutex();
```

**先决条件**<br>
*this必须没有上锁。

**效果**<br>
销毁*this。

**抛出**<br>
无

####std::timed_mutex::lock 成员函数

为当前线程获取`std::timed_mutex`上的锁。

**声明**
```c++
void lock();
```

**先决条件**<br>
调用线程不能已经持有*this上的锁。

**效果**<br>
阻塞当前线程，直到获取到*this上的锁。

**后置条件**<br>
*this被调用线程锁住。

**抛出**<br>
当有错误产生，抛出`std::system_error`类型异常。

####std::timed_mutex::try_lock 成员函数

尝试获取为当前线程获取`std::timed_mutex`上的锁。

**声明**
```c++
bool try_lock();
```

**先决条件**<br>
调用线程不能已经持有*this上的锁。

**效果**<br>
尝试获取*this上的锁，当获取失败时，不阻塞调用线程。

**返回**<br>
当锁被调用线程获取，返回true；反之，返回false。

**后置条件**<br>
当函数返回为true，*this则被当前线程锁住。

**抛出**<br>
无

**NOTE** 即使没有线程已获取*this上的锁，函数还是有可能获取不到锁(并返回false)。

####std::timed_mutex::try_lock_for 成员函数

尝试获取为当前线程获取`std::timed_mutex`上的锁。

**声明**
```c++
template<typename Rep,typename Period>
bool try_lock_for(
    std::chrono::duration<Rep,Period> const& relative_time);
```

**先决条件**<br>
调用线程不能已经持有*this上的锁。

**效果**<br>
在指定的relative_time时间内，尝试获取*this上的锁。当relative_time.count()为0或负数，将会立即返回，就像调用try_lock()一样。否则，将会阻塞，直到获取锁或超过给定的relative_time的时间。

**返回**<br>
当锁被调用线程获取，返回true；反之，返回false。

**后置条件**<br>
当函数返回为true，*this则被当前线程锁住。

**抛出**<br>
无

**NOTE** 即使没有线程已获取*this上的锁，函数还是有可能获取不到锁(并返回false)。线程阻塞的时长可能会长于给定的时间。逝去的时间可能是由一个稳定时钟所决定。

####std::timed_mutex::try_lock_until 成员函数

尝试获取为当前线程获取`std::timed_mutex`上的锁。

**声明**
```c++
template<typename Clock,typename Duration>
bool try_lock_until(
    std::chrono::time_point<Clock,Duration> const& absolute_time);
```

**先决条件**<br>
调用线程不能已经持有*this上的锁。

**效果**<br>
在指定的absolute_time时间内，尝试获取*this上的锁。当`absolute_time<=Clock::now()`时，将会立即返回，就像调用try_lock()一样。否则，将会阻塞，直到获取锁或Clock::now()返回的时间等于或超过给定的absolute_time的时间。

**返回**<br>
当锁被调用线程获取，返回true；反之，返回false。

**后置条件**<br>
当函数返回为true，*this则被当前线程锁住。

**抛出**<br>
无

**NOTE** 即使没有线程已获取*this上的锁，函数还是有可能获取不到锁(并返回false)。这里不保证调用函数要阻塞多久，只有在函数返回false后，在Clock::now()返回的时间大于或等于absolute_time时，线程才会接触阻塞。

####std::timed_mutex::unlock 成员函数

将当前线程持有`std::timed_mutex`对象上的锁进行释放。

**声明**
```c++
void unlock();
```

**先决条件**<br>
调用线程已经持有*this上的锁。

**效果**<br>
当前线程释放`*this`上的锁。任一阻塞等待获取`*this`上的线程，将被解除阻塞。

**后置条件**<br>
*this未被调用线程上锁。

**抛出**<br>
无

###D.5.4 std::recursive_timed_mutex类

`std::recursive_timed_mutex`类型在`std::recursive_mutex`提供的互斥和同步工具的基础上，让锁支持超时。互斥量可以用来保护数据，互斥量上锁必须要调用lok(),try_lock_for(),或try_lock_until()。当有一个线程获取已经获取了锁，那么其他线程想要在获取锁的时候，会在尝试或取锁的时候失败(调用try_lock())或阻塞(调用lock())，或直到想要获取锁可以获取，亦或想要获取的锁超时(调用try_lock_for()或try_lock_until())。在线程调用unlock()对锁进行释放，其他线程才能获取这个锁被获取(不管是调用的哪个函数)。

该互斥量是可递归的，所以获取`std::recursive_timed_mutex`锁的线程，可以多次的对该实例上的锁获取。所有的锁将会在调用相关unlock()操作后，可由其他线程获取该实例上的锁。

`std::recursive_timed_mutex`符合TimedLockable的需求。

**类型定义**
```c++
class recursive_timed_mutex
{
public:
  recursive_timed_mutex(recursive_timed_mutex const&)=delete;
  recursive_timed_mutex& operator=(recursive_timed_mutex const&)=delete;

  recursive_timed_mutex();
  ~recursive_timed_mutex();

  void lock();
  void unlock();
  bool try_lock() noexcept;

  template<typename Rep,typename Period>
  bool try_lock_for(
      std::chrono::duration<Rep,Period> const& relative_time);

  template<typename Clock,typename Duration>
  bool try_lock_until(
      std::chrono::time_point<Clock,Duration> const& absolute_time);
};
```

####std::recursive_timed_mutex 默认构造函数

构造一个`std::recursive_timed_mutex`对象。

**声明**
```c++
recursive_timed_mutex();
```

**效果**<br>
构造一个`std::recursive_timed_mutex`实例。

**后置条件**<br>
新构造的`std::recursive_timed_mutex`实例是没有上锁的。

**抛出**<br>
当无法创建一个`std::recursive_timed_mutex`实例时，抛出`std::system_error`类异常。

####std::recursive_timed_mutex 析构函数

析构一个`std::recursive_timed_mutex`对象。

**声明**
```c++
~recursive_timed_mutex();
```

**先决条件**<br>
*this不能上锁。

**效果**<br>
销毁*this。

**抛出**<br>
无

####std::recursive_timed_mutex::lock 成员函数

为当前线程获取`std::recursive_timed_mutex`对象上的锁。

**声明**
```c++
void lock();
```

**先决条件**<br>
*this上的锁不能被线程调用。

**效果**<br>
阻塞当前线程，直到获取*this上的锁。

**后置条件**<br>
`*this`被调用线程锁住。当调用线程已经获取`*this`上的锁，那么锁的计数会再增加1。

**抛出**<br>
当错误出现时，抛出`std::system_error`类型异常。

####std::recursive_timed_mutex::try_lock 成员函数

尝试为当前线程获取`std::recursive_timed_mutex`对象上的锁。

**声明**
```c++
bool try_lock() noexcept;
```

**效果**<br>
尝试获取*this上的锁，当获取失败时，直接不阻塞线程。

**返回**<br>
当调用线程获取了锁，返回true，否则返回false。

**后置条件**<br>
当函数返回true，`*this`会被调用线程锁住。

**抛出**<br>
无

**NOTE** 该函数在获取锁时，当函数返回true时，`*this`上对锁的计数会加一。如果当前线程还未获取`*this`上的锁，那么该函数在获取锁时，可能失败(并返回false)，即使没有其他线程持有`*this`上的锁。

####std::recursive_timed_mutex::try_lock_for 成员函数

尝试为当前线程获取`std::recursive_timed_mutex`对象上的锁。

**声明**
```c++
template<typename Rep,typename Period>
bool try_lock_for(
    std::chrono::duration<Rep,Period> const& relative_time);
```

**效果**<br>
在指定时间relative_time内，尝试为调用线程获取*this上的锁。当relative_time.count()为0或负数时，将会立即返回，就像调用try_lock()一样。否则，调用会阻塞，直到获取相应的锁，或超出了relative_time时限时，调用线程解除阻塞。

**返回**<br>
当调用线程获取了锁，返回true，否则返回false。

**后置条件**<br>
当函数返回true，`*this`会被调用线程锁住。

**抛出**<br>
无

**NOTE** 该函数在获取锁时，当函数返回true时，`*this`上对锁的计数会加一。如果当前线程还未获取`*this`上的锁，那么该函数在获取锁时，可能失败(并返回false)，即使没有其他线程持有`*this`上的锁。等待时间可能要比指定的时间长很多。逝去的时间可能由一个稳定时钟来计算。

####std::recursive_timed_mutex::try_lock_until 成员函数

尝试为当前线程获取`std::recursive_timed_mutex`对象上的锁。

**声明**
```c++
template<typename Clock,typename Duration>
bool try_lock_until(
    std::chrono::time_point<Clock,Duration> const& absolute_time);
```

**效果**<br>
在指定时间absolute_time内，尝试为调用线程获取*this上的锁。当absolute_time<=Clock::now()时，将会立即返回，就像调用try_lock()一样。否则，调用会阻塞，直到获取相应的锁，或Clock::now()返回的时间大于或等于absolute_time时，调用线程解除阻塞。

**返回**<br>
当调用线程获取了锁，返回true，否则返回false。

**后置条件**<br>
当函数返回true，`*this`会被调用线程锁住。

**抛出**<br>
无

**NOTE** 该函数在获取锁时，当函数返回true时，`*this`上对锁的计数会加一。如果当前线程还未获取`*this`上的锁，那么该函数在获取锁时，可能失败(并返回false)，即使没有其他线程持有`*this`上的锁。这里阻塞的时间并不确定，只有当函数返回false，然后Clock::now()返回的时间大于或等于absolute_time时，调用线程将会解除阻塞。

####std::recursive_timed_mutex::unlock 成员函数

释放当前线程获取到的`std::recursive_timed_mutex`上的锁。

**声明**
```c++
void unlock();
```

**效果**<br>
当前线程释放`*this`上的锁。当`*this`上最后一个锁被释放后，任何等待获取`*this`上的锁将会解除阻塞，不过只能解除其中一个线程的阻塞。

**后置条件**<br>
调用线程*this上锁的计数减一。

**抛出**<br>
无

###D.5.5 std::lock_guard类型模板

`std::lock_guard`类型模板为基础锁包装所有权。所要上锁的互斥量类型，由模板参数Mutex来决定，并且必须符合Lockable的需求。指定的互斥量在构造函数中上锁，在析构函数中解锁。这就为互斥量锁部分代码提供了一个简单的方式；当程序运行完成时，阻塞解除，互斥量解锁(无论是执行到最后，还是通过控制流语句break或return，亦或是抛出异常)。

`std::lock_guard`是不可MoveConstructible(移动构造), CopyConstructible(拷贝构造)和CopyAssignable(拷贝赋值)。

**类型定义**
```c++
template <class Mutex>
class lock_guard
{
public:
  typedef Mutex mutex_type;

  explicit lock_guard(mutex_type& m);
  lock_guard(mutex_type& m, adopt_lock_t);
  ~lock_guard();

  lock_guard(lock_guard const& ) = delete;
  lock_guard& operator=(lock_guard const& ) = delete;
};
```

####std::lock_guard 自动上锁的构造函数

使用互斥量构造一个`std::lock_guard`实例。

**声明**
```c++
explicit lock_guard(mutex_type& m);
```

**效果**<br>
通过引用提供的互斥量，构造一个新的`std::lock_guard`实例，并调用m.lock()。

**抛出**<br>
m.lock()抛出的任何异常。

**后置条件**<br>
*this拥有m上的锁。

####std::lock_guard 获取锁的构造函数

使用已提供互斥量上的锁，构造一个`std::lock_guard`实例。

**声明**
```c++
lock_guard(mutex_type& m,std::adopt_lock_t);
```

**先决条件**<br>
调用线程必须拥有m上的锁。

**效果**<br>
调用线程通过引用提供的互斥量，以及获取m上锁的所有权，来构造一个新的`std::lock_guard`实例。

**抛出**<br>
无

**后置条件**<br>
*this拥有m上的锁。

####std::lock_guard 析构函数

销毁一个`std::lock_guard`实例，并且解锁相关互斥量。

**声明**
```c++
~lock_guard();
```

**效果**<br>
当*this被创建后，调用m.unlock()。

**抛出**<br>
无

###D.5.6 std::unique_lock类型模板

`std::unique_lock`类型模板相较`std::loc_guard`提供了更通用的所有权包装器。上锁的互斥量可由模板参数Mutex提供，这个类型必须满足BasicLockable的需求。虽然，通常情况下，制定的互斥量会在构造的时候上锁，析构的时候解锁，但是附加的构造函数和成员函数提供灵活的功能。互斥量上锁，意味着对操作同一段代码的线程进行阻塞；当互斥量解锁，就意味着阻塞解除(不论是裕兴到最后，还是使用控制语句break和return，亦或是抛出异常)。`std::condition_variable`的邓丹函数是需要`std::unique_lock<std::mutex>`实例的，并且所有`std::unique_lock`实例都适用于`std::conditin_variable_any`等待函数的Lockable参数。

当提供的Mutex类型符合Lockable的需求，那么`std::unique_lock<Mutex>`也是符合Lockable的需求。此外，如果提供的Mutex类型符合TimedLockable的需求，那么`std::unique_lock<Mutex>`也符合TimedLockable的需求。

`std::unique_lock`实例是MoveConstructible(移动构造)和MoveAssignable(移动赋值)，但是不能CopyConstructible(拷贝构造)和CopyAssignable(拷贝赋值)。

**类型定义**
```c++
template <class Mutex>
class unique_lock
{
public:
  typedef Mutex mutex_type;

  unique_lock() noexcept;
  explicit unique_lock(mutex_type& m);
  unique_lock(mutex_type& m, adopt_lock_t);
  unique_lock(mutex_type& m, defer_lock_t) noexcept;
  unique_lock(mutex_type& m, try_to_lock_t);

  template<typename Clock,typename Duration>
  unique_lock(
      mutex_type& m,
      std::chrono::time_point<Clock,Duration> const& absolute_time);

  template<typename Rep,typename Period>
      unique_lock(
      mutex_type& m,
      std::chrono::duration<Rep,Period> const& relative_time);

  ~unique_lock();

  unique_lock(unique_lock const& ) = delete;
  unique_lock& operator=(unique_lock const& ) = delete;

  unique_lock(unique_lock&& );
  unique_lock& operator=(unique_lock&& );

  void swap(unique_lock& other) noexcept;

  void lock();
  bool try_lock();
  template<typename Rep, typename Period>
  bool try_lock_for(
      std::chrono::duration<Rep,Period> const& relative_time);
  template<typename Clock, typename Duration>
  bool try_lock_until(
      std::chrono::time_point<Clock,Duration> const& absolute_time);
  void unlock();

  explicit operator bool() const noexcept;
  bool owns_lock() const noexcept;
  Mutex* mutex() const noexcept;
  Mutex* release() noexcept;
};
```

####std::unique_lock 默认构造函数

不使用相关互斥量，构造一个`std::unique_lock`实例。

**声明**
```c++
unique_lock() noexcept;
```

**效果**<br>
构造一个`std::unique_lock`实例，这个新构造的实例没有相关互斥量。

**后置条件**<br>
this->mutex()==NULL, this->owns_lock()==false.

####std::unique_lock 自动上锁的构造函数

使用相关互斥量，构造一个`std::unique_lock`实例。

**声明**
```c++
explicit unique_lock(mutex_type& m);
```

**效果**<br>
通过提供的互斥量，构造一个`std::unique_lock`实例，且调用m.lock()。

**抛出**<br>
m.lock()抛出的任何异常。

**后置条件**<br>
this->owns_lock()==true, this->mutex()==&m.

####std::unique_lock 获取锁的构造函数

使用相关互斥量和持有的锁，构造一个`std::unique_lock`实例。

**声明**
```c++
unique_lock(mutex_type& m,std::adopt_lock_t);
```

**先决条件**<br>
调用线程必须持有m上的锁。

**效果**<br>
通过提供的互斥量和已经拥有m上的锁，构造一个`std::unique_lock`实例。

**抛出**<br>
无

**后置条件**<br>
this->owns_lock()==true, this->mutex()==&m.

####std::unique_lock 递延锁的构造函数

使用相关互斥量和非持有的锁，构造一个`std::unique_lock`实例。

**声明**
```c++
unique_lock(mutex_type& m,std::defer_lock_t) noexcept;
```

**效果**<br>
构造的`std::unique_lock`实例引用了提供的互斥量。

**抛出**<br>
无

**后置条件**<br>
this->owns_lock()==false, this->mutex()==&m.

####std::unique_lock 尝试获取锁的构造函数

使用提供的互斥量，并尝试从互斥量上获取锁，从而构造一个`std::unique_lock`实例。

**声明**
```c++
unique_lock(mutex_type& m,std::try_to_lock_t);
```

**先决条件**<br>
使`std::unique_lock`实例化的Mutex类型，必须符合Loackable的需求。

**效果**<br>
构造的`std::unique_lock`实例引用了提供的互斥量，且调用m.try_lock()。

**抛出**<br>
无

**后置条件**<br>
this->owns_lock()将返回m.try_lock()的结果，且this->mutex()==&m。

####std::unique_lock 在给定时长内尝试获取锁的构造函数

使用提供的互斥量，并尝试从互斥量上获取锁，从而构造一个`std::unique_lock`实例。

**声明**
```c++
template<typename Rep,typename Period>
unique_lock(
    mutex_type& m,
    std::chrono::duration<Rep,Period> const& relative_time);
```

**先决条件**<br>
使`std::unique_lock`实例化的Mutex类型，必须符合TimedLockable的需求。

**效果**<br>
构造的`std::unique_lock`实例引用了提供的互斥量，且调用m.try_lock_for(relative_time)。

**抛出**<br>
无

**后置条件**<br>
this->owns_lock()将返回m.try_lock_for()的结果，且this->mutex()==&m。

####std::unique_lock 在给定时间点内尝试获取锁的构造函数

使用提供的互斥量，并尝试从互斥量上获取锁，从而构造一个`std::unique_lock`实例。

**声明**
```c++
template<typename Clock,typename Duration>
unique_lock(
    mutex_type& m,
    std::chrono::time_point<Clock,Duration> const& absolute_time);
```

**先决条件**<br>
使`std::unique_lock`实例化的Mutex类型，必须符合TimedLockable的需求。

**效果**<br>
构造的`std::unique_lock`实例引用了提供的互斥量，且调用m.try_lock_until(absolute_time)。

**抛出**<br>
无

**后置条件**<br>
this->owns_lock()将返回m.try_lock_until()的结果，且this->mutex()==&m。

####std::unique_lock 移动构造函数

将一个已经构造`std::unique_lock`实例的所有权，转移到新的`std::unique_lock`实例上去。

**声明**
```c++
unique_lock(unique_lock&& other) noexcept;
```

**先决条件**<br>
使`std::unique_lock`实例化的Mutex类型，必须符合TimedLockable的需求。

**效果**<br>
构造的`std::unique_lock`实例。当other在函数调用的时候拥有互斥量上的锁，那么该锁的所有权将被转移到新构建的`std::unique_lock`对象当中去。

**后置条件**<br>
对于新构建的`std::unique_lock`对象x，x.mutex等价与在构造函数调用前的other.mutex()，并且x.owns_lock()等价于函数调用前的other.owns_lock()。在调用函数后，other.mutex()==NULL，other.owns_lock()=false。

**抛出**<br>
无

**NOTE** `std::unique_lock`对象是不可CopyConstructible(拷贝构造)，所以这里没有拷贝构造函数，只有移动构造函数。

####std::unique_lock 移动赋值操作

将一个已经构造`std::unique_lock`实例的所有权，转移到新的`std::unique_lock`实例上去。

**声明**
```c++
unique_lock& operator=(unique_lock&& other) noexcept;
```

**效果**<br>
当this->owns_lock()返回true时，调用this->unlock()。如果other拥有mutex上的锁，那么这个所将归*this所有。

**后置条件**<br>
this->mutex()等于在为进行赋值前的other.mutex()，并且this->owns_lock()的值与进行赋值操作前的other.owns_lock()相等。other.mutex()==NULL, other.owns_lock()==false。

**抛出**<br>
无

**NOTE** `std::unique_lock`对象是不可CopyAssignable(拷贝赋值)，所以这里没有拷贝赋值函数，只有移动赋值函数。

####std::unique_lock 析构函数

销毁一个`std::unique_lock`实例，如果该实例拥有锁，那么会将相关互斥量进行解锁。

**声明**
```c++
~unique_lock();
```

**效果**<br>
当this->owns_lock()返回true时，调用this->mutex()->unlock()。

**抛出**<br>
无

####std::unique_lock::swap 成员函数

交换`std::unique_lock`实例中相关的所有权。

**声明**
```c++
void swap(unique_lock& other) noexcept;
```

**效果**<br>
如果other在调用该函数前拥有互斥量上的锁，那么这个锁将归`*this`所有。如果`*this`在调用哎函数前拥有互斥量上的锁，那么这个锁将归other所有。

**抛出**<br>
无

####std::unique_lock 上非成员函数swap

交换`std::unique_lock`实例中相关的所有权。

**声明**
```c++
void swap(unique_lock& lhs,unique_lock& rhs) noexcept;
```

**效果**<br>
lhs.swap(rhs)

**抛出**<br>
无

####std::unique_lock::lock 成员函数

获取与*this相关互斥量上的锁。

**声明**
```c++
void lock();
```

**先决条件**<br>
this->mutex()!=NULL, this->owns_lock()==false.

**效果**<br>
调用this->mutex()->lock()。

**抛出**<br>
抛出任何this->mutex()->lock()所抛出的异常。当this->mutex()==NULL，抛出`std::sytem_error`类型异常，错误码为`std::errc::operation_not_permitted`。当this->owns_lock()==true时，抛出`std::system_error`，错误码为`std::errc::resource_deadlock_would_occur`。

**后置条件**<br>
this->owns_lock()==true。

####std::unique_lock::try_lock 成员函数

尝试获取与*this相关互斥量上的锁。

**声明**
```c++
bool try_lock();
```

**先决条件**<br>
`std::unique_lock`实例化说是用的Mutex类型，必须满足Lockable需求。this->mutex()!=NULL, this->owns_lock()==false。

**效果**<br>
调用this->mutex()->try_lock()。

**抛出**<br>
抛出任何this->mutex()->try_lock()所抛出的异常。当this->mutex()==NULL，抛出`std::sytem_error`类型异常，错误码为`std::errc::operation_not_permitted`。当this->owns_lock()==true时，抛出`std::system_error`，错误码为`std::errc::resource_deadlock_would_occur`。

**后置条件**<br>
当函数返回true时，this->ows_lock()==true，否则this->owns_lock()==false。

####std::unique_lock::unlock 成员函数

释放与*this相关互斥量上的锁。

**声明**
```c++
void unlock();
```

**先决条件**<br>
this->mutex()!=NULL, this->owns_lock()==true。

**抛出**<br>
抛出任何this->mutex()->unlock()所抛出的异常。当this->owns_lock()==false时，抛出`std::system_error`，错误码为`std::errc::operation_not_permitted`。

**后置条件**<br>
this->owns_lock()==false。

####std::unique_lock::try_lock_for 成员函数

在指定时间内尝试获取与*this相关互斥量上的锁。

**声明**
```c++
template<typename Rep, typename Period>
bool try_lock_for(
    std::chrono::duration<Rep,Period> const& relative_time);
```

**先决条件**<br>
`std::unique_lock`实例化说是用的Mutex类型，必须满足TimedLockable需求。this->mutex()!=NULL, this->owns_lock()==false。

**效果**<br>
调用this->mutex()->try_lock_for(relative_time)。

**返回**<br>
当this->mutex()->try_lock_for()返回true，返回true，否则返回false。

**抛出**<br>
抛出任何this->mutex()->try_lock_for()所抛出的异常。当this->mutex()==NULL，抛出`std::sytem_error`类型异常，错误码为`std::errc::operation_not_permitted`。当this->owns_lock()==true时，抛出`std::system_error`，错误码为`std::errc::resource_deadlock_would_occur`。

**后置条件**<br>
当函数返回true时，this->ows_lock()==true，否则this->owns_lock()==false。

####std::unique_lock::try_lock_until 成员函数

在指定时间点尝试获取与*this相关互斥量上的锁。

**声明**
```c++
template<typename Clock, typename Duration>
bool try_lock_until(
    std::chrono::time_point<Clock,Duration> const& absolute_time);
```

**先决条件**<br>
`std::unique_lock`实例化说是用的Mutex类型，必须满足TimedLockable需求。this->mutex()!=NULL, this->owns_lock()==false。

**效果**<br>
调用this->mutex()->try_lock_until(absolute_time)。

**返回**<br>
当this->mutex()->try_lock_for()返回true，返回true，否则返回false。

**抛出**<br>
抛出任何this->mutex()->try_lock_for()所抛出的异常。当this->mutex()==NULL，抛出`std::sytem_error`类型异常，错误码为`std::errc::operation_not_permitted`。当this->owns_lock()==true时，抛出`std::system_error`，错误码为`std::errc::resource_deadlock_would_occur`。

**后置条件**<br>
当函数返回true时，this->ows_lock()==true，否则this->owns_lock()==false。

####std::unique_lock::operator bool成员函数

检查*this是否拥有一个互斥量上的锁。

**声明**
```c++
explicit operator bool() const noexcept;
```

**返回**<br>
this->owns_lock()

**抛出**<br>
无

**NOTE** 这是一个explicit转换操作，所以当这样的操作在上下文中只能被隐式的调用，所返回的结果需要被当做一个布尔量进行使用，而非仅仅作为整型数0或1。

####std::unique_lock::owns_lock 成员函数

检查*this是否拥有一个互斥量上的锁。

**声明**
```c++
bool owns_lock() const noexcept;
```

**返回**<br>
当*this持有一个互斥量的锁，返回true；否则，返回false。

**抛出**<br>
无

####std::unique_lock::mutex 成员函数

当*this具有相关互斥量时，返回这个互斥量

**声明**
```c++
mutex_type* mutex() const noexcept;
```

**返回**<br>
当*this有相关互斥量，则返回该互斥量；否则，返回NULL。

**抛出**<br>
无

####std::unique_lock::release 成员函数

当*this具有相关互斥量时，返回这个互斥量，并将这个互斥量进行释放。

**声明**
```c++
mutex_type* release() noexcept;
```

**效果**<br>
将*this与相关的互斥量之间的关系解除，同时解除所有持有锁的所有权。

**返回**<br>
返回与*this相关的互斥量指针，如果没有相关的互斥量，则返回NULL。

**后置条件**<br>
this->mutex()==NULL, this->owns_lock()==false。

**抛出**<br>
无

**NOTE** 如果this->owns_lock()在调用该函数前返回true，那么调用者则有责任里解除互斥量上的锁。

###D.5.7 std::lock函数模板

`std::lock`函数模板提供同时锁住多个互斥量的功能，且不会有因改变锁的一致性而导致的死锁。

**声明**
```c++
template<typename LockableType1,typename... LockableType2>
void lock(LockableType1& m1,LockableType2& m2...);
```

**先决条件**<br>
提供的可锁对象LockableType1, LockableType2...，需要满足Lockable的需求。

**效果**<br>
使用未指定顺序调用lock(),try_lock()获取每个可锁对象(m1, m2...)上的锁，还有unlock()成员来避免这个类型陷入死锁。

**后置条件**<br>
当前线程拥有提供的所有可锁对象上的锁。

**抛出**<br>
任何lock(), try_lock()和unlock()抛出的异常。

**NOTE** 如果一个异常由`std::lock`所传播开来，当可锁对象上有锁被lock()或try_lock()获取，那么unlock()会使用在这些可锁对象上。

###D.5.8 std::try_lock函数模板

`std::try_lock`函数模板允许尝试获取一组可锁对象上的锁，所以要不全部获取，要不一个都不获取。

**声明**
```c++
template<typename LockableType1,typename... LockableType2>
int try_lock(LockableType1& m1,LockableType2& m2...);
```

**先决条件**<br>
提供的可锁对象LockableType1, LockableType2...，需要满足Lockable的需求。

**效果**<br>
使用try_lock()尝试从提供的可锁对象m1,m2...上逐个获取锁。当锁在之前获取过，但被当前线程使用unlock()对相关可锁对象进行了释放后，try_lock()会返回false或抛出一个异常。

**返回**<br>
当所有锁都已获取(每个互斥量调用try_lock()返回true)，则返回-1，否则返回以0为基数的数字，其值为调用try_lock()返回false的个数。

**后置条件**<br>
当函数返回-1，当前线程获取从每个可锁对象上都获取一个锁。否则，通过该调用获取的任何锁都将被释放。

**抛出**<br>
try_lock()抛出的任何异常。

**NOTE** 如果一个异常由`std::try_lock`所传播开来，则通过try_lock()获取锁对象，将会调用unlock()解除对锁的持有。

###D.5.9 std::once_flag类

###D.5.10 std::call_once函数模板

##D.6 &lt;ratio&gt;头文件

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

##D.7 &lt;thread&gt;头文件

###D.7.1 std::thread类

###D.7.2 this_thread命名空间