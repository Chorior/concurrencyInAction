---
title:      "C++ 多线程"
subtitle:   "concurrency in action"
date:       2017-04-24 20:20:00 +0800
header-img: "img/stock-photo-4.jpg"
tags:
    - C++
    - thread
---

本文知识来自[C++ Concurrency In Action](https://chenxiaowei.gitbooks.io/cpp_concurrency_in_action/content/)，介绍有关C++11多线程(multithreading)相关的知识。

#   本文结构

*   [多线程概述](#multithreading_overview)
*   [多线程管理](#managing_threads)
*   [多线程识别](#thread_identification)
*   [线程间共享数据](#sharing_data_between_threads)
*   [多线程同步](#synchronizing_thread)

<h2 id="multithreading_overview">多线程概述</h2>

### 基本概念

并发：**同一时间内发生两个或更多独立的活动**；

任务切换: 每个任务交织进行,但切换会有时间开销；

硬件并发(hardware concurrency): 真正的并行多个任务。

**即便是具有真正硬件并发的系统，有时候也会需要任务切换**；

### 使用并发的两种方法

将应用程序分为多个独立的进程,它们在同一时刻运行

*   缺点

    *   进程间通信设置复杂，速度慢，因为系统在进程间提供了很多保护措施用以保护数据；
    *   运行多个进程需要固定开销：需要时间启动进程，操作系统需要内部资源来管理进程，等等；

*   优点

    *   更容易编写安全(safe)的并发代码；
    *   可以使用远程连接(可能需要联网)，在不同的机器上运行独立的进程；

在单个进程中运行多个线程

*   缺点

    *   进程中的所有线程都共享地址空间；        
    *   如果数据要被多个线程访问，那么必须确保每个线程所访问到的数据是一致的；

*   优点

    *   使用多线程相关的开销远远小于使用多个进程；
    *   全局变量仍然是全局的，指针、对象的引用或数据可以在线程之间传递。

**C++标准并未对进程间通信提供任何原生支持**。

### 使用并发的两种原因

分离关注点(separation of concerns(SOC))

*   将相关的代码与无关的代码分离，使程序更容易理解和测试，减少出错的可能性；
*   独立的线程通常用来执行那些必须在后台持续运行的任务，如socket服务器。

提升性能

*   任务并行(task parallelism)：将一个单个任务分成几部分，且各自并行运行，从而降低总运行时间；
*   数据并行(data parallelism)：同时对多组数据执行相同的操作。

### 并发的缺点

*   如果在线程上的任务完成得很快，那么任务实际执行的时间要比启动线程的时间小很多；
*   运行太多的线程会耗尽进程的可用内存或地址空间(**可以使用线程池来限定线程数量**)；
*   线程数越多，操作系统就需要做越多的上下文切换；
*   **使用并发可能使代码复杂化、更难理解，并且更容易出错**。

**当并发收益小于实际开发和维护的成本时，不要使用并发**。

**在绝大多数情况下，额外增加的复杂性和出错几率都远大于性能的小幅提升带来的收益**。

### Hello, world

```c++
#include <iostream>
#include <thread>

void hello() { 
    std::cout << "Hello Concurrent World!\n"; 
}

int main()
{
    std::thread t(hello);
    t.join();
}
```

*   g++编译时须加上`-pthread -std=c++11`；
*   管理线程的函数和类在`<thread>`中声明，而保护共享数据的函数和类在其他头文件中声明；
*   初始线程始于`main()`，新线程始于`hello()`；
*   `join()`使得初始线程等待新线程结束后，才能运行下面的语句或结束自己的线程；
*   该示例使用多线程并没有带来任何收益；
*   **使用多线程并不复杂，复杂的是如何设计代码以实现其预期的行为**。

<h2 id="managing_threads">多线程管理</h2>

**每个程序至少有一个线程：执行main()函数的线程**，其余线程有其各自的入口函数。线程与原始线程(以main()为入口函数的线程)同时运行。如同main()函数执行完会退出一样，当线程执行完入口函数后，线程也会退出。

```c++
class thread
{	// class for observing and managing threads
public:
	class id;

	typedef void *native_handle_type;

	thread() _NOEXCEPT
	{	// construct with no thread
		_Thr_set_null(_Thr);
	}


	template<class _Fn,
		class... _Args,
		class = typename enable_if<
		!is_same<typename decay<_Fn>::type, thread>::value>::type>
		explicit thread(_Fn&& _Fx, _Args&&... _Ax)
	{	// construct with _Fx(_Ax...)
		_Launch(&_Thr,
			_STD make_unique<tuple<decay_t<_Fn>, decay_t<_Args>...> >(
				_STD forward<_Fn>(_Fx), _STD forward<_Args>(_Ax)...));
	}


	~thread() _NOEXCEPT
	{	// clean up
		if (joinable())
			_XSTD terminate();
	}

	thread(thread&& _Other) _NOEXCEPT
		: _Thr(_Other._Thr)
	{	// move from _Other
		_Thr_set_null(_Other._Thr);
	}

	thread& operator=(thread&& _Other) _NOEXCEPT
	{	// move from _Other
		return (_Move_thread(_Other));
	}

	thread(const thread&) = delete;
	thread& operator=(const thread&) = delete;

	void swap(thread& _Other) _NOEXCEPT
	{	// swap with _Other
		_STD swap(_Thr, _Other._Thr);
	}

	bool joinable() const _NOEXCEPT
	{	// return true if this thread can be joined
		return (!_Thr_is_null(_Thr));
	}

	void join()
	{   // join thread
		if (!joinable())
			_Throw_Cpp_error(_INVALID_ARGUMENT);
		const bool _Is_null = _Thr_is_null(_Thr);	// Avoid Clang -Wparentheses-equality
		if (_Is_null)
			_Throw_Cpp_error(_INVALID_ARGUMENT);
		if (get_id() == _STD this_thread::get_id())
			_Throw_Cpp_error(_RESOURCE_DEADLOCK_WOULD_OCCUR);
		if (_Thrd_join(_Thr, 0) != _Thrd_success)
			_Throw_Cpp_error(_NO_SUCH_PROCESS);
		_Thr_set_null(_Thr);
	}

	void detach()
	{	// detach thread
		if (!joinable())
			_Throw_Cpp_error(_INVALID_ARGUMENT);
		_Thrd_detachX(_Thr);
		_Thr_set_null(_Thr);
	}

	id get_id() const _NOEXCEPT
	{	// return id for this thread
		return (_Thr_val(_Thr));
	}

	static unsigned int hardware_concurrency() _NOEXCEPT
	{	// return number of hardware thread contexts
		return (_Thrd_hardware_concurrency());
	}

	native_handle_type native_handle()
	{	// return Win32 HANDLE as void *
		return (_Thr._Hnd);
	}
```

通过查看thread类的公有成员，我们得知：

*   thread类包含三个构造函数：一个默认构造函数(什么都不做)、一个接受可调用对象及其参数的explicit构造函数(参数可能没有，这时相当于转换构造函数，需要定义为explicit)、和一个移动构造函数；
*   析构函数会在thread对象销毁时自动调用，如果销毁时thread对象还是joinable，那么程序会调用terminate()终止进程；
*   thread类没有拷贝操作，只有移动操作，即**thread对象是可移动不可拷贝的**，这保证了在同一时间点，一个thread实例只能关联一个执行线程；
*   swap函数用来交换两个thread对象管理的线程；
*   joinable函数用来判断该thread对象是否是可加入的；
*   join函数使得该thread对象管理的线程加入到原始线程，**只能使用一次**，并使joinable为false；
*   detach函数使得该thread对象管理的线程与原始线程分离，独立运行，并使joinable为false；
*   `get_id`返回线程标识；
*   `hardware_concurrency`返回能同时并发在一个程序中的线程数量，当系统信息无法获取时，函数也会返回0。

**调用`join()`清理了与线程相关的存储部分，所以该thread对象不再与任何线程相关联直到重新赋值** 。

detach线程又称守护线程(daemon threads)，**C++运行库保证，当detach线程退出时，相关资源的能够正确回收，后台线程的归属和控制C++运行库都会处理**。

<h2 id="start_a_thread">线程启动</h2>

**使用C++线程库开启一个线程通常归结为构造一个`std::thread`类对象**。

由于thread类只有一个有用的构造函数，所以只能使用可调用对象来构造thread对象。

可调用对象包括：

*   函数
*   函数指针
*   lambda表达式
*   bind创建的对象
*   重载了函数调用符的类

**如果参数需要转换才能匹配函数,最好使用显式转换**,因为默认转换也许在没有转换成功之前住线程就结束了；**如果线程参数是引用类型,传递参数时,一定要使用`std::ref(arg)`**；**如果线程函数是成员函数，那么需要传递一个合适的对象实例指针,后跟成员函数参数**。

根据析构函数，可以知道：在构造一个thread对象之后，需要决定调用`join()`或是`detach()`。如果要调用`detach()`，那么需要保证线程结束之前，可访问的数据的有效性，举个栗子，detach的线程如果包含原始线程的局部变量的指针或引用，就很大概率会出现问题。

**detach的线程通常将要用的数据全部复制到自己的线程中**。

```c++
// 线程启动示例
#include <iostream>
#include <cstdlib>
#include <thread>
#include <string>

using namespace std;

void add(float a, float b) { cout << a + b << endl; }
void add(int a, int b) { cout << "add: " << a + b << endl; }
void divide(int a, int b) { cout << a << "/" << b << " = " << a / b << endl; }

struct mod
{
	void operator()(int a, int b) { cout << "mod: " << a % b << endl; }
};

struct noargs
{
	void operator()() { cout << "voidreturn() called.\n"; }
};

struct test
{
	void do_something(string &str) { cout << str << endl; }
};

int main()
{
	auto func_ptr = static_cast<void(*)(int, int)>(add);
	auto lambda = [](int a, int b) { cout << "lambda: " << a * b << endl; };
	auto obj_bind = std::bind(divide, placeholders::_2, placeholders::_1);

	// 这里都用了移动赋值运算符
	auto t0 = thread(divide, 4, 3);
	auto t1 = thread(func_ptr, 4, 3);
	auto t2 = thread(lambda, 4, 3);
	auto t3 = thread(obj_bind, 4, 3);
	auto t4 = thread(mod(), 4, 3);
	auto t5 = thread(std::plus<int>(), 4, 3);
	auto t6 = thread((noargs()));
	// auto t6 = thread{noargs()}; // 正确
	// thread t6(noargs());        // 会被解析为函数声明！	
	string str{ "hahaha" };
	auto t7 = thread(&test::do_something, &test(), std::ref(str));

	t0.join();
	t1.join();
	t2.join();
	t3.join();
	t4.join();
	t5.join();
	t6.join();
	t7.join();

	getchar();
	return EXIT_SUCCESS;
}
```

由于`std::cout`是共用的标准输出，所以会造成竞争，结果打印出来的结果可能会很乱：

```text
4/lambda: 123hahaha
voidreturn() called.
add: 7
 = 1
3/4 = 0

mod: 1


```

如果在thread对象join或者detach之前，原始线程发生了异常，那么对象会使用栈的方式对对象进行销毁，这时因为thread对象还是joinable的，所以销毁thread对象会调用`terminate()`。为了避免这样的情况，程序员通常使用两种方式来解决这个问题：

*   使用异常处理，try-catch；
*   构造一个类，在析构函数里调用join或detach。

```c++
// 异常处理
std::thread t(my_func);
try
{
    do_something_in_current_thread();
}
catch(...)
{
    t.join();  
    throw;
}
t.join();

// 构造类
class scoped_thread
{
	std::thread t;
public:
	explicit scoped_thread(std::thread t_) :
		t(std::move(t_))
	{
		if (!t.joinable())
			throw std::logic_error("No thread");
	}
	~scoped_thread()
	{
		t.join();
	}
	scoped_thread(scoped_thread const&) = delete;
	scoped_thread& operator=(scoped_thread const&) = delete;
};

void f()
{  
    std::thread t(my_func);
    scoped_thread g(std::move(t));
    do_something_in_current_thread();
} 
```

<h2 id="thread_identification">多线程识别</h2>

线程标识类型是`std::thread::id`，这个值可以通过两种方式进行检索：

*   成员函数`std::thread::get_id()`，当没有线程与该thread对象关联时，此函数返回0；
*   命名空间函数`std::this_thread::get_id()`；

如果两个对象的`std::thread::id`相等，那它们就是同一个线程，或者都“没有线程”。如果不等，那么就代表了两个不同线程，或者一个有线程，另一没有。

```c++
// std::thread::id 支持的各种操作
inline bool operator==(thread::id _Left, thread::id _Right) _NOEXCEPT
	{	// return true if _Left and _Right identify the same thread
	return (_Left._Id == _Right._Id);
	}

inline bool operator!=(thread::id _Left, thread::id _Right) _NOEXCEPT
	{	// return true if _Left and _Right do not identify the same thread
	return (!(_Left == _Right));
	}

inline bool operator<(thread::id _Left, thread::id _Right) _NOEXCEPT
	{	// return true if _Left precedes _Right
	return (_Left._Id < _Right._Id);
	}

inline bool operator<=(thread::id _Left, thread::id _Right) _NOEXCEPT
	{	// return true if _Left precedes or equals _Right
	return (!(_Right < _Left));
	}

inline bool operator>(thread::id _Left, thread::id _Right) _NOEXCEPT
	{	// return true if _Left follows _Right
	return (_Right < _Left);
	}

inline bool operator>=(thread::id _Left, thread::id _Right) _NOEXCEPT
	{	// return true if _Left follows or equals _Right
	return (!(_Left < _Right));
	}

template<class _Ch,
	class _Tr>
	basic_ostream<_Ch, _Tr>& operator<<(
		basic_ostream<_Ch, _Tr>& _Str,
		thread::id _Id)
	{	// insert id into stream
	return (_Id._To_text(_Str));
	}

	// TEMPLATE STRUCT SPECIALIZATION hash
template<>
	struct hash<thread::id>
	{	// hash functor for thread::id
	typedef thread::id argument_type;
	typedef size_t result_type;

	size_t operator()(const argument_type& _Keyval) const
		{	// hash _Keyval to size_t value by pseudorandomizing transform
		return (_Keyval._Hash_id());
		}
	};
```

查看`std::thread::id`源代码，发现其支持`==,!=,<,<=,>,>=,<<`运算符，还定义了hash模板的特例化版本，所以`std::thread::id`支持各种算法和无序容器，甚至可以用来作为键值。

<h2 id="sharing_data_between_threads">线程间共享数据</h2>

条件竞争(race condition)：当一个线程A正在修改共享数据时，另一个线程B却在使用这个共享数据，这时B访问到的数据可能不是正确的数据，这种情况称为条件竞争。

数据竞争(data race)：一种特殊的条件竞争；并发的去修改一个独立对象。

多线程的一个关键优点(key benefit)是可以简单的直接共享数据，但如果有多个线程拥有修改共享数据的权限，那么就会很容易出现数据竞争(data race)。

### 使用mutex

**C++标准保护共享数据最基本的技巧是使用互斥量(mutex)**：当访问共享数据前，使用互斥量将相关数据锁住，再当访问结束后，再将数据解锁。线程库需要保证，当一个线程使用特定互斥量锁住共享数据时，其他的线程想要访问锁住的数据，都必须等到之前那个线程对数据进行解锁后，才能进行访问。

在C++中使用互斥量

*	创建互斥量：建造一个`std::mutex`的实例；
*	锁住互斥量：调用成员函数`lock()`；
*	释放互斥量：调用成员函数`unlock()`；
*	由于`lock()`与`unlock()`必须配对，就像new和delete一样，所以为了方便和异常处理，C++标准库也专门提供了一个模板类`std::lock_guard`，其在构造时lock互斥量,析构时unlock互斥量。

`std::mutex`和`std::lock_guard`定义于头文件`<mutex>`中。

```c++
// std::mutex 使用示例
#include <iostream>
#include <cstdlib>
#include <thread>
#include <mutex>
#include <set>

using namespace std;

class Example{
private:
	set<int> mySet;
	mutex myMutex;
public:
	Example() = default;
	~Example() = default;
	Example(const Example&) = delete;
	Example& operator=(const Example&) = delete;

	void add_to_set(int val) {
		lock_guard<mutex> lg(myMutex);
		mySet.emplace(val);
	}

	bool set_contains(int val) {
		lock_guard<mutex> lg(myMutex);
		return mySet.find(val) != mySet.cend();
	}

	void print_set() {
		lock_guard<mutex> lg(myMutex);
		for (auto &i : mySet) {
			cout << i << " ";
		}
	}
};

void thread1(Example &e) {
	for (int i = 0; i < 10; ++i) {
		e.add_to_set(i);
	}
}

void thread2(Example &e) {
	for (int i = 0; i < 10; ++i) {
		cout << e.set_contains(i);
	}
	cout << endl;
}

int main()
{		
	Example e;

	auto t1 = thread(thread1, std::ref(e));
	auto t2 = thread(thread2, std::ref(e));
	t1.join();
	t2.join();

	e.print_set();

	getchar();
	return EXIT_SUCCESS;
}
```

示例结果如下：

```text
1111111111
0 1 2 3 4 5 6 7 8 9
```

**具有访问能力的指针或引用可以访问(并可能修改)被保护的数据，而不会被互斥锁限制**，所以接口的设计一定要确保互斥量能锁住任何对保护数据的访问，并且不留后门。

**切勿将受保护数据的指针或引用传递到互斥锁作用域之外，无论是函数返回值，还是存储在外部可见内存，亦或是以参数的形式传递到用户提供的函数中去**。

```c++
// 错误示例
// 假设Example定义了set<int>* get_set(){return &mySet;} 
set<int> *is = e.get_set();
auto t1 = thread(thread1, std::ref(e));
auto t2 = thread(thread2, std::ref(e));
is->erase(2);
t1.join();
t2.join();
```

结果可能跟最初的意图不太一样：

```text
1101111111
0 1 3 4 5 6 7 8 9
```

### 与mutex相关的接口设计

假设有一个stack，它的所有操作(push,top,pop等)都使用了mutex进行保护，但是下面的代码在并发的情况下依然会出现错误：

```c++
stack<int> s;
if (! s.empty()){
	int const value = s.top(); 
	s.pop();
	do_something(value);
}
```

在调用`empty()`和`top()`之间，或者调用`top()`与`pop()`之间，可能有来自另一个线程的`pop()`调用并删除了最后一个元素。这是接口设计造成的条件竞争(race condition)。

之所以将`top()`和`pop()`分为两部分(好像java就是合成一个函数的)，是为了防止在top发生异常时，保证数据没有丢失。但这造成了条件竞争。

解决这个问题的方案就是将`top()`和`pop()`合成一个函数，在pop之前就将数据传递出去(参数引用或指针返回)，这时就算发生异常也不会造成数据丢失：

```c++
// 一个线程安全的栈的简单设计
#include <exception>
#include <memory>
#include <mutex>
#include <stack>

struct empty_stack : std::exception
{
	// 老版本使用throw()代替noexcept
	const char* what() const noexcept {
		return "empty stack!";
	};
};

template<typename T>
class threadsafe_stack
{
private:
	std::stack<T> data;
	mutable std::mutex m;

public:
	threadsafe_stack()
		: data(std::stack<T>()) {}

	threadsafe_stack(const threadsafe_stack& other)
	{
		std::lock_guard<std::mutex> lock(other.m);
		data = other.data; // 在构造函数体中执行拷贝，而非成员初始化
	}

	// 栈不能直接赋值
	threadsafe_stack& operator=(const threadsafe_stack&) = delete;

	void push(T new_value)
	{
		std::lock_guard<std::mutex> lock(m);
		data.push(new_value);
	}
	
	void pop(T& value)
	{
		std::lock_guard<std::mutex> lock(m);
		if (data.empty()) throw empty_stack(); // 在调用pop前，检查栈是否为空

		value = data.top(); // 就算T的拷贝或移动赋值运算符发生异常，数据也不会丢失
		data.pop();
	}

	// 不返回值是为了防止T的拷贝或移动构造函数发生异常
	// 如果T的拷贝或移动构造函数都是noexcept的，那么可以返回值
	// 返回shared_ptr是为了方便内存管理
	std::shared_ptr<T> pop()
	{
		std::lock_guard<std::mutex> lock(m);
		if (data.empty()) throw empty_stack();

		std::shared_ptr<T> const res(std::make_shared<T>(data.top())); 
		data.pop();
		return res;
	}	

	bool empty() const
	{
		std::lock_guard<std::mutex> lock(m);
		return data.empty();
	}
};
```

**互斥量保护的粒度不能太小，那样保护不完全；也不能太大，那样会降低性能；对于非共享数据的操作，就不需要互斥量保护；但是要确保一个操作的结果是正确的**。

粒度：通过一个锁保护着的数据量大小。

<h3 id="dead_lock">死锁</h3>

**死锁(deadlock)：两个线程相互等待，导致两个线程都无法正常工作**。

**如果一个操作需要锁住两个或更多互斥量，那么可能会造成死锁**。

避免死锁的一般方法是：**让互斥量总是以相同的顺序上锁**。**标准库函数`std::lock(mutexs)`可以一次性锁住两个或更多互斥量，且不会有死锁的危险**，所以**如果一个操作确实需要两个或更多互斥量，那么使用`std::lock(mutexs)`**。

```c++
// std::lock(mutexs) 示例
class some_big_object;
void swap(some_big_object& lhs, some_big_object& rhs);
class X
{
private:
	some_big_object some_detail;
	std::mutex m;
public:
	X(some_big_object const& sd) :some_detail(sd) {}

	friend void swap(X& lhs, X& rhs)
	{
		// 同一线程如果已经对一个mutex进行lock操作
		// 再次对其进行lock操作会引发未定义行为
		// 但std::recursive_mutex提供这样的操作
		if (&lhs == &rhs)
			return;

		// 当std::lock成功获取了一个互斥量上的锁，并且尝试从另一个互斥量上再获取锁时抛出了异常时
		// 第一个锁也会随着异常的产生而自动释放，所以std::lock要么将两个锁都锁住，要么一个都不锁
		// 使用std::lock需要参数能够提供lock(),unlock(),try_lock()成员函数
		std::lock(lhs.m, rhs.m);

		// 参数std::adopt_lock告诉std::lock_guard对象，互斥量已经被锁住了
		// 只需接收已存在的互斥量的锁定的所有权即可，不要试图构造时锁住它们
		std::lock_guard<std::mutex> lock_a(lhs.m, std::adopt_lock);
		std::lock_guard<std::mutex> lock_b(rhs.m, std::adopt_lock);

		// 如果非函数模板与函数模板提供同样好的匹配，则选择非模板版本
		using std::swap; 
		swap(lhs.some_detail, rhs.some_detail);
	}
};
```

### 避免死锁的进阶指导

死锁不仅存在于有锁的情况，当两个线程调用join相互等待时也能发生死锁。

```c++
// 一个无锁死锁示例
void t2();
void t1() {
	auto t = std::thread(t2);
	t.join();
}
void t2() {
	auto t = std::thread(t1);
	t.join();
}
```

*	**尽量不要使用嵌套锁**，因为这是造成死锁最常见的原因；如果一定要的话，使用`std::lock(mutexs)`；
*	**尽量不要在持有锁的情况下调用用户代码**，因为用户代码可能会获取锁，这样会造成嵌套锁；
*	如果确实不能使用`std::lock(mutexs)`，那么最好在每个线程上**使用固定的顺序获取锁**。典型的如一个线程正在倒序访问双向链表，而另一个正在顺序访问，那么在中间部分就可能发生死锁；如果只能按顺序访问，那就不会出现死锁；
*	**使用锁的层次结构**。只能对比当前层次低(不包括等于)的互斥量上锁，这保证了锁的顺序性，且同一层不可能在同一时间持有两个锁，所以层级结构的互斥量是不可能产生死锁的。

由于标准库并没有定义层次锁，所以需要自己定义`hierarchical_mutex`，为使其能够使用`std::lock_guard`模板，我们查看`std::lock_guard`的公有成员：

```c++
template<class _Mutex>
	class lock_guard<_Mutex>
	{	// specialization for a single mutex
public:
	typedef _Mutex mutex_type;

	explicit lock_guard(_Mutex& _Mtx)
		: _MyMutex(_Mtx)
		{	// construct and lock
		_MyMutex.lock();
		}

	lock_guard(_Mutex& _Mtx, adopt_lock_t)
		: _MyMutex(_Mtx)
		{	// construct but don't lock
		}

	~lock_guard() _NOEXCEPT
		{	// unlock
		_MyMutex.unlock();
		}

	lock_guard(const lock_guard&) = delete;
	lock_guard& operator=(const lock_guard&) = delete;
```

发现**要想使用`std::lock_guard`模板，必须为模板参数提供`lock()`和`unlock()`成员，貌似并没有书中提到的`try_lock()`成员**，下面是一个简单的层次锁实现：

```c++
// 一个简单的层次锁实现
class hierarchical_mutex
{
	std::mutex internal_mutex;

	// 当前锁的层级值
	unsigned long const hierarchy_value;

	// 前一个锁的层级值，用以解锁后恢复原状态
	unsigned long previous_hierarchy_value;

	// 当前线程的层级值
	// thread_local 使得每个线程都有其独立的实例
	static thread_local unsigned long this_thread_hierarchy_value;

	void check_for_hierarchy_violation()
	{
		// 只能使用小于当前层级值的hierarchical_mutex
		if (this_thread_hierarchy_value <= hierarchy_value)
		{
			throw std::logic_error("mutex hierarchy violated");
		}
	}

	void update_hierarchy_value()
	{
		previous_hierarchy_value = this_thread_hierarchy_value;
		this_thread_hierarchy_value = hierarchy_value;
	}

public:
	explicit hierarchical_mutex(unsigned long value) :
		hierarchy_value(value),
		previous_hierarchy_value(0)
	{}

	// 不需要拷贝操作和默认构造函数
	hierarchical_mutex() = delete;
	~hierarchical_mutex() = default;
	hierarchical_mutex(const hierarchical_mutex&) = delete;
	hierarchical_mutex& operator=(const hierarchical_mutex&) = delete;

	void lock()
	{
		check_for_hierarchy_violation();
		internal_mutex.lock();
		update_hierarchy_value();
	}

	void unlock()
	{
		this_thread_hierarchy_value = previous_hierarchy_value;
		internal_mutex.unlock();
	}

	bool try_lock()
	{
		check_for_hierarchy_violation();
		// try_lock: 尝试lock互斥量，并立即返回，成功lock返回true
		// 如果另一个线程已经对该锁进行了lock，那么返回false
		// 如果该线程已经持有该互斥量，则调用try_lock行为未定义
		if (!internal_mutex.try_lock())
			return false;
		update_hierarchy_value();
		return true;
	}
};
// 初始化为最大值是为了一开始任何层次锁都能被lock
thread_local unsigned long
hierarchical_mutex::this_thread_hierarchy_value(ULONG_MAX);
```

```c++
// hierarchical_mutex的简单使用
hierarchical_mutex high_level_mutex(10000);
hierarchical_mutex low_level_mutex(5000);

int do_low_level_stuff();

int low_level_func()
{
	std::lock_guard<hierarchical_mutex> lk(low_level_mutex);
	return do_low_level_stuff();
}

void high_level_stuff(int some_param);

void high_level_func()
{
	std::lock_guard<hierarchical_mutex> lk(high_level_mutex);
	high_level_stuff(low_level_func());
}

// 正确，层次锁从高到低进行上锁
void thread_a()
{
	high_level_func();
}

hierarchical_mutex other_mutex(100);
void do_other_stuff();

void other_stuff()
{
	high_level_func();
	do_other_stuff();
}

// 异常！已lock的other_mutex的层次值比high_level_mutex低
void thread_b()
{
	std::lock_guard<hierarchical_mutex> lk(other_mutex);	
	other_stuff();
}
```

### `std::unique_lock`

`std::unique_lock`比`std::lock_guard`更加灵活，查看几个常见的`std::unique_lock`公有成员：

```c++
// 默认构造函数，不持有mutex
unique_lock() _NOEXCEPT
	: _Pmtx(0), _Owns(false)
	{	// default construct
	}
// 成功lock后，持有该mutex
explicit unique_lock(_Mutex& _Mtx)
	: _Pmtx(&_Mtx), _Owns(false)
	{	// construct and lock
	_Pmtx->lock();
	_Owns = true;
	}
// 当第二个参数是std::adopt_lock时，_Mtx已经lock，并持有(own)该_Mtx
unique_lock(_Mutex& _Mtx, adopt_lock_t)
	: _Pmtx(&_Mtx), _Owns(true)
	{	// construct and assume already locked
	}
// 当第二个参数是std::defer_lock时，_Mtx是unlock状态，但不持有(own)该_Mtx
unique_lock(_Mutex& _Mtx, defer_lock_t) _NOEXCEPT
	: _Pmtx(&_Mtx), _Owns(false)
	{	// construct but don't lock
	}
// 当第二个参数是std::try_to_lock时，尝试lock该mutex，持有(own)与否取决于try_lock
unique_lock(_Mutex& _Mtx, try_to_lock_t)
	: _Pmtx(&_Mtx), _Owns(_Pmtx->try_lock())
	{	// construct and try to lock
	}
// 移动构造函数
unique_lock(unique_lock&& _Other) _NOEXCEPT
	: _Pmtx(_Other._Pmtx), _Owns(_Other._Owns)
	{	// destructive copy
	_Other._Pmtx = 0;
	_Other._Owns = false;
	}
// 移动赋值运算符
unique_lock& operator=(unique_lock&& _Other)
	{	// destructive copy
	if (this != &_Other)
		{	// different, move contents
		if (_Owns)
			_Pmtx->unlock();
		_Pmtx = _Other._Pmtx;
		_Owns = _Other._Owns;
		_Other._Pmtx = 0;
		_Other._Owns = false;
		}
	return (*this);
	}
// 析构函数
~unique_lock() _NOEXCEPT
	{	// clean up
	if (_Owns)
		_Pmtx->unlock();
	}
// 拷贝操作不需要
unique_lock(const unique_lock&) = delete;
unique_lock& operator=(const unique_lock&) = delete;

void lock()
	{	// lock the mutex
	_Validate();
	_Pmtx->lock();
	_Owns = true;
	}

bool try_lock()
	{	// try to lock the mutex
	_Validate();
	_Owns = _Pmtx->try_lock();
	return (_Owns);
	}

void unlock()
	{	// try to unlock the mutex
	if (!_Pmtx || !_Owns)
		_THROW_NCEE(system_error,
			_STD make_error_code(errc::operation_not_permitted));

	_Pmtx->unlock();
	_Owns = false;
	}

void swap(unique_lock& _Other) _NOEXCEPT
	{	// swap with _Other
	_STD swap(_Pmtx, _Other._Pmtx);
	_STD swap(_Owns, _Other._Owns);
	}

_Mutex *release() _NOEXCEPT
	{	// disconnect
	_Mutex *_Res = _Pmtx;
	_Pmtx = 0;
	_Owns = false;
	return (_Res);
	}

bool owns_lock() const _NOEXCEPT
	{	// return true if this object owns the lock
	return (_Owns);
	}
// 类型转换运算符
explicit operator bool() const _NOEXCEPT
	{	// return true if this object owns the lock
	return (_Owns);
	}
```

为了对`std::unique_lock`进行示例，修改[使用std::lock(mutex)的程序](#dead_lock)如下：

```c++
// 相比而言，std::unique_lock会占用比较多的空间，并且比std::lock_guard稍慢一些
class some_big_object;
void swap(some_big_object& lhs, some_big_object& rhs);
class X
{
private:
	some_big_object some_detail;
	std::mutex m;
public:
	X(some_big_object const& sd) :some_detail(sd) {}
	friend void swap(X& lhs, X& rhs)
	{
		if (&lhs == &rhs)
			return;

		// 获取mutex，保持unlock状态，但不持有(own)它
		std::unique_lock<std::mutex> lock_a(lhs.m, std::defer_lock);
		std::unique_lock<std::mutex> lock_b(rhs.m, std::defer_lock);

		// 对获取的mutex进行lock，并持有(own)它
		// lock_a.lock();
		// lock_b.lock();

		// 因为unique_lock能够提供lock(),unlock(),try_lock()成员函数
		// 所以能够使用std::lock
		std::lock(lock_a, lock_b);

		// 使用自定义函数进行数据交换
		using std::swap;
		swap(lhs.some_detail, rhs.some_detail);
	}
};
```

### 保护共享数据的初始化过程

如果数据初始化后锁住一个互斥量，纯粹是为了保护其初始化过程，那么这是没有必要的，并且这会给性能带来不必要的冲击。出于以上的原因，C++标准提供了一种纯粹保护共享数据初始化过程的机制。

很多单线程代码有类似下面的代码块，该代码块称为延迟初始化(Lazy initialization)：

```c++
std::shared_ptr<some_resource> resource_ptr;
void foo()
{
  if(!resource_ptr)
  {
	// reset: 释放原有指针，并获取新指针
    resource_ptr.reset(new some_resource);
  }
  resource_ptr->do_something();
}
```

这种代码转到多线程时，可能会变成：

```c++
std::shared_ptr<some_resource> resource_ptr;
std::mutex resource_mutex;

void foo()
{
	// 所有线程在此序列化，这是没必要的
	std::unique_lock<std::mutex> lk(resource_mutex);
	if (!resource_ptr)
	{
		// 只有初始化过程需要保护
		resource_ptr.reset(new some_resource);
	}
	lk.unlock();
	resource_ptr->do_something();
}
```

解决方案是使用C++标准库`std::once_flag`结构体和`std::call_once`模板：

```c++
std::shared_ptr<some_resource> resource_ptr;
std::once_flag resource_flag; // 和mutex一样，不可复制，也不可移动，只能默认初始化

void init_resource()
{
	resource_ptr.reset(new some_resource);
}

void foo()
{
	// 使用std::call_once比显式使用互斥量消耗更少资源(特别是当初始化完成后)
	// std::call_once可以和任何可调用对象一起使用
	std::call_once(resource_flag, init_resource);
	resource_ptr->do_something();
}
```

静态局部变量的初始化过程在多线程中可能会出现条件竞争，但C++11解决了这个问题：**初始化及定义完全在一个线程中发生，并且没有其他线程可在初始化完成前对其进行处理，条件竞争终止于哪个线程来做这个初始化**。

```c++
class my_class;

// 多线程可以安全的调用get_my_class_instance()
my_class& get_my_class_instance()
{
	static my_class instance;  // 线程安全的初始化过程
	return instance;
}
```

**如果数据很长时间才更新一次的话，使用mutex会降低性能**，因为大部分情况下都只是读取数据而非修改数据，这时**可以使用`boost::shared_mutex`来优化同步性能**。当数据进行更新操作时,可以使用`std::lock_guard<boost::shared_mutex>`和`std::unique_lock<boost::shared_mutex>`进行锁定,这能保证单独访问。这样做的唯一限制是：当一个线程尝试获取独占锁时，它需要等待其它拥有共享锁的线程解锁；当一个线程拥有独占锁时，其它线程不能获取独占锁或共享锁。

```c++
// 一个使用boost::shared_mutex的示例
#include <map>
#include <string>
#include <mutex>
#include <boost/thread/shared_mutex.hpp>
class dns_entry;
class dns_cache
{
	std::map<std::string, dns_entry> entries;
	mutable boost::shared_mutex entry_mutex;

public:
	dns_entry find_entry(std::string const& domain) const
	{
		boost::shared_lock<boost::shared_mutex> lk(entry_mutex);
		std::map<std::string, dns_entry>::const_iterator const it =
			entries.find(domain);
		return (it == entries.end()) ? dns_entry() : it->second;
	}

	void update_or_add_entry(std::string const& domain,
		dns_entry const& dns_details)
	{
		std::lock_guard<boost::shared_mutex> lk(entry_mutex);
		entries[domain] = dns_details;
	}
};
```

<h2 id="synchronizing_thread">多线程同步</h2>

在一个线程完成之前，可能需要等待另一个线程执行完成(如双目摄像头的图像获取显示)，这种情况就需要线程同步。C++标准库提供条件变量(condition variables)和期望(futures)来处理线程同步问题。

当一个线程等待另一个线程完成任务时，可以有很多种方法：

*	持续检查共享数据标志(被mutex保护着)，直到另一线程完成工作时对这个标志进行重设。但这样会消耗不必要的时间检查，并且其它需要该mutex的线程会处于等待状态，消耗系统资源。
*	在等待完成期间,使用`std::this_thread::sleep_for()`函数进行周期性的间歇。这样线程没有浪费执行时间，但是很难确定正确的休眠时间。
*	**(最佳选择)使用C++标准库提供的工具去等待事件发生**。等待一个事件被另一个线程触发最基本的工具是使用条件变量(condition variables)；从概念上讲，一个条件变量与多个事件或其它条件相关联，并且一个或更多线程会等待那个条件的达成；当某些线程确定那个条件达成时，它会通知一个或多个在条件变量上等待的线程，继而唤醒它们，继续工作。

```c++
// 周期间歇
bool flag;
std::mutex m;
void wait_for_flag()
{
	std::unique_lock<std::mutex> lk(m);
	while(!flag)
	{
		lk.unlock();
		std::this_thread::sleep_for(std::chrono::milliseconds(100));
		lk.lock();
	}
}
```

### 条件变量(condition variables)

标准库提供两种条件变量的实现(`<condition_variable>`)：

*	`std::condition_variable`
*	`td::condition_variable_any`

为了进行合适的同步，它们都需要与一个mutex一起工作。但是`std::condition_variable`只能与`std::mutex`一起使用，而`td::condition_variable_any`可以和任何满足最低标准的互斥量一起工作，但是会有一些额外的开销。**一般情况下，首选`std::condition_variable`，当对灵活性有需求时，才会使用`td::condition_variable_any`**。

```c++
// condition_variable的简单使用
#include <iostream>
#include <cstdlib>
#include <thread>
#include <mutex>
#include <string>
#include <queue>
#include <condition_variable>

// 共享队列
std::queue<std::string> data_queue;

// 锁住共享队列，并且与condition_variable相关联
std::mutex mut; 
std::condition_variable data_cond;

// 将每次输入打印在屏幕上，由于只有一个线程在使用std::cout，所以无需上锁
void process(const std::string& str){ std::cout << str << "\n"; }

// 当输入字符串为quit时，结束线程
bool is_last_chunk(const std::string& str){ return str == "quit"; }

void data_preparation_thread(){
	std::string str;
	// 这里只有一个线程在使用std::cin，不用上锁
	while(std::cin >> str){		
		// 在执行共享队列的修改操作时，需要上锁
		std::lock_guard<std::mutex> lk(mut);
		data_queue.push(str);

		// 通知等待data_cond的线程，条件达成
		data_cond.notify_one();
		if(is_last_chunk(str)) break;
	}
}

void data_processing_thread(){
	while(true){
		// 在等待条件变量时及数据弹出队列后，不需要对mut进行lock
		// 所以使用std::unique_lock对mut进行灵活的控制
		std::unique_lock<std::mutex> lk(mut);
		// wait(unique_lock<mutex>&, _Predicate): 接受一个unique_lock实例和一个bool型可调用对象(断言)
		// 如果断言返回true，则wait立即返回，如果断言返回false，则wait()会解锁传递的互斥量,并使线程阻塞
		// 当成员函数notify_one()成员函数被调用时，调用wait()的其中一个线程醒来，重新获取锁，并再次检查断言，如未返回true，则继续解锁等待
		// 当成员函数notify_all()成员函数被调用时，调用wait()的所有线程醒来，重新获取锁，并再次检查断言，如未返回true，则继续解锁等待
		data_cond.wait(
			lk,[]{return !data_queue.empty();});

		// 弹出数据
		auto data = data_queue.front();
		data_queue.pop();
		lk.unlock(); 

		// 对弹出的数据进行操作
		process(data);
		if(is_last_chunk(data)) break;
	}
}

int main()
{
	std::thread t1(data_preparation_thread);
	std::thread t2(data_processing_thread);

	t1.join();
	t2.join();

	getchar();
	return EXIT_SUCCESS;
}
```

结果

```text
kdjfa
kdjfa
quit
quit

```

### 一个使用条件变量的简单线程安全队列

```c++
// 一个使用条件变量的简单线程安全队列
#include <queue>
#include <memory>
#include <mutex>
#include <condition_variable>

template<typename T>
class threadsafe_queue
{
private:
	// mutable：不管函数是否有const限定符，都可以对其进行修改
	// 由于empty()操作(const限定)需要对mut进行lock，但lock会修改mut的状态，所以标记为mutable
	mutable std::mutex mut;
	std::queue<T> data_queue;
	std::condition_variable data_cond;
public:
	threadsafe_queue() = default;
	~threadsafe_queue() = default;
	// 不允许拷贝赋值操作
	threadsafe_queue(const threadsafe_queue&) = delete;
	threadsafe_queue& operator=(const threadsafe_queue&) = delete;

	void push(T new_value)
	{
		std::lock_guard<std::mutex> lk(mut);
		data_queue.push(new_value);
		data_cond.notify_one();
	}

	void wait_and_pop(T& value)
	{
		std::unique_lock<std::mutex> lk(mut);
		data_cond.wait(lk,[this]{return !data_queue.empty();});
		value=data_queue.front();
		data_queue.pop();
	}

	std::shared_ptr<T> wait_and_pop()
	{
		std::unique_lock<std::mutex> lk(mut);
		data_cond.wait(lk,[this]{return !data_queue.empty();});
		std::shared_ptr<T> res(std::make_shared<T>(data_queue.front()));
		data_queue.pop();
		return res;
	}

	bool try_pop(T& value)
	{
		std::lock_guard<std::mutex> lk(mut);
		if(data_queue.empty())
			return false;
		value=data_queue.front();
		data_queue.pop();
		return true;
	}

	std::shared_ptr<T> try_pop()
	{
		std::lock_guard<std::mutex> lk(mut);
		if(data_queue.empty())
			return std::shared_ptr<T>();
		std::shared_ptr<T> res(std::make_shared<T>(data_queue.front()));
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

### 期望(future)

**C++标准库将一次性(one-off)事件称为期望(future)**。当一个线程需要等待一个特定的一次性事件时，在某种程度上来说它知道这个事件在未来的表现形式。之后，这个线程会周期性的等待事件被触发，在等待期间也可以执行其他任务。**期望的事件发生后不能被重置**。

C++标准库有两种类型的futures模板实现(`<future>`)：

*	`std::future<>`：独立期望(unique futures);
*	`std::shared_future<>`：共享期望(shared futures)。

一个`std::future`的实例是它关联事件的唯一实例；而多个`std::shared_future`实例却可以同时关联到同一个事件，这种情况下，所有实例会在同一时间变为ready状态，它们可以访问与事件相关的任何数据;

模板参数是关联数据的类型，void表示事件无关联数据。

**期望类不提供同步访问**，如果多个线程需要访问同一个期望实例，那么它们需要通过[线程间共享数据](#sharing_data_between_threads)来保护访问。

最基本的一次性(one-off)事件是有返回值的后台线程，但是因为`std::thread`并不提供直接接收返回值的机制，所以这里就需要`std::async`函数模板(定义于`<future>`)了。

```c++
// 一个使用std::async的简单示例
#include <iostream>
#include <future>
#include <string>

int find_the_answer_to_ltuae(int, double&);
void do_other_stuff();

struct X
{
	void foo(int,std::string const&);
	std::string bar(std::string const&);
};

struct Y
{
	double operator()(double);
};

class move_only
{
public:
	move_only();
	move_only(move_only&&);
	move_only(move_only const&) = delete;
	move_only& operator=(move_only&&);
	move_only& operator=(move_only const&) = delete;
	void operator()();
};

int main()
{
	// std::async启动一个不需要立即获取结果的异步任务，参数传入与std::thread一样
	// 并返回一个std::future对象，这个对象最后持有函数的返回值
	// 对返回的future对象调用get()，线程阻塞直到future状态变为ready，然后返回结果
	int a = 2;
	double b = 3;
	std::future<int> the_answer = std::async(find_the_answer_to_ltuae,a,std::ref(b));
	do_other_stuff();
	std::cout<<"The answer is "<<the_answer.get()<<std::endl;
	
	X x;
	auto f1 = std::async(&X::foo,&x,42,"hello");// Calls p->foo(42,"hello") where p is &x
	auto f2 = std::async(&X::bar,x,"goodbye");  // Calls tmpx.bar("goodbye") where tmpx is a copy of x
	
	Y y;
	auto f3=std::async(Y(),3.141);              // Calls tmpy(3.141) where tmpy is move-constructed from Y()
	auto f4=std::async(std::ref(y),2.718);      // Calls y(2.718)

	X baz(X&);                                  // function declaration
	std::async(baz,std::ref(x));                // Calls baz(x)
	
	auto f5=std::async(move_only());            // Calls tmp() where tmp is constructed from std::move(move_only())

	getchar();
	return EXIT_SUCCESS;
}
```

你可以在调用`std::async`前传入一些额外的参数：

*	`std::launch::deferred`：函数调用延迟直到成员函数`wait()`或`get()`被调用；
*	`std::launch::async`：函数必须在其自己的线程上运行；
*	`std::launch::deferred | std::launch::async`：默认参数，编译器自己选择。

```c++
// std::async额外参数的简单示例
auto f6=std::async(std::launch::async,Y(),1.2);            // Run in new thread
auto f7=std::async(std::launch::deferred,baz,std::ref(x)); // Run in wait() or get()
auto f8=std::async(
	std::launch::deferred | std::launch::async,
	baz,std::ref(x));                                      // Implementation chooses
auto f9=std::async(baz,std::ref(x));                       // Implementation chooses
f7.wait();                                                 // Invoke deferred function
```

**`std::async`使得算法可以被轻松的分成多个任务，然后同步执行**，但这不是任务关联到`std::future`的唯一方法。你还可以使用类模板`std::packaged_task`和`std::promise`。

### std::packaged_task

`std::packaged_task`绑定一个future`到可调用对象。当`std::packaged_task`被唤醒时，它调用关联的可调用对象，使future变为ready状态，并存储返回值到这个future。

`std::packaged_task`的模板参数是一个函数签名，void表示无返回值无参数函数，`int(std::string&,double*)`代表返回值为int，参数类型为`string &,double *`。

构造`std::packaged_task`实例时，必须传入一个函数或可调用对象，这个函数或可调用对象需要能接收指定的参数和返回可转换为指定返回类型的值。

```c++
// 一个std::packaged_task特例化版本的局部定义
template<>
class packaged_task<std::string(std::vector<char>*,int)>
{
public:
	// 模板构造函数
	template<typename Callable>
	explicit packaged_task(Callable&& f);
	
	// 函数签名返回类型指定get_future的返回类型
	std::future<std::string> get_future();
	
	// 函数签名参数类型指定调用运算符的参数类型
	void operator()(std::vector<char>*,int);
};
```

### std::promise

`std::promise<T>`提供一种设置值(类型为T)的方法，这个值可以在设置之后被关联的`std::future<T>`对象读取。可以通过调用成员函数`get_future()`获取`std::promise<T>`关联的future。当promise通过成员函数`set_value()`设置完值后，关联的future状态变为ready，并且通过其可以获取存储的值。如果promise没有设置值就被销毁了，那么异常会被存储在future中。
