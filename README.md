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

<h3 id="start_a_thread">线程启动</h3>

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

<h3 id="thread_identification">多线程识别</h3>

线程标识类型是`std::thread::id`，这个值可以通过两种方式进行检索：

*   成员函数`std::thread::get_id()`，当没有线程与该thread对象关联时，此函数返回0；
*   当前线程中调用`std::this_thread::get_id()`；

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
