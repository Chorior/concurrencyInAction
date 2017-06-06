---
title:      "C++ 多线程高级篇"
subtitle:   "Concurrency In Action"
date:       2017-05-24 20:20:00 +0800
header-img: "img/stock-photo-5.jpg"
tags:
    - C++
    - thread
---

本文知识来自[C++ Concurrency In Action](https://chenxiaowei.gitbooks.io/cpp_concurrency_in_action/content/)，介绍有关C++11多线程(multithreading)相关的高级知识。

#   本文结构

*   [C++ 内存模型和原子操作](#the_c_plus_plus_memory_model_and_atomic_operation)
	*	[内存模型](#memory_model)
	*	[对象、存储区与并发](#objects_memory_locations_and_concurrency)
	*	[modification order](#modification_order)
	*	[原子操作与原子类型](#atomic_operations_and_types)
	*	[std::atomic_flag](#std_atomic_flag)
	*	[atomic_bool](#std_atomic_bool)
	*	[atomic_pointer](#std_atomic_pointer)
	*	[标准整型原子类型](#standard_atomic_integral_type)
	*	[std::atomic 模板](#std_atomic_template)
	*	[原子类型的非成员函数](#nonmember_funtions_on_atomic_types)
	*	[同步操作与强制顺序](#synchronize_operations_and_enforce_order)
	*	[synchronizes-with,happen-before](#synchronize_with_and_happen_before)
	*	[原子操作内存顺序(memory order)](#memory_order_for_atomic_operations)
	*	[release sequences and synchronizes-with](#release_sequences_and_synchronizes_with)
	*	[fences](#fences)
*	[基于锁的并发数据结构设计](#designing_lock_based_concurrent_data_structures)
	*	[基于锁的简单并发数据结构设计](#simple_lock_based_concurrent_data_structures)
	*	[基于锁的复杂并发数据结构设计](#complex_lock_based_concurrent_data_structures)

<h2 id="the_c_plus_plus_memory_model_and_atomic_operation">C++ 内存模型和原子操作</h2>

<h3 id="memory_model">内存模型</h3>

**C++ 程序中的所有数据都是由对象(object)构成，C++ 标准定义一个对象(object)为“存储区”**。

像int、float这样的对象(object)是简单基本类型；像array、自定义类型对象拥有子对象(数据成员)。**不管一个对象是什么类型，它都会被存储在一个或多个存储区(memory location)里**，每个存储区(memory location)里面存放着一个对象或子对象，如`unsigned short`或`my_class*`或相邻位位域(bit field)序列。

如果你使用位域(bit field)，那么有一个重要的点你需要注意：**虽然相邻位域都是不同的对象，它们仍被视为相同的存储区(memory location)**。

<h3 id="objects_memory_locations_and_concurrency">对象、存储区与并发</h3>

**C++ 多线程应用程序的关键部分：一切都取决于存储区(everything hinges on those memory locations)**。

如果两个线程访问分离(separate)的存储区，一切都会工作的很好(everything works fine)；如果两个线程访问相同(same)的存储区，你就要小心了：如果没有线程更新这个存储区(只读)，那么没有问题；但如果两个线程都在修改数据，就会有条件竞争(race condition)的危险。

**为了避免条件竞争(race condition)，两个线程不得不按一定的顺序访问存储区**，一种确保访问顺序的方法是使用`std::mutex`，另一种方法就是使用原子操作(atomic operation)。

如果两个线程没有按顺序访问相同的存储区，那么这两个访问中的一个或两个就不是原子(atomic)的，这会造成未定义行为(undefined behavior)。

**原子操作并没有避免竞争本身，哪个原子操作先访问存储区仍然没有被指定，但是它将程序拉回了定义行为的区域内**。

<h3 id="modification_order">modification order</h3>

**一个C++ 程序中的每个对象从初始化开始，就拥有一个定义的修改顺序(modification order)，用来限定程序中的所有线程对该对象的修改操作(Every object in a C++ program has a defined modification order composed of all the writes to that object from all threads in the program, starting with the object’s initialization)**。

**大多数情况下，这个修改顺序(modification order)在运行时会有所变化，但是在程序已经执行的情况下，所有线程都必须遵守这个顺序**。如果一个对象不是原子类型(atomic type)，你就要负责充分的同步，用于确保所有线程都遵守了每个变量的修改顺序(modification order)，否则就会出现数据竞争(data race)和未定义行为(undefined behavior)；如果该对象是原子类型，那么编译器就会帮你确保必要的同步。

**虽然所有线程必须遵守每个对象的修改顺序，但是它们并没有必要遵守不同对象的相对顺序(Although all threads must agree on the modification orders of each individual object in a program, they don’t necessarily have to agree on the relative order of operations on separate objects)**。

<h3 id="atomic_operations_and_types">原子操作与原子类型</h3>

顾名思义，**一个原子操作(atomic operation)是一个不可分割(indivisible)的操作**。

一个原子操作的结果要么未完成，要么已完成(Y/N)，你不能在一个原子操作进行一半的时候进行查看。如果对一个对象读值的操作是原子的，那么所有对该对象的修改操作都是原子的，因为读到的值要么是初始值，要么是已修改后的值。

这意味着**一个非原子操作在进行一半的时候可能被另一个线程看到了，这就很大概率造成数据竞争(data race)和未定义行为(undefined behavior)**。

**在 C++ 里面，大多数情况下，你需要用一个原子类型去获取一个原子操作**。

**标准原子类型定义于头文件`<atomic>`中，对标准原子类型的所有操作都是原子的，且在C++定义中只有对标准原子类型的操作是原子的，虽然你可以使用`std::mutex`来达到原子操作的效果**。

**原子类型没有拷贝操作**，几乎所有的原子类型都有一个成员函数`is_lock_free()`

*	如果该函数返回true，那么对该类型的操作直接以原子指令完成；
*	如果该函数返回false，那么对该类型的操作以使用对编译器和库的内部锁的形式完成。

**唯一不提供`is_lock_free()`成员函数的类型是`std::atomic_flag`**。这个类型是一个简单的布尔标志，对该类型的操作需要是无锁的(lock free)。剩下的原子类型全部都是模板`std::atomic<>`的特例化版本，它们比`std::atomic_flag`拥有更多的功能，但是可能不是无锁的(lock free)。

头文件`<atomic>`中定义了很多`std::atomic<>`模板特例化的别名，命名的格式很简单：

```c++
// ATOMIC TYPEDEFS
typedef atomic<bool> atomic_bool;

typedef atomic<char> atomic_char;
typedef atomic<signed char> atomic_schar;
typedef atomic<unsigned char> atomic_uchar;
typedef atomic<short> atomic_short;
typedef atomic<unsigned short> atomic_ushort;
typedef atomic<int> atomic_int;
typedef atomic<unsigned int> atomic_uint;
typedef atomic<long> atomic_long;
typedef atomic<unsigned long> atomic_ulong;
typedef atomic<long long> atomic_llong;
typedef atomic<unsigned long long> atomic_ullong;

typedef atomic<char16_t> atomic_char16_t;
typedef atomic<char32_t> atomic_char32_t;

typedef atomic<wchar_t> atomic_wchar_t;

typedef atomic<int8_t> atomic_int8_t;
typedef atomic<uint8_t> atomic_uint8_t;
typedef atomic<int16_t> atomic_int16_t;
typedef atomic<uint16_t> atomic_uint16_t;
typedef atomic<int32_t> atomic_int32_t;
typedef atomic<uint32_t> atomic_uint32_t;
typedef atomic<int64_t> atomic_int64_t;
typedef atomic<uint64_t> atomic_uint64_t;

typedef atomic<int_least8_t> atomic_int_least8_t;
typedef atomic<uint_least8_t> atomic_uint_least8_t;
typedef atomic<int_least16_t> atomic_int_least16_t;
typedef atomic<uint_least16_t> atomic_uint_least16_t;
typedef atomic<int_least32_t> atomic_int_least32_t;
typedef atomic<uint_least32_t> atomic_uint_least32_t;
typedef atomic<int_least64_t> atomic_int_least64_t;
typedef atomic<uint_least64_t> atomic_uint_least64_t;

typedef atomic<int_fast8_t> atomic_int_fast8_t;
typedef atomic<uint_fast8_t> atomic_uint_fast8_t;
typedef atomic<int_fast16_t> atomic_int_fast16_t;
typedef atomic<uint_fast16_t> atomic_uint_fast16_t;
typedef atomic<int_fast32_t> atomic_int_fast32_t;
typedef atomic<uint_fast32_t> atomic_uint_fast32_t;
typedef atomic<int_fast64_t> atomic_int_fast64_t;
typedef atomic<uint_fast64_t> atomic_uint_fast64_t;

typedef atomic<intptr_t> atomic_intptr_t;
typedef atomic<uintptr_t> atomic_uintptr_t;
typedef atomic<size_t> atomic_size_t;
typedef atomic<ptrdiff_t> atomic_ptrdiff_t;
typedef atomic<intmax_t> atomic_intmax_t;
typedef atomic<uintmax_t> atomic_uintmax_t;
```

你也可以使用自己的类型来特例化`std::atomic<>`模板。

**每种原子类型的操作都有一个可选的内存顺序(memory order)参数，根据操作的类别，每种操作能使用的参数不尽相同，但所有操作的默认参数都是`memory_order_seq_cst`：**

*	储存操作(store operation)：`memory_order_relaxed`,`memory_order_release`,`memory_order_seq_cst`；
*	加载操作(load  operation)：`memory_order_relaxed`,`memory_order_consume`,`memory_order_acquire`,`memory_order_seq_cst`；
*	读改写操作(read-modify-write)：`memory_order_relaxed`,`memory_order_consume`,`memory_order_acquire`,`memory_order_release`,`memory_order_acq_rel`,`memory_order_seq_cst`。

<h3 id="std_atomic_flag">std::atomic_flag</h3>

**`std::atomic_flag`是最简单的标准原子类型，它表示一个bool flag**。

**`std::atomic_flag`的对象只有两种状态：set、clear**。**这种类型几乎从没被使用过**，除非在十分特别的情况下，但是它会展示一些原子类型的通用策略。

**`std::atomic_flag`对象必须以`ATOMIC_FLAG_INIT`初始化，该初始化使得其状态设置为clear**。

```c++
std::atomic_flag f = ATOMIC_FLAG_INIT;
```

**`std::atomic_flag`是唯一需要以如此特别的方式初始化的原子类型，但它也是唯一保证无锁(lock free)的类型**。

如果一个`std::atomic_flag`对象拥有静态存储期，那么它保证会被静态初始化，这意味着没有初始化顺序问题--对该对象的第一次操作会将其初始化。

**一旦一个`std::atomic_flag`对象被初始化了，你只能对其做三件事：destroy、clear、set并查询前一个值**，分别使用析构函数、`clear()`成员函数、`test_and_set()`成员函数。其中`clear()`是一个存储操作(store operation)，`test_and_set()`是一个读改写操作(read-modify-write)。

**同时对两个不同的对象进行的操作不可能是原子的**，因为先要对一个对象进行操作，再对另一个对象进行操作，这样中间状态就可能被看到，所以原子操作都删除了拷贝操作(包括拷贝构造函数和拷贝赋值运算符)。

有限的特性使得`std::atomic_flag`非常适合用来做spinlock mutex(一种死等的锁机制)：

```c++
// 一个使用std::atomic_flag的spinlock mutex实现
class spinlock_mutex
{
	std::atomic_flag flag;
public:
	spinlock_mutex() :
		flag{ ATOMIC_FLAG_INIT }
	{}
	void lock()
	{
		// test_and_set: 将状态设置为set，即flag设置为true，返回前一个值
		// 如果flag是clear状态，那么循环终止
		// 如果flag是set状态，即该对象已经被lock了，就一直循环等待unlock
		while (flag.test_and_set(std::memory_order_acquire));
	}
	void unlock()
	{
		flag.clear(std::memory_order_release);
	}
};
```

<h3 id="std_atomic_bool">atomic_bool</h3>

由于`std::atomic_flag`没有无修改查询操作，所以它甚至都不能当做一个通用的布尔标志来使用。

最基本的原子整形类型是`std::atomic<bool>`，它比`std::atomic_flag`拥有更多的特性，虽然它仍然没有拷贝操作，但是它**可以使用非原子类型bool变量构造或赋值**：

```c++
std::atomic<bool> b(true);
b = false;
```

查看`std::atomic<>`模板的源代码：

```c++
_ITYPE operator=(_ITYPE _Val) _NOEXCEPT
{	// assign from _Val
return (_ATOMIC_ITYPE::operator=(_Val));
}
```

发现其赋值运算符与常规的赋值运算符不太一样，常规的赋值运算符返回的都是引用，但是`std::atomic<>`模板的赋值运算符返回的却是一个值。**如果返回的是一个引用，那么任何依赖赋值结果的代码就不得不在做一次获取值的操作，这样就可能获取到其它线程修改后的值，通过直接返回存储的非原子类型值，就可以避免这次获取值的操作**。

与`std::atomic_flag`不同，`std::atomic<bool>`通过`store()`成员函数来写值，并且可以设置为true or false；通过`exchange()`来设置新值并返回前一个值；并且提供一个无修改查询函数`load()`，用以将对象隐式的转换为一个普通的bool值：

```c++
std::atomic<bool> b;
bool x = b.load(std::memory_order_acquire);
b.store(true);
x = b.exchange(false, std::memory_order_acq_rel);
```

一个新的操作叫做“比较/交换”(compare/exchange)，它以成员函数`compare_exchange_weak()`和`compare_exchange_strong()`的形式表现出来。**“比较/交换”(compare/exchange)操作是原子类型编程的基石，若期望值与实际值相等，则存储提供的值；若不等，则期望值将被更新为实际值**。

“比较/交换”(compare/exchange)操作的返回值是一个bool值，若期望值与实际值相等则返回true，否则返回false。

```c++
bool compare_exchange_weak(_Ty& _Exp, _Ty _Value,
	memory_order _Order1, memory_order _Order2) _NOEXCEPT
	{	// compare and exchange value stored in *this with *_Exp, _Value
	return (this->_Compare_exchange_weak(
		(void *)_STD addressof(_My_val), (void *)_STD addressof(_Exp), (const void *)_STD addressof(_Value),
			_Order1, _Order2));
	}
bool _Compare_exchange_weak(
	void *_Tgt, void *_Exp, const void *_Value,
	memory_order _Order1, memory_order _Order2) volatile
	{	// lock and compare/exchange
	return (_Atomic_compare_exchange_weak(
		&_My_flag, _Bytes, _Tgt, _Exp, _Value, _Order1, _Order2));
	}
inline int _Atomic_compare_exchange_weak(
	volatile _Atomic_flag_t *_Flag, size_t _Size,
		volatile void *_Tgt, volatile void *_Exp, const volatile void *_Src,
			memory_order, memory_order)
	{	/* atomically compare and exchange with memory ordering */
	int _Result;

	_Lock_spin_lock(_Flag);
	_Result = _CSTD memcmp((const void *)_Tgt, (const void *)_Exp, _Size) == 0;
	if (_Result != 0)
		_CSTD memcpy((void *)_Tgt, (void *)_Src, _Size);
	else
		_CSTD memcpy((void *)_Exp, (void *)_Tgt, _Size);
	_Unlock_spin_lock(_Flag);
	return (_Result);
	}
```

**`compare_exchange_weak()`的存储操作即使在期望值与实际值相等的情况下也可能不会成功**，这使得实际值并没有改变，返回值也是false，这可能发生在缺少“比较/交换”(compare/exchange)指令的机器上。如果处理器不能保证该操作是原子的，就可能发生这种情况，这被称为伪失败(spurious failure)，因为失败的原因不是变量的值。

由于`compare_exchange_weak()`可能发生伪失败(spurious failure)，所以它通常被用在一个循环中：

```c++
bool expected = false;
extern std::atomic<bool> b; // set somewhere else
// 当发生伪失败时，就会继续循环，否则会退出循环
while (!b.compare_exchange_weak(expected, true) && !expected);
```

**`compare_exchange_strong()`保证只有当期望值与实际值不等的情况下才返回false**，这就可以消除上面的循环。

**如果存储的值比较简单，那么使用`compare_exchange_weak()`，否则使用`compare_exchange_strong()`**。

“比较/交换”(compare/exchange)操作接受两个内存顺序(memory order)参数，对应成功和失败两种情况。**禁止提供`memeory_order_release`和`memory_order_acq_rel`给失败的情况；如果你想提供`memory_order_acquire`或`memory_order_seq_cst`给失败情况，那么你必须也提供给成功的情况**。

**如果你没有指定顺序给失败的情况，那么失败的顺序跟成功的顺序一样，除了release部分：`memory_order_release`会变成`memory_order_relaxed`，`memory_order_acq_rel`会变成`memory_order_acquire`。如果你两个都没有指定，那么默认为`memory_order_seq_cst`**。

```c++
std::atomic<bool> b;
bool expected;
b.compare_exchange_weak(expected, true,
	std::memory_order_acq_rel, std::memory_order_acquire);
// 跟上面等价
b.compare_exchange_weak(expected, true, std::memory_order_acq_rel);
```

`std::atomic<bool>`与`std::atomic_flag`的另一个不同是：`std::atomic<bool>`可能不是无锁的(lock free)，其实现可能需要一个内部互斥锁，如上面查找的源码所示。你可以使用`is_lock_free()`成员函数来确认是否是lock free的。

<h3 id="std_atomic_pointer">atomic_pointer</h3>

指针的原子形式是`std::atomic<T*>`，其接口与一般的原子类型基本相同，但它**提供指针算数运算**。基本操作有`fetch_add()`和`fetch_sub()`，它们在存储的地址上做原子加减法并返回原值；另外还有运算符`+=,-=,++,--`，运算符不能提供内存顺序(memory order)，它们总是使用`memory_order_seq_cst`。

```c++
class Foo {};
Foo some_array[5];
std::atomic<Foo*> p(some_array);
Foo* x = p.fetch_add(2);
assert(x == some_array);
assert(p.load() == &some_array[2]);
x = (p -= 1);
assert(x == &some_array[1]);
assert(p.load() == &some_array[1]);
```

<h3 id="standard_atomic_integral_type">标准整型原子类型</h3>

标准整型原子类型如`std::atomic<int>`或`std::atomic<unsigned long long>`拥有相当全面的操作可供使用：`fetch_add()`,`fetch_sub()`,`fetch_and()`,`fetch_or()`,`fetch_xor()`,`+=,-=,&=,|=,^=,++,--`，只有乘除、位移操作没有被支持。

<h3 id="std_atomic_template">std::atomic 模板</h3>

std::atomic 模板的存在使得用户可以创建自己的原子类型。要想创建自己的原子类型，**自定义类不能有任何的虚函数或虚基类，且必须使用编译器生成的拷贝赋值运算符。其基类和非静态数据成员也需要满足该要求**，这使得编译器可以使用memcpy或其它等价的赋值操作，因为没有用户写入的代码。另外，**自定义类型必须是按位可比的(bitwise equality comparable)**，这使得可以使用memcpy和memcmp，并保证了“比较/交换”(compare/exchange)操作的正确工作。

虽然`std::atomic<float>`和`std::atomic<double>`可以使用memcpy和memcmp，但是可能由于其表达形式(科学计数法)，使得原本相等的值使用memcmp得到的结果却不相同。

**越复杂的数据结构就需要越多的操作，到一定复杂度时，你需要使用`std::mutex`**。

**用户定义的原子类型只能使用`load()`、`store()`、`exchange()`、`compare_exchange_weak()`、`compare_exchange_strong()`、以及和自定义类型实例的赋值和转换操作**。

**原子类型能使用的操作**：

operation | `atomic_flag` | `atomic<bool>` | `atomic<T*>` | `atomic<integral type>` | `atomic<other type>`
----------|---------------|----------------|--------------|-------------------------|-------------------------
`test_and_set` | √ | | | | 
`clear` | √ | | | | 
`is_lock_free` | | √ | √ | √ | √
`load` | | √ | √ | √ | √
`store` | | √ | √ | √ | √
`exchange` | | √ | √ | √ | √
`compare_exchange_weak` <br> `compare_exchange_strong` | | √ | √ | √ | √
`fetch_add, +=` | | | √ | √ | 
`fetch_sub, -=` | | | √ | √ | 
`fetch_or, |=` | | | | √ | 
`fetch_and, &=` | | | | √ | 
`fetch_xor, ^=` | | | | √ | 
`++, --` | | | √ | √ | 

<h3 id="nonmember_funtions_on_atomic_types">原子类型的非成员函数</h3>

不同的原子类型也有相同的非成员函数存在：

*	大多数非成员函数相对于相应的成员函数，只是多了`atomic_`前缀；
*	带有`_explicit`后缀的可以指定内存顺序(memory order)，如`std::atomic_store_explicit(&atomic_var,new_value,std::memory_order_release)`；
*	由于成员函数拥有原子对象的隐式引用，所以**非成员函数第一个参数都是原子对象的指针**。

查找visual studio 2017的源代码：

```c++
// GENERAL OPERATIONS ON ATOMIC TYPES (FORWARD DECLARATIONS)
template <class _Ty>
	struct atomic;
template <class _Ty>
	bool atomic_is_lock_free(const volatile atomic<_Ty> *) _NOEXCEPT;
template <class _Ty>
	bool atomic_is_lock_free(const atomic<_Ty> *) _NOEXCEPT;
template <class _Ty>
	void atomic_init(volatile atomic<_Ty> *, _Ty) _NOEXCEPT;
template <class _Ty>
	void atomic_init(atomic<_Ty> *, _Ty) _NOEXCEPT;
template <class _Ty>
	void atomic_store(volatile atomic<_Ty> *, _Ty) _NOEXCEPT;
template <class _Ty>
	void atomic_store(atomic<_Ty> *, _Ty) _NOEXCEPT;
template <class _Ty>
	void atomic_store_explicit(volatile atomic<_Ty> *, _Ty,
		memory_order) _NOEXCEPT;
template <class _Ty>
	void atomic_store_explicit(atomic<_Ty> *, _Ty,
		memory_order) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_load(const volatile atomic<_Ty> *) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_load(const atomic<_Ty> *) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_load_explicit(const volatile atomic<_Ty> *,
		memory_order) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_load_explicit(const atomic<_Ty> *,
		memory_order) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_exchange(volatile atomic<_Ty> *, _Ty) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_exchange(atomic<_Ty> *, _Ty) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_exchange_explicit(volatile atomic<_Ty> *, _Ty,
		memory_order) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_exchange_explicit(atomic<_Ty> *, _Ty,
		memory_order) _NOEXCEPT;
template <class _Ty>
	bool atomic_compare_exchange_weak(volatile atomic<_Ty> *,
		_Ty *, _Ty) _NOEXCEPT;
template <class _Ty>
	bool atomic_compare_exchange_weak(atomic<_Ty> *,
		_Ty *, _Ty) _NOEXCEPT;
template <class _Ty>
	bool atomic_compare_exchange_weak_explicit(
		volatile atomic<_Ty> *, _Ty *, _Ty,
			memory_order, memory_order) _NOEXCEPT;
template <class _Ty>
	bool atomic_compare_exchange_weak_explicit(
		atomic<_Ty> *, _Ty *, _Ty,
			memory_order, memory_order) _NOEXCEPT;
template <class _Ty>
	bool atomic_compare_exchange_strong(volatile atomic<_Ty> *,
		_Ty *, _Ty) _NOEXCEPT;
template <class _Ty>
	bool atomic_compare_exchange_strong(atomic<_Ty> *,
		_Ty *, _Ty) _NOEXCEPT;
template <class _Ty>
	bool atomic_compare_exchange_strong_explicit(
		volatile atomic<_Ty> *, _Ty *, _Ty,
			memory_order, memory_order) _NOEXCEPT;
template <class _Ty>
	bool atomic_compare_exchange_strong_explicit(
		atomic<_Ty> *, _Ty *, _Ty,
			memory_order, memory_order) _NOEXCEPT;

		// TEMPLATED OPERATIONS ON ATOMIC TYPES (DECLARED BUT NOT DEFINED)
template <class _Ty>
	_Ty atomic_fetch_add(volatile atomic<_Ty>*, _Ty) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_fetch_add(atomic<_Ty>*, _Ty) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_fetch_add_explicit(volatile atomic<_Ty>*, _Ty,
		memory_order) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_fetch_add_explicit(atomic<_Ty>*, _Ty,
		memory_order) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_fetch_sub(volatile atomic<_Ty>*, _Ty) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_fetch_sub(atomic<_Ty>*, _Ty) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_fetch_sub_explicit(volatile atomic<_Ty>*, _Ty,
		memory_order) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_fetch_sub_explicit(atomic<_Ty>*, _Ty,
		memory_order) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_fetch_and(volatile atomic<_Ty>*, _Ty) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_fetch_and(atomic<_Ty>*, _Ty) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_fetch_and_explicit(volatile atomic<_Ty>*, _Ty,
		memory_order) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_fetch_and_explicit(atomic<_Ty>*, _Ty,
		memory_order) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_fetch_or(volatile atomic<_Ty>*, _Ty) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_fetch_or(atomic<_Ty>*, _Ty) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_fetch_or_explicit(volatile atomic<_Ty>*, _Ty,
		memory_order) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_fetch_or_explicit(atomic<_Ty>*, _Ty,
		memory_order) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_fetch_xor(volatile atomic<_Ty>*, _Ty) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_fetch_xor(atomic<_Ty>*, _Ty) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_fetch_xor_explicit(volatile atomic<_Ty>*, _Ty,
		memory_order) _NOEXCEPT;
template <class _Ty>
	_Ty atomic_fetch_xor_explicit(atomic<_Ty>*, _Ty,
		memory_order) _NOEXCEPT;
```

`std::atomic_flag`的非成员函数名与上面有些不同：

```c++
inline bool atomic_flag_test_and_set(volatile atomic_flag *_Flag) _NOEXCEPT
	{	// atomically set *_Flag to true and return previous value
	return (_Atomic_flag_test_and_set(&_Flag->_My_flag, memory_order_seq_cst));
	}

inline bool atomic_flag_test_and_set(atomic_flag *_Flag) _NOEXCEPT
	{	// atomically set *_Flag to true and return previous value
	return (_Atomic_flag_test_and_set(&_Flag->_My_flag, memory_order_seq_cst));
	}

inline bool atomic_flag_test_and_set_explicit(
	volatile atomic_flag *_Flag, memory_order _Order) _NOEXCEPT
	{	// atomically set *_Flag to true and return previous value
	return (_Atomic_flag_test_and_set(&_Flag->_My_flag, _Order));
	}

inline bool atomic_flag_test_and_set_explicit(
	atomic_flag *_Flag, memory_order _Order) _NOEXCEPT
	{	// atomically set *_Flag to true and return previous value
	return (_Atomic_flag_test_and_set(&_Flag->_My_flag, _Order));
	}

inline void atomic_flag_clear(volatile atomic_flag *_Flag) _NOEXCEPT
	{	// atomically clear *_Flag
	_Atomic_flag_clear(&_Flag->_My_flag, memory_order_seq_cst);
	}

inline void atomic_flag_clear(atomic_flag *_Flag) _NOEXCEPT
	{	// atomically clear *_Flag
	_Atomic_flag_clear(&_Flag->_My_flag, memory_order_seq_cst);
	}

inline void atomic_flag_clear_explicit(
	volatile atomic_flag *_Flag, memory_order _Order) _NOEXCEPT
	{	// atomically clear *_Flag
	_Atomic_flag_clear(&_Flag->_My_flag, _Order);
	}

inline void atomic_flag_clear_explicit(
	atomic_flag *_Flag, memory_order _Order) _NOEXCEPT
	{	// atomically clear *_Flag
	_Atomic_flag_clear(&_Flag->_My_flag, _Order);
	}
```

C++ 标准库也提供了**以原子的方式访问`std::shared_ptr<>`实例的非成员函数**，这打破了只有原子类型才能使用原子操作的原则，因为`std::shared_ptr<>`决然不是原子类型。但C++标准委员会觉得提供这些额外的函数很重要，支持的原子操作有：load、store、exchange、compare/exchange，这些操作作为标准原子类型相同操作的重载，**使用`std::shared_ptr<>*`当做第一个参数，并且根据是否带有`_explicit`后缀决定是否可以指定内存顺序(memory order)，还可以使用`std::atomic_is_lock_free(shared_ptr<> *)`确认实现是否使用了内部锁**。

```c++
std::shared_ptr<my_data> p;
void update_global_data()
{
	std::shared_ptr<my_data> local(new my_data);
	std::atomic_store(&p, local);
}
void process_global_data()
{
	std::shared_ptr<my_data> local = std::atomic_load(&p);
	process_data(local);
}
```

<h2 id="synchronize_operations_and_enforce_order">同步操作与强制顺序</h2>

```c++
// 在不同的线程中读写变量
#include <vector>
#include <atomic>
#include <thread>
#include <iostream>

std::vector<int> data;
std::atomic<bool> data_ready(false);
void reader_thread()
{
	while (!data_ready.load())
	{
		std::this_thread::sleep_for(std::chrono::milliseconds(1));
	}
	std::cout << "The answer = " << data[0] << "\n";
}
void writer_thread()
{
	data.push_back(42);
	data_ready = true;
}
```

忽略循环等待的低效率，上面的程序保证了数据的写入在`data_ready`置位之前，数据的读取在`data_ready`的置位之后，所以保证了数据的读取在写入之后。这看起来好像是理所当然的，但是原子操作还有其它的顺序选项。

<h3 id="synchronize_with_and_happen_before">synchronizes-with,happen-before</h3>

**有两个非常重要的关系类型：同步(synchronizes-with)、发生在之前(happen-before)**。

*	synchronizes-with：一个变量X的适当标记了的原子写操作W，如果synchronizes-with一个对X的适当标记了的原子读操作，那么该读操作读取到的值一定是W写入的值；默然情况下，所有原子类型都被适当标记过，所以你可以认为：当一个线程写数据，一个线程读数据时，就会发生synchronizes-with关系；
*	happens-before：在单线程中，如果一条语句在另一条之前，那么这条语句happens-before另一条；但如果两个操作发生在同一条语句中，那么这两个操作的顺序根据编译器的不同是不确定的。多线程中，如果一个线程中的操作A比另一个线程中的操作B先发生，那么称A inter-thread happens-before B。

**当一个线程中的操作A synchronizes-with 另一个线程中的操作B时，那么A inter-thread happens-before B**。

```c++
#include <iostream>
#include <cstdlib>

void foo(int a, int b)
{
	std::cout << a << ", " << b << std::endl;
}
int get_num()
{
	static int i = 0;
	return ++i;
}

int main()
{
	// get_num()的调用顺序是不确定的
	foo(get_num(), get_num());

	getchar();
	return EXIT_SUCCESS;
}
```

<h3 id="memory_order_for_atomic_operations">原子操作内存顺序(memory order)</h3>

**原子类型操作有六个可选的内存顺序(memory order)选项**：`memory_order_relaxed`,`memory_order_consume`,`memory_order_acquire`,`memory_order_release`,`memory_order_acq_rel`,`memory_order_seq_cst`，**所有操作的默认内存顺序(memory order)都是`memory_order_seq_cst`**。它们被分成三个模型：

*	序列一致(sequentially consistent)顺序：`memory_order_seq_cst`；
*	获取释放(acquire-release)顺序：`memory_order_consume`,`memory_order_acquire`,`memory_order_release`,`memory_order_acq_rel`；
*	自由(relaxed)顺序：`memory_order_relaxed`。

这些模型在不同CPU架构下的功耗是不同的，如何选择需要了解它们是怎样影响程序的行为的。

#### 序列一致(sequentially consistent)顺序

如果所有原子类型的实例的操作都是序列一致(sequentially consistent)的，那么多线程的行为就像所有这些操作以一定的顺序在单线程中运行一样，这就是为什么它是默认内存顺序(memory order)的原因。**你必须对所有线程使用序列一致(sequentially consistent)，才能获得这一特点**。

**序列一致(sequentially consistent)是最简单、最直观的排序，但是由于其需要对所有线程进行全局同步，所以它也是最昂贵的内存排序(memory order)**。在一个多处理器系统上，使用序列一致(sequentially consistent)可能会导致处理器间进行大量且耗时的通信工作。

```c++
// 一个使用序列一致(sequentially consistent)的简单例子
#include <atomic>
#include <thread>
#include <cassert>

std::atomic<bool> x, y;
std::atomic<int> z;

void write_x()
{
	x.store(true, std::memory_order_seq_cst);
}
void write_y()
{
	y.store(true, std::memory_order_seq_cst);
}
void read_x_then_y()
{
	while (!x.load(std::memory_order_seq_cst));
	if (y.load(std::memory_order_seq_cst))
		++z;
}
void read_y_then_x()
{
	while (!y.load(std::memory_order_seq_cst));
	if (x.load(std::memory_order_seq_cst))
		++z;
}

int main()
{
	x = false;
	y = false;
	z = 0;
	std::thread a(write_x);
	std::thread b(write_y);
	std::thread c(read_x_then_y);
	std::thread d(read_y_then_x);
	a.join();
	b.join();
	c.join();
	d.join();
	assert(z.load() != 0);
}
```

上面的示例中，assert永远不会造成中断，其运行情况可以分为三种：

*	如果`x.store`发生在`y.store`之前，且`read_x_then_y`的`y.load`发生在`y.store`之前，那么由于`read_y_then_x`中`x.load`一定是发生在`y.store`之后，所以`x.load`一定发生在`x.store`之后，所以`x.load`一定返回true，导致z递增加一；
*	上面情况的对称；
*	不管`x.store`和`y.store`谁先发生，如果`read`都发生在`write`之后，那么z会递增两次；

#### 非序列一致(non-sequentially consistent memory)顺序

一旦跨出序列一致(sequentially consistent)的世界，事情就开始变得复杂起来。**其中最大的问题就是：再也没有一个单一的全局的事件顺序了(there’s no longer a single global order of events)**。这意味着：**即使是同一个操作，不同的线程也将看到不同的视界(different views)，你必须将不同线程中操作的精神模型(mental model)整洁的交错在一起**。

**不仅你必须确保事件是真正并发的，并且线程并不需要去遵守事件的顺序(Not only do you have to account for things happening truly concurrently, but threads don’t have to agree on the order of events)**。即使不同线程跑的是同一段代码，由于不同CPU缓存和内部缓冲区在同一个内存区里面可以存储不同的值，导致不同线程的操作缺少了严格的顺序限制。

**在没有其他排序限制的情况下，唯一的要求是：所有线程都必须遵守每个变量的修改顺序(modification order)**。不同变量的操作在不同线程中可以呈现出不同的顺序，只要所观察到的值与施加的任何排序约束一致。

#### 自由(relaxed)顺序

**使用自由(relaxed)顺序的原子操作没有同步(synchronizes-with)关系**。单一线程中**同一变量**的操作依然遵守发生在之前(happen-before)关系，但是线程间几乎不需要相对顺序。唯一的要求是：**同一线程中对同一个原子变量的访问不能被重新排序**。一旦给定的线程已经看到一个原子变量的特定的值，那么该线程随后的读操作就不能获取该变量更早的值了。

**在没有附加同步操作的情况下，使用`memory_order_relaxed`时线程间共享的唯一东西就是：每个变量的修改顺序(modification order)**。

```c++
// 一个使用自由(relaxed)顺序的简单例子
#include <atomic>
#include <thread>
#include <cassert>

std::atomic<bool> x, y;
std::atomic<int> z;

void write_x_then_y()
{
	x.store(true, std::memory_order_relaxed);
	y.store(true, std::memory_order_relaxed);
}
void read_y_then_x()
{
	while (!y.load(std::memory_order_relaxed));
	if (x.load(std::memory_order_relaxed))
		++z;
}

int main()
{
	x = false;
	y = false;
	z = 0;
	std::thread a(write_x_then_y);
	std::thread b(read_y_then_x);
	a.join();
	b.join();
	assert(z.load() != 0);
}
```

上面的示例中，assert可能发生中断：根据自由(relaxed)顺序只支持同一变量在同一线程中的发生在之前(happen-before)关系，得出x、y的store之间是没有发生的先后关系的；根据自由(relaxed)顺序并不支持同步(synchronizes-with)关系，得出x、y的store和load之间也没有先后关系；最后根据以上两点，得出：`x.load`可能发生在`x.store`之前，所以z可能为0。

```c++
// 一个使用自由(relaxed)顺序的复杂例子
#include <thread>
#include <atomic>
#include <iostream>

std::atomic<int> x(0), y(0), z(0);
std::atomic<bool> go(false);

unsigned const loop_count = 10;

struct read_values
{
	int x, y, z;
};

read_values values1[loop_count];
read_values values2[loop_count];
read_values values3[loop_count];
read_values values4[loop_count];
read_values values5[loop_count];

void increment(std::atomic<int>* var_to_inc, read_values* values)
{
	while (!go)
		std::this_thread::yield();
	for (unsigned i = 0; i<loop_count; ++i)
	{
		values[i].x = x.load(std::memory_order_relaxed);
		values[i].y = y.load(std::memory_order_relaxed);
		values[i].z = z.load(std::memory_order_relaxed);
		var_to_inc->store(i + 1, std::memory_order_relaxed);
		std::this_thread::yield();
	}
}

void read_vals(read_values* values)
{
	while (!go)
		std::this_thread::yield();
	for (unsigned i = 0; i<loop_count; ++i)
	{
		values[i].x = x.load(std::memory_order_relaxed);
		values[i].y = y.load(std::memory_order_relaxed);
		values[i].z = z.load(std::memory_order_relaxed);
		std::this_thread::yield();
	}
}

void print(read_values* v)
{
	for (unsigned i = 0; i<loop_count; ++i)
	{
		if (i)
			std::cout << ",";
		std::cout << "(" << v[i].x << "," << v[i].y << "," << v[i].z << ")";
	}
	std::cout << std::endl;
}

int main()
{
	std::thread t1(increment, &x, values1);
	std::thread t2(increment, &y, values2);
	std::thread t3(increment, &z, values3);
	std::thread t4(read_vals, values4);
	std::thread t5(read_vals, values5);

	go = true; // 确保线程几乎同时开始循环

	t5.join();
	t4.join();
	t3.join();
	t2.join();
	t1.join();

	print(values1);
	print(values2);
	print(values3);
	print(values4);
	print(values5);
}
```

上面代码运行起来稍微有些复杂，

*	首先还是从主函数开始看：主函数开了五个线程，其中三个线程除了读取全局原子变量之外，还对传入的原子参数进行了更新操作，然后是两个对原子变量只读线程，在所有线程完成之后，对所有赋值后的结构体数组变量进行打印操作；
*	再来看三个进行过原子更新操作的线程：根据自由(relaxed)顺序支持同一变量在同一线程中的发生在之前(happen-before)关系，所以线程t1对x的操作是有先后关系的，即load->store->load->...->store，t2、t3类似，所以values1所有元素的x成员、values2所有元素的y成员、values3所有元素的z成员一定是0123456789；
*	最后根据自由(relaxed)顺序并不支持同步(synchronizes-with)关系，所以所有线程对各个原子变量操作的相对顺序并没有保证，所以其它值将是0~10之间的任意值，但是因为单一原子变量的load是有先后关系的，所以values同一位置的值将会呈现不严格递增(大于等于)趋势。

一种visual studio 2017的结果是：

```text
(0,4,4),(1,7,7),(2,10,10),(3,10,10),(4,10,10),(5,10,10),(6,10,10),(7,10,10),(8,10,10),(9,10,10)
(0,0,0),(0,1,1),(0,2,2),(0,3,3),(0,4,4),(1,5,5),(1,6,6),(1,7,7),(2,8,8),(2,9,9)
(0,0,0),(0,1,1),(0,2,2),(0,3,3),(0,4,4),(1,5,5),(1,6,6),(1,7,7),(2,8,8),(2,9,9)
(10,10,10),(10,10,10),(10,10,10),(10,10,10),(10,10,10),(10,10,10),(10,10,10),(10,10,10),(10,10,10),(10,10,10)
(0,3,3),(1,6,6),(2,8,8),(3,10,10),(4,10,10),(5,10,10),(6,10,10),(7,10,10),(8,10,10),(9,10,10)
```

再次运行：

```text
(0,4,4),(1,7,7),(2,9,10),(3,10,10),(4,10,10),(5,10,10),(6,10,10),(7,10,10),(8,10,10),(9,10,10)
(0,0,0),(0,1,1),(0,2,2),(0,3,3),(0,4,4),(1,5,5),(1,6,6),(1,7,7),(2,8,8),(2,9,9)
(0,0,0),(0,1,1),(0,2,2),(0,3,3),(0,4,4),(1,5,5),(1,6,6),(1,7,7),(2,8,8),(2,9,9)
(0,0,0),(0,1,1),(0,2,2),(0,2,3),(0,3,3),(0,4,4),(1,5,5),(1,6,6),(1,7,7),(2,8,8)
(0,3,3),(1,6,6),(2,8,8),(3,10,10),(4,10,10),(5,10,10),(6,10,10),(7,10,10),(8,10,10),(9,10,10)
```

**自由原子操作非常难处理，除非特别必要，不要使用自由(relaxed)顺序**。

#### 获取-释放(acquire-release)顺序

获取-释放(acquire-release)顺序是自由(relaxed)顺序的加强版，所有操作仍然没有统一的排序，但是它加入了一些同步。在这种顺序模型下，**原子加载(load)是获取(acquire)操作(memory_order_acquire)，原子存储(store)是释放(release)操作(memory_order_release)，原子读改写(read-modify-write)是获取(acquire)、释放(release)、或者两者都有的操作(memory_order_acq_rel)**。

**同步是成对的，一个线程获取(acquire)，一个线程释放(release)**。一个释放(release)操作同步到(synchronizes-with)一个获取操作，即释放操作发生在获取操作之前，根据上面的知识，即store发生在load之前。这意味着：不同线程仍然可以看到不同的顺序，但这些顺序是有限制的。

```c++
// 一个使用获取-释放(acquire-release)顺序的简单例子
#include <atomic>
#include <thread>
#include <cassert>

std::atomic<bool> x, y;
std::atomic<int> z;

void write_x()
{
	x.store(true, std::memory_order_release);
}
void write_y()
{
	y.store(true, std::memory_order_release);
}
void read_x_then_y()
{
	while (!x.load(std::memory_order_acquire));
	if (y.load(std::memory_order_acquire))
		++z;
}
void read_y_then_x()
{
	while (!y.load(std::memory_order_acquire));
	if (x.load(std::memory_order_acquire))
		++z;
}

int main()
{
	x = false;
	y = false;
	z = 0;
	std::thread a(write_x);
	std::thread b(write_y);
	std::thread c(read_x_then_y);
	std::thread d(read_y_then_x);
	a.join();
	b.join();
	c.join();
	d.join();
	assert(z.load() != 0);
}
```

上面的示例中，assert可能发生中断。主函数开了四个线程，其中两个写，两个读；根据获取-释放(acquire-release)顺序的release和acquire是同步的，所以线程`read_x_then_y`中`x.load`一定发生在`x.store`之后，但`y.load`是不是发生在`y.store`之后就不一定了；同理，线程`read_y_then_x`中`x.load`也不一定发生在`y.store`之后；所以当线程c和d看到线程a和b的不同相对顺序时，z不会发生改变。

```c++
// 一个体现获取-释放(acquire-release)顺序优点的简单例子
#include <atomic>
#include <thread>
#include <cassert>

std::atomic<bool> x, y;
std::atomic<int> z;

void write_x_then_y()
{
	x.store(true, std::memory_order_relaxed);
	y.store(true, std::memory_order_release);
}
void read_y_then_x()
{
	while (!y.load(std::memory_order_acquire));
	if (x.load(std::memory_order_relaxed))
		++z;
}

int main()
{
	x = false;
	y = false;
	z = 0;
	std::thread a(write_x_then_y);
	std::thread b(read_y_then_x);
	a.join();
	b.join();
	assert(z.load() != 0);
}
```

上面的示例中，assert永远不会发生中断。根据获取-释放(acquire-release)顺序的release和acquire是同步的，所以`y.load`一定发生在`y.store`之前，**`memory_order_acquire`保证在`memory_order_release`之前的所有修改(原子的、非原子的)都能被其看见**，所以`x.load`一定发生在`x.store`之后，所以z一定会递增。但如果`y.load`不是在循环里，结果就会不同。

```c++
// memory_order_acquire保证在memory_order_release之前的所有修改都能被其看见
#include <atomic>
#include <thread>
#include <cassert>

std::atomic<int> data[5];
std::atomic<bool> sync1(false), sync2(false);

void thread_1()
{
	data[0].store(42, std::memory_order_relaxed);
	data[1].store(97, std::memory_order_relaxed);
	data[2].store(17, std::memory_order_relaxed);
	data[3].store(-141, std::memory_order_relaxed);
	data[4].store(2003, std::memory_order_relaxed);
	sync1.store(true, std::memory_order_release);
}
void thread_2()
{
	while (!sync1.load(std::memory_order_acquire));
	sync2.store(true, std::memory_order_release);
}
void thread_3()
{
	while (!sync2.load(std::memory_order_acquire));
	assert(data[0].load(std::memory_order_relaxed) == 42);
	assert(data[1].load(std::memory_order_relaxed) == 97);
	assert(data[2].load(std::memory_order_relaxed) == 17);
	assert(data[3].load(std::memory_order_relaxed) == -141);
	assert(data[4].load(std::memory_order_relaxed) == 2003);
}

int main()
{
	std::thread t1(thread_1);
	std::thread t2(thread_2);
	std::thread t3(thread_3);

	t1.join();
	t2.join();
	t3.join();
}
```

上面的示例中，`thread_2`作为中间线程，使得`thread_1`与`thread_3`进行了同步。为了验证`memory_order_release`之后的修改是否能被`memory_order_acquire`看见，修改上面程序，添加一个额外的int全局变量test，在`memory_order_release`之后改变它的值，为了确保验证的正确性，添加一个一秒的延时：

```c++
// memory_order_release之后的修改并不一定能被memory_order_acquire看见
#include <atomic>
#include <thread>
#include <cassert>

std::atomic<int> data[5];
std::atomic<bool> sync1(false), sync2(false);

int test{ 0 };

void thread_1()
{
	data[0].store(42, std::memory_order_relaxed);
	data[1].store(97, std::memory_order_relaxed);
	data[2].store(17, std::memory_order_relaxed);
	data[3].store(-141, std::memory_order_relaxed);
	data[4].store(2003, std::memory_order_relaxed);
	sync1.store(true, std::memory_order_release);

	std::this_thread::sleep_for(std::chrono::milliseconds(1000));
	test = 233;
}
void thread_2()
{
	while (!sync1.load(std::memory_order_acquire));
	sync2.store(true, std::memory_order_release);
}
void thread_3()
{
	while (!sync2.load(std::memory_order_acquire));
	assert(data[0].load(std::memory_order_relaxed) == 42);
	assert(data[1].load(std::memory_order_relaxed) == 97);
	assert(data[2].load(std::memory_order_relaxed) == 17);
	assert(data[3].load(std::memory_order_relaxed) == -141);
	assert(data[4].load(std::memory_order_relaxed) == 2003);

	assert(test == 233); // fire
}

int main()
{
	std::thread t1(thread_1);
	std::thread t2(thread_2);
	std::thread t3(thread_3);

	t1.join();
	t2.join();
	t3.join();
}
```

你也可以使用读改写(read-modify-write)操作来实现上上个示例：

```c++
std::atomic<int> sync(0);
void thread_1()
{
	// ...
	sync.store(1, std::memory_order_release);
}
void thread_2()
{
	int expected = 1;
	while (!sync.compare_exchange_strong(expected, 2,
		std::memory_order_acq_rel))
		expected = 1;
}
void thread_3()
{
	while (sync.load(std::memory_order_acquire)<2);
	// ...
}
```

**如果你确认不需要严格的序列一致(sequentially consistent)顺序，使用获取-释放(require-release)顺序是个不错的选择**。

#### memory_order_consume

`memory_order_consume`是获取-释放(require-release)顺序模型的一部分。它很特别：它完全依赖于数据。

这里有两种新的数据依赖关系：

*	携带依赖(carries-a-dependency-to)：如果操作A的结果被用作操作B的操作数，则A carries-a-dependency-to B。该关系具有传递性。
*	前序依赖(dependency-ordered-before)：该关系以标记为`memory_order_consume`的原子加载(load)操作进行引入，它是`memory_order_acquire`的一个特例；一个标记为`memory_order_release`、`memory_order_acq_rel`或`memory_order_seq_cst`的存储(store)操作A前序依赖于一个标记为`memory_order_consume`的加载(load)操作B，如果B读取的是A存储(store)的值的话。**如果A dependency-ordered-before B，那么A inter-thread happens-before B**。

```c++
// 一个使用memory_order_consume的简单例子
#include <atomic>
#include <thread>
#include <cassert>

struct X
{
	int i;
	std::string s;
};

std::atomic<X*> p;
std::atomic<int> a;

void create_x()
{
	X* x = new X;
	x->i = 42;
	x->s = "hello";
	a.store(99, std::memory_order_relaxed);
	p.store(x, std::memory_order_release);
}
void use_x()
{
	X* x;
	while (!(x = p.load(std::memory_order_consume)))
		std::this_thread::yield();
	assert(x->i == 42);
	assert(x->s == "hello"); 
	assert(a.load(std::memory_order_relaxed) == 99);
}

int main()
{
	std::thread t1(create_x);
	std::thread t2(use_x);
	t1.join();
	t2.join();
}
```

上面的示例中，x的值 dependency-ordered-before p，所以x的值一定是`p.store`存储的值，所以关于x的断言(assert)永远不会发生中断；但是a的断言(assert)并不依赖于p的值，所以它可能发生中断。

当一个值并不 carries-a-dependency-to `memory_order_consume`加载(load)的值时，使用`std::kill_dependency`可以让编译器有更大的空间进行优化，搞不清楚就去[SOF](https://stackoverflow.com/questions/7150395/what-does-stdkill-dependency-do-and-why-would-i-want-to-use-it)看看，但是我觉得并没有必要使用这个东西：

```c++
int global_data[] = { ... };
std::atomic<int> index;
void f()
{
	int i = index.load(std::memory_order_consume);
	do_something_with(global_data[std::kill_dependency(i)]);
}
```

<h3 id="release_sequences_and_synchronizes_with">release sequences and synchronizes-with</h3>

如果store以`memory_order_release`、`memory_order_acq_rel`或`memory_order_seq_cst`标记，load以`memory_order_consume`、`memory_order_acquire`或`memory_order_seq_cst`标记，且链上的每一load操作得到的值都是前一store操作写下的值，那么这个操作链组成了一个释放序列(release sequences)，并且初始化store synchronizes-with (对应`memory_order_acquire`和`memory_order_seq_cst`)或 dependency-ordered-before (对应`memory_order_consume`)最终load。该链上任何原子读改写(read-modify-write)操作可以拥有任何内存顺序(memory order)，甚至`memory_order_relaxed`。

```c++
// 使用原子操作从一个队列中读值
#include <atomic>
#include <thread>
#include <vector>
#include <iostream>

std::vector<int> queue_data;
std::atomic<int> count;

void populate_queue()
{
	unsigned const number_of_items = 20;
	queue_data.clear();
	for (unsigned i = 0; i<number_of_items; ++i)
	{
		queue_data.push_back(i);
	}
	count.store(number_of_items, std::memory_order_release);
}

void process(int &i) { ++i; }

void consume_queue_items()
{
	while (true)
	{
		int item_index;
		// fetch_sub: 调用值减去参数值，并返回原值
		if ((item_index = count.fetch_sub(1, std::memory_order_acquire)) <= 0)
		{
			std::this_thread::yield();
			continue;
		}
		process(queue_data[item_index - 1]);
	}
}

int main()
{
	std::thread a(populate_queue);
	std::thread b(consume_queue_items);
	std::thread c(consume_queue_items);
	a.join();
	b.detach();
	c.detach();

	for (auto &i : queue_data)
	{
		std::cout << i << " ";
	}
	std::cout << std::endl;
}
```

结果：

```text
1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20
```

分析：我们知道当consumer只开一个时，由于`memory_order_acquire`会看到`memory_order_release`之前的全部修改，所以代码会工作的很好。但是现在consumer开了两个，那么第二个consumer就可能看到第一个consumer修改后的值，除非第一个`fetch_sub`使用`memory_order_release`，但这会带来不必要的同步；如果没有释放序列(release sequence)规则或者`fetch_sub`没有使用`memory_order_release`的话，没有什么能够要求第二个consumer能够看到`queue_data`的store操作。庆幸的是，`fetch_sub`参与了释放序列(release sequence)，所以store也同步到第二个`fetch_sub`，并且两个consumer也没有进行同步。下面图的实线显示了发生在之前(happens-before)关系，虚线显示了释放序列(release sequence)。

![release sequence](https://raw.githubusercontent.com/xiaoweiChen/Cpp_Concurrency_In_Action/master/images/chapter5/5-7.png)

这里的链可以有任意数量的连接，只要它们都是读改写(read-modify-write)操作，那么store将会同步到每一个标记为`memory_order_acquire`的操作。

<h3 id="fences">fences</h3>

fences是在没有修改数据的情况下强制内存顺序(memory order)限制的操作，典型的是与`memory_order_relaxed`的原子操作一起使用。fences是全局操作，并且会影响线程中执行了fences的原子操作的顺序。fences通常也被称为内存屏障(memory barriers)，因为它们就像在代码中画出了一条某些(certain)操作无法跨越的线一样。

你应该知道，不同变量上的relaxed操作根据编译器或硬件的不同，可以被任意排序，但fences限制了这个自由，并且引入了happens-before和synchronizes-with关系。

```c++
// 一个使用atomic_thread_fence的简单例子
#include <atomic>
#include <thread>
#include <cassert>

std::atomic<bool> x, y;
std::atomic<int> z;

void write_x_then_y()
{
	x.store(true, std::memory_order_relaxed);
	std::atomic_thread_fence(std::memory_order_release);
	y.store(true, std::memory_order_relaxed);
}
void read_y_then_x()
{
	while (!y.load(std::memory_order_relaxed));
	std::atomic_thread_fence(std::memory_order_acquire);
	if (x.load(std::memory_order_relaxed))
		++z;
}

int main()
{
	x = false;
	y = false;
	z = 0;
	std::thread a(write_x_then_y);
	std::thread b(read_y_then_x);
	a.join();
	b.join();
	assert(z.load() != 0);
}
```

上面的assert永远不会发生中断，因为`atomic_thread_fence`使得`x.store`一定发生在`x.load`之前，所以z一定会递增；但是如果没有`atomic_thread_fence`的话，assert可能会发生中断，因为`x.store`与`y.store`没有顺序。

>This is the general idea with fences: if an acquire operation sees the result of a store that takes place after a release fence, the fence synchronizes-with that acquire operation; and if a load that takes place before an acquire fence sees the result of a release operation, the release operation synchronizes-with the acquire fence.

<h2 id="designing_lock_based_concurrent_data_structures">基于锁的并发数据结构设计</h2>

数据结构的选择是编程问题的重要组成部分，并发编程也不例外。如果一个数据结构被多个线程访问，这个数据结构要么是不可变的，要么需要设计程序用以确保改变在线程间是正确同步的。

**并发设计的一条路是使用mutex，另一条路就是设计数据结构自身，用以支持并发访问，这样的数据结构被称为线程安全(thread safe)**。

使用mutex序列化了线程对数据的访问，要想获得真正的并发访问，你必须对数据结构的设计仔细斟酌。**保护区域越少，被序列化的操作就越少，并发访问的潜力就越大**。

在设计数据结构时，你有两方面需要思考：

*	确保访问是安全(safe)的：一个线程在修改数据时，不能被另一个线程看到中间状态；一个完整的操作不要分成几个操作写在不同的函数里；数据结构的异常处理；死锁(dead-lock)问题；等等；
*	启动真正的并发：当前锁范围内的操作是否能移到锁范围外执行？该数据结构的不同部分能否使用不同的mutex来保护？是不是所有操作都需要相同层级的保护？有没有不改变操作语义又能提升并发概率的简单操作？等等。

<h3 id="simple_lock_based_concurrent_data_structures">基于锁的简单并发数据结构设计</h3>

基于锁的并发数据结构设计就是：**确保在访问数据的时候lock了正确的mutex，并且lock的时长最短**。

你需要保证数据不能在保护区域外被访问，且接口(interface)间不能有条件竞争，如果有多个锁的话还要避免死锁的发生。

#### A thread-safe stack using locks

```c++
#pragma once

// 一个线程安全的简单stack模板
#include <exception>
#include <stack>
#include <mutex>

struct empty_stack : std::exception
{
	const char* what() const throw()
	{
		return "empty stack.\n";
	}
};

template<typename T>
class threadsafe_stack
{
private:
	std::stack<T> data;
	mutable std::mutex m;

public:
	threadsafe_stack() {}

	threadsafe_stack(const threadsafe_stack& other)
	{
		std::lock_guard<std::mutex> lock(other.m);
		data = other.data;
	}

	threadsafe_stack& operator=(const threadsafe_stack&) = delete;

	void push(T new_value)
	{
		std::lock_guard<std::mutex> lock(m);
		data.push(std::move(new_value));
	}

	std::shared_ptr<T> pop()
	{
		std::lock_guard<std::mutex> lock(m);
		if (data.empty()) throw empty_stack();
		std::shared_ptr<T> const res(
			std::make_shared<T>(std::move(data.top())));
		data.pop();
		return res;
	}

	void pop(T& value)
	{
		std::lock_guard<std::mutex> lock(m);
		if (data.empty()) throw empty_stack();
		value = std::move(data.top());
		data.pop();
	}

	bool empty() const
	{
		std::lock_guard<std::mutex> lock(m);
		return data.empty();
	}
};
```

```c++
#include <iostream>
#include <cstdlib>
#include <thread>

#include "myStack.h"

threadsafe_stack<int> g_stack;

void add_data()
{
	int loop_count = 10;
	for (int i = 0; i < loop_count; ++i)
	{
		g_stack.push(i);
	}
}

void print_data()
{
	while (g_stack.empty())
	{
		std::this_thread::yield();
	}

	while (1)
	{
		try {
			std::cout << *g_stack.pop() << std::endl;
		}
		catch(empty_stack &e){
			std::cout << e.what();
			break;
		}
	}
}

int main()
{
	auto t1 = std::thread(add_data);
	auto t2 = std::thread(print_data);

	t1.join();
	t2.join();
}
```

测试结果：

```text
9
8
7
6
5
4
3
2
1
0
empty stack.
```

现在来分析上面的`threadsafe_stack`模板：

*	首先，每个成员函数在操作数据时，都对成员mutex进行了lock，这确保了一个时间只能有一个成员函数访问数据；
*	再来，将标准stack的top与pop放在一起是为了防止条件竞争(race condition)，具体原因可参考[基础篇](https://chorior.github.io/2017/04/24/C++-%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%9F%BA%E7%A1%80%E7%AF%87/#api_design_with_mutex)；
*	然后，对一个mutex进行lock时可能抛出一个异常，但这非常罕见，因为这是mutex出错或者系统资源缺失造成的，但是由于该操作是第一个操作，所以就算发生异常，数据也不会被更改，所以是安全的；**对一个mutex进行unlock时永远不会失败**；
*	接着，对`data.push(std::move(new_value))`的调用可能会抛出异常，如果value的拷贝或移动函数(当T没有移动构造函数时，std::move会调用拷贝构造函数)抛出异常，或者内存不足的话。不管哪种情况，标准stack保证是安全的；
*	继续，两个pop成员函数中的`std::make_shared`和`std::move`都可能抛出异常，但是就算抛出异常，也没有对数据进行更改，所以是安全的；
*	最后，也许你觉得对代码已经分析完了，然而并不是，这里还用三个潜在的使用上的问题：
	*	如果你的模板参数T是自定义类型，并且自定义了赋值运算符=，那么在使用复制构造函数或pop成员函数时，就会造成嵌套锁，这可能会造成死锁(dead-lock)，但是由于上锁的顺序是一定的，所以不太可能发生这个问题；
	*	有两个不安全的成员函数：构造函数和析构函数。因为它们只能被调用一次。**你不能在构造完成之前或析构结束之后访问其实例**；
	*	由于锁的原因，实际上一个时间点只有一个线程在使用这个stack工作；如果一个线程需要等待，那么它需要周期性的调用empty或pop抓取`empty_stack`异常，这消耗了不必要的资源用于检测数据，或者你必须写一些额外的等待代码(如condition variables)，但这又造成了不必要的中间锁，非常浪费。

#### A thread-safe queue using locks and condition variables

```c++
#pragma once

// 一个线程安全的简单queue模板
#include <queue>
#include <mutex>

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
		data_queue.push(std::move(new_value));
		data_cond.notify_one();
	}

	void wait_and_pop(T& value)
	{
		std::unique_lock<std::mutex> lk(mut);
		data_cond.wait(lk, [this] {return !data_queue.empty(); });
		value = std::move(data_queue.front());
		data_queue.pop();
	}

	std::shared_ptr<T> wait_and_pop()
	{
		std::unique_lock<std::mutex> lk(mut);
		data_cond.wait(lk, [this] {return !data_queue.empty(); });
		std::shared_ptr<T> res(
			std::make_shared<T>(std::move(data_queue.front())));
		data_queue.pop();
		return res;
	}

	bool try_pop(T& value)
	{
		std::lock_guard<std::mutex> lk(mut);
		if (data_queue.empty())
			return false;
		value = std::move(data_queue.front());
		data_queue.pop();
		return true;
	}

	std::shared_ptr<T> try_pop()
	{
		std::lock_guard<std::mutex> lk(mut);
		if (data_queue.empty())
			return std::shared_ptr<T>();
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

```c++
#include <iostream>
#include <cstdlib>
#include <thread>

#include "myQueue.h"

threadsafe_queue<int> g_queue;

void add_data()
{
	int loop_count = 10;
	for (int i = 0; i < loop_count; ++i)
	{
		g_queue.push(i);
	}
}

void print_data()
{
	while (1)
	{
		std::cout << *g_queue.wait_and_pop() << std::endl;
	}
}

int main()
{
	auto t1 = std::thread(add_data);
	auto t2 = std::thread(print_data);

	t1.join();
	t2.join();
}
```

测试结果：

```text
0
1
2
3
4
5
6
7
8
9
^C
```

现在来分析上面的`threadsafe_queue`模板：

*	该模板与上面的stack模板类似，唯一不同的就是使用了`std::condition_variable`，所以你再也不用周期性的调用empty来做等待操作了，也不用抓取`empty_stack`异常了；
*	当有多个线程在等待时，由于调用的是`notify_one()`，所以只会有一个线程被唤醒，然而当该线程发生异常时(如`std::make_shared`)，其它线程就不会被唤醒了，这里有三个方案来解决这个问题：
	*	直接使用`notify_all()`；
	*	当异常发生时，调用`notify_one()`尝试唤醒其它线程；
	*	将`std::shared_ptr<>`的初始化移到push成员函数中去，并且标准queue改为存储`std::shared_ptr<>`而不是直接存储数据。下面的代码展示了第三个方案：

```c++
#pragma once

// 一个改善了的线程安全的简单queue模板
#include <queue>
#include <mutex>

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

	void push(T new_value)
	{
		// 内存分配往往是非常耗时的，
		// 将其放置在lock之外，极大地缩短了持有mutex的时长
		std::shared_ptr<T> data(
			std::make_shared<T>(std::move(new_value)));
		std::lock_guard<std::mutex> lk(mut);
		data_queue.push(data);
		data_cond.notify_one();
	}

	void wait_and_pop(T& value)
	{
		std::unique_lock<std::mutex> lk(mut);
		data_cond.wait(lk, [this] {return !data_queue.empty(); });
		value = std::move(*data_queue.front());
		data_queue.pop();
	}

	bool try_pop(T& value)
	{
		std::lock_guard<std::mutex> lk(mut);
		if (data_queue.empty())
			return false;
		value = std::move(*data_queue.front());
		data_queue.pop();
		return true;
	}

	std::shared_ptr<T> wait_and_pop()
	{
		std::unique_lock<std::mutex> lk(mut);
		data_cond.wait(lk, [this] {return !data_queue.empty(); });
		std::shared_ptr<T> res = data_queue.front();
		data_queue.pop();
		return res;
	}

	std::shared_ptr<T> try_pop()
	{
		std::lock_guard<std::mutex> lk(mut);
		if (data_queue.empty())
			return std::shared_ptr<T>();
		std::shared_ptr<T> res = data_queue.front();
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

#### A thread-safe queue using fine-grained locks and condition variables

上面的stack和queue只有一个需要保护的成员，因此只需要一个mutex。为了使用细粒度的锁，你需要观察queue的内部组成，并且关联一个mutex到每一个数据项。

最简单的队列数据结构就是单链表了。

![singly linked list](https://raw.githubusercontent.com/xiaoweiChen/Cpp_Concurrency_In_Action/master/images/chapter6/6-1.png)

```c++
#pragma once

// 一个简单的单线程队列实现
template<typename T>
class queue
{
private:
	struct node
	{
		T data;
		std::unique_ptr<node> next;
		node(T data_) :
			data(std::move(data_))
		{}
	};

	std::unique_ptr<node> head;
	node* tail; // 使用标准指针而非智能指针是因为unique_ptr只能独占

public:
	queue()
	{}

	queue(const queue& other) = delete;
	queue& operator=(const queue& other) = delete;

	std::shared_ptr<T> try_pop()
	{
		if (!head)
		{
			return std::shared_ptr<T>();
		}
		std::shared_ptr<T> const res(
			std::make_shared<T>(std::move(head->data)));
		std::unique_ptr<node> const old_head = std::move(head);
		head = std::move(old_head->next);
		return res;
	}

	void push(T new_value)
	{
		std::unique_ptr<node> p(new node(std::move(new_value)));
		node* const new_tail = p.get();
		if (tail)
		{
			tail->next = std::move(p);
		}
		else
		{
			head = std::move(p);
		}
		tail = new_tail;
	}
};
```

上面的实现对单线程来说，没什么问题。但是如果你要使用细粒度的锁来对应多线程的话，就会出现一系列可能导致你出错的事情。该队列有两个数据成员head和tail，原则上你可以使用两个mutex分别对其进行保护，但这会造成一系列问题：

*	最明显的问题是push函数既可以修改head，也可以修改tail，所以它不得不锁住两个mutex；
*	最严重的问题是当队列只有一个元素时，如果两个线程分别使用push和`try_pop`，由于此时`head.get() == tail`，所以`tail->next`和`head->next`指向的是同一个元素，所以需要同一个mutex进行保护，这还不如最开始的stack和queue呢。解决该问题的方法就是预先添加一个假的没有数据的node，这样就避免了这个问题：
	*	对于一个空队列，head和tail指向同一个假的node而非原来的nullptr；
	*	当你添加一个元素时，更新tail，然后使其指向下一个假元素，这样head和tail将永远指向不同的元素，因此就不会造成对`head->next`和`tail->next`的竞争了。

```c++
#pragma once

// 一个改善的简单的单线程队列实现
template<typename T>
class queue
{
private:
	struct node
	{
		std::shared_ptr<T> data;
		std::unique_ptr<node> next;
	};

	std::unique_ptr<node> head;
	node* tail;

public:
	queue() :
		head(new node), tail(head.get())
	{}

	queue(const queue& other) = delete;
	queue& operator=(const queue& other) = delete;

	std::shared_ptr<T> try_pop()
	{
		if (head.get() == tail)
		{
			return std::shared_ptr<T>();
		}
		std::shared_ptr<T> const res(head->data);
		std::unique_ptr<node> old_head = std::move(head);
		head = std::move(old_head->next);
		return res;
	}

	void push(T new_value)
	{
		std::shared_ptr<T> new_data(
			std::make_shared<T>(std::move(new_value)));
		std::unique_ptr<node> p(new node);
		tail->data = new_data;
		node* const new_tail = p.get();
		tail->next = std::move(p);
		tail = new_tail;
	}
};
```

现在push只访问tail了，`try_pop`虽然同时访问head和tail，但是tail只是被用作初始比较，因此只需要短暂的lock。最大的增益就是`try_pop`和push永远不会在同一个node上操作了，所以你可以大胆的使用一个mutex保护head，一个mutex保护tail。

```c++
#pragma once

// 一个简单的使用细粒度锁的thread safe队列实现
#include <mutex>

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
		if (head.get() == get_tail())
		{
			return nullptr;
		}
		std::unique_ptr<node> old_head = std::move(head);
		head = std::move(old_head->next);
		return old_head;
	}

public:
	threadsafe_queue() :
		head(new node), tail(head.get())
	{}

	threadsafe_queue(const threadsafe_queue& other) = delete;
	threadsafe_queue& operator=(const threadsafe_queue& other) = delete;

	std::shared_ptr<T> try_pop()
	{
		std::unique_ptr<node> old_head = pop_head();
		return old_head ? old_head->data : std::shared_ptr<T>();
	}

	void push(T new_value)
	{
		std::shared_ptr<T> new_data(
			std::make_shared<T>(std::move(new_value)));
		std::unique_ptr<node> p(new node);
		node* const new_tail = p.get();
		std::lock_guard<std::mutex> tail_lock(tail_mutex);
		tail->data = new_data;
		tail->next = std::move(p);
		tail = new_tail;
	}
};
```

将`get_tail`放在`head_mutex`的lock范围内是为了保证得到的tail一定指向假元素，head也一定是当前的head。设想一种情况：将`get_tail`放置在`head_mutex`的lock范围外，如下所示：

```c++
std::unique_ptr<node> pop_head()
{
	node* const old_tail = get_tail();
	std::lock_guard<std::mutex> head_lock(head_mutex);
	if (head.get() == old_tail)
	{
		return nullptr;
	}
	std::unique_ptr<node> old_head = std::move(head);
	head = std::move(old_head->next);
	return old_head;
}
```

当`get_tail`返回时，此时head可能已经发生了改变，在head和tail做比较时，也许tail已经被重新附了值，这样head不是你想要的head，tail也不是你想要的tail，就会造成跟预想结果不同的行为发生，将`get_tail`放在`head_mutex`的lock范围内就能完美的避免这种情况。

```c++
#pragma once

// 一个较完整的使用细粒度锁的thread safe队列
#include <condition_variable>

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

	node* get_tail()
	{
		std::lock_guard<std::mutex> tail_lock(tail_mutex);
		return tail;
	}

	std::unique_ptr<node> pop_head()
	{
		std::unique_ptr<node> old_head = std::move(head);
		head = std::move(old_head->next);
		return old_head;
	}

	std::unique_lock<std::mutex> wait_for_data()
	{
		std::unique_lock<std::mutex> head_lock(head_mutex);
		data_cond.wait(head_lock, [&] {return head.get() != get_tail(); });
		return std::move(head_lock);
	}

	std::unique_ptr<node> wait_pop_head()
	{
		std::unique_lock<std::mutex> head_lock(wait_for_data());
		return pop_head();
	}

	std::unique_ptr<node> wait_pop_head(T& value)
	{
		std::unique_lock<std::mutex> head_lock(wait_for_data());
		value = std::move(*head->data);
		return pop_head();
	}

	std::unique_ptr<node> try_pop_head()
	{
		std::lock_guard<std::mutex> head_lock(head_mutex);
		if (head.get() == get_tail())
		{
			return std::unique_ptr<node>();
		}
		return pop_head();
	}

	std::unique_ptr<node> try_pop_head(T& value)
	{
		std::lock_guard<std::mutex> head_lock(head_mutex);
		if (head.get() == get_tail())
		{
			return std::unique_ptr<node>();
		}
		value = std::move(*head->data);
		return pop_head();
	}

public:
	threadsafe_queue() :
		head(new node), tail(head.get())
	{}

	threadsafe_queue(const threadsafe_queue& other) = delete;
	threadsafe_queue& operator=(const threadsafe_queue& other) = delete;

	void push(T new_value)
	{
		std::shared_ptr<T> new_data(
			std::make_shared<T>(std::move(new_value)));
		std::unique_ptr<node> p(new node);
		{
			std::lock_guard<std::mutex> tail_lock(tail_mutex);
			tail->data = new_data;
			node* const new_tail = p.get();
			tail->next = std::move(p);
			tail = new_tail;
		}
		data_cond.notify_one();
	}

	std::shared_ptr<T> wait_and_pop()
	{
		std::unique_ptr<node> const old_head = wait_pop_head();
		return old_head->data;
	}

	void wait_and_pop(T& value)
	{
		std::unique_ptr<node> const old_head = wait_pop_head(value);
	}

	std::shared_ptr<T> try_pop()
	{
		std::unique_ptr<node> old_head = try_pop_head();
		return old_head ? old_head->data : std::shared_ptr<T>();
	}

	bool try_pop(T& value)
	{
		std::unique_ptr<node> const old_head = try_pop_head(value);
		return old_head;
	}

	void empty()
	{
		std::lock_guard<std::mutex> head_lock(head_mutex);
		return (head.get() == get_tail());
	}
};
```

<h3 id="complex_lock_based_concurrent_data_structures">基于锁的复杂并发数据结构设计</h3>

stack和queue非常简单，它们的接口相对固定，并且它们也应用于比较特殊的场合。大多数数据结构支持各种各样的操作，原则上这能**增大并发的可能性，但是也增大了数据保护的困难度**。

#### Writing a thread-safe lookup table using locks

查找表(lookup table)就是键值对(key-value)的集合，在标准库中，如果键与值相等就是set，不等就是map。

查询表的使用与栈和队列不同。栈和队列上，几乎每个操作都会对数据结构进行修改，不是添加一个元素，就是删除一个，而对于查询表来说，几乎不需要什么修改。和队列和栈一样，标准容器的接口不适合多线程进行并发访问，因为这些接口在设计的时候都存在固有的条件竞争，所以这些接口需要砍掉，以及重新修订。

并发访问时，`std::map<>`接口最大的问题在于——迭代器。当你在处理一个迭代器时，也许其它线程已经删除了该迭代器指向的元素。关于迭代器的多线程设计可以放到后面，现在我们先来看一些查找表(lookup table)的基本操作：

*	添加一个键值对；
*	对于一个给定的键值，修改与其关联的值；
*	删除一个键值和其关联的值；
*	对于一个给定的键值，获取其关联的所有值；
*	是否为空。

如果你坚持使用简单的线程安全规范的话，你也可以在每个成员函数中只使用一个mutex进行上锁保护，这样肯定是安全的。有一个潜在的问题是：当两个线程同时对一个单映射查找表(lookup table)添加相同的键值对时，只有先开始的线程会成功。一种办法就是将添加和修改放在同一个成员函数中去。

当要查找的键值不存在时，可以添加一个默认值。

```c++
mapped_type get_value(key_type const& key, mapped_type default_value);
```

你也可以返回一个`std::pair<mapped_type,bool>`，其中bool值指示该键值是否存在；或者你也可以返回一个智能指针，当其为nullptr时，表示该键值不存在。

一旦接口决定了，你就可以开始码代码了。使用单个mutex对每个成员函数上锁会浪费并发的可能性；一个选项是使用一个支持多线程读和单线程改的mutex，如`boost::shared_mutex`；但理想情况下，我们可以做得更好。

#### designing a map data structure for fine-grained locking

要想得到细粒度的锁，你必须小心查看数据结构的实现细节，而不能直接包装已存在的容器，如`std::map<>`。这边有三个常见的方式来实现关联容器：

*	二叉树，如红黑树；
*	已排序数组；
*	哈希表。

二叉树并不能提供太多扩展并发的机会，每个查找或修改都不得不从访问根节点开始，因此根节点需要被上锁；虽然这个锁在访问线程向树下移动时可以被释放，但并不比直接使用单个锁锁住整个数据结构好太多。

已排序数组更差，因为你不能提前知晓一个给定的数据在该数组中的位置，所以你需要对整个数组上锁。

剩下的就只有哈希表了。假设有一个固定数量的buckets(一个键值拥有一个bucket，其关联的所有值都放在该bucket中)，一个键值所属的bucket纯粹是该键值的一个属性及哈希函数(**Assuming a fixed number of buckets, which bucket a key belongs to is purely a property of the key and its hash function**)。这意味着你可以安全的对每个bucket使用不同的锁。如果你仍然使用类似`boost::shared_mutex`的mutex，你就能将并发的机会增加N倍，其中N是bucket的数量(显然原来只能有一个线程在写，现在可以有N个了)。缺点就是你需要为键值(key)设计一个好的哈希函数(对于一个给定值，任何时候调用此函数都应该返回相同的结果，对于不等的对象几乎总是产生不同的结果)，庆幸的是，C++标准库提供了`std::hash<>`模板，一些基本的类型如int，还有一些常用的标准库类型如`std::string`都提供了特例化，你可以直接拿来使用，对于自定义类型可以查看[模板特例化](https://chorior.github.io/2017/04/04/C++-%E9%87%8A%E7%96%91-%E4%B8%89/#template_specialization)自行特例化。

```c++
#pragma once

// 一个完成了基本功能的线程安全查找表
#include <vector>
#include <list>
#include <mutex>
#include <boost\thread\shared_mutex.hpp>
#include <boost\thread\locks.hpp>

template<typename Key, typename Value, typename Hash = std::hash<Key>>
class threadsafe_lookup_table
{
private:
	class bucket_type
	{
	private:
		typedef std::pair<Key, Value> bucket_value;
		typedef std::list<bucket_value> bucket_data;
		typedef typename bucket_data::iterator bucket_iterator;

		bucket_data data;
		mutable boost::shared_mutex mutex;

		bucket_iterator find_entry_for(Key const& key) const
		{
			return std::find_if(data.begin(), data.end(),
				[&](bucket_value const& item)
			{return item.first == key; });
		}

	public:
		Value value_for(Key const& key, Value const& default_value) const
		{
			boost::shared_lock<boost::shared_mutex> lock(mutex);
			bucket_iterator const found_entry = find_entry_for(key);
			return (found_entry == data.end()) ?
				default_value : found_entry->second;
		}

		void add_or_update_mapping(Key const& key, Value const& value)
		{
			std::unique_lock<boost::shared_mutex> lock(mutex);
			bucket_iterator const found_entry = find_entry_for(key);
			if (found_entry == data.end())
			{
				data.push_back(bucket_value(key, value));
			}
			else
			{
				found_entry->second = value;
			}
		}

		void remove_mapping(Key const& key)
		{
			std::unique_lock<boost::shared_mutex> lock(mutex);
			bucket_iterator const found_entry = find_entry_for(key);
			if (found_entry != data.end())
			{
				data.erase(found_entry);
			}
		}
	};

	std::vector<std::unique_ptr<bucket_type> > buckets;
	Hash hasher;

	bucket_type& get_bucket(Key const& key) const
	{
		std::size_t const bucket_index = hasher(key) % buckets.size();
		return *buckets[bucket_index];
	}

public:
	typedef Key key_type;
	typedef Value mapped_type;
	typedef Hash hash_type;

	threadsafe_lookup_table(unsigned num_buckets = 19, Hash const& hasher_ = Hash()) :
		buckets(num_buckets), hasher(hasher_)
	{
		for (unsigned i = 0; i<num_buckets; ++i)
		{
			buckets[i].reset(new bucket_type);
		}
	}

	threadsafe_lookup_table(threadsafe_lookup_table const& other) = delete;
	threadsafe_lookup_table& operator=(threadsafe_lookup_table const& other) = delete;

	Value value_for(Key const& key, Value const& default_value = Value()) const
	{
		return get_bucket(key).value_for(key, default_value);
	}

	void add_or_update_mapping(Key const& key, Value const& value)
	{
		get_bucket(key).add_or_update_mapping(key, value);
	}

	void remove_mapping(Key const& key)
	{
		get_bucket(key).remove_mapping(key);
	}
};
```

上面的示例函数不多，仔细看个十分钟左右应该能够看懂。仔细看完，你会发现，该代码其实是相当简单的，但是每个关键的地方都做了适当的数据保护。其中bucket数量的默认值是19，这个值是随便写的，但是哈希表在拥有质数个buckets的时候工作的最好；因为bucket的数量是固定的，所以`get_bucket`可以在无锁的状态下被调用，然后再通过`bucket_type`的成员函数进行适当的数据保护。

该示例是线程安全的，但是不是异常安全的呢？首先，`value_for`并不修改数据，它抛不抛出异常无关紧要；`remove_mapping`调用了erase，但erase保证不会抛出异常，所以是安全的；最后是`add_or_update_mapping`，它的两个if分支都可能抛出异常，`push_back`如果抛出异常会将list保持在原有状态，所以是安全的，如果赋值运算符抛出异常，你可以祈祷它保持在原有状态，然而这不并影响整个数据结构，这完全是用户提供的类型的一个属性，你可以安全的留给用户来处理这件事。

在获取全部键值对快照的时候，需要锁住全部的buckets：

```c++
std::map<Key, Value> get_map() const
{
	std::vector<std::unique_lock<boost::shared_mutex> > locks;
	for (unsigned i = 0; i<buckets.size(); ++i)
	{
		locks.push_back(
			std::unique_lock<boost::shared_mutex>(buckets[i].mutex));
	}
	std::map<Key, Value> res;
	for (unsigned i = 0; i<buckets.size(); ++i)
	{
		for (auto it = buckets[i].data.begin();
			it != buckets[i].data.end();
			++it)
		{
			res.insert(*it);
		}
	}
	return res;
}

std::vector<Key> get_Key() const
{
	std::vector<std::unique_lock<boost::shared_mutex> > locks;
	for (unsigned i = 0; i<buckets.size(); ++i)
	{
		locks.push_back(
			std::unique_lock<boost::shared_mutex>(buckets[i].mutex));
	}
	std::vector<Key> res;
	for (unsigned i = 0; i<buckets.size(); ++i)
	{
		for (auto it = buckets[i].data.begin();
			it != buckets[i].data.end();
			++it)
		{
			res.insert(it->first);
		}
	}
	return res;
}
```

在测试编译时，`bucket_type`的`find_entry_for`一直提示不能将`const_iterator`转换成`iterator`，发现其是const限定的，所以data是const的，故而`data.begin()`、`data.end()`返回的都是`const_iterator`，而函数要求返回的是`iterator`，由于后面的`add_or_update_mapping`和`remove_mapping`也调用了该函数，并且修改了其指向的数据，所以不能直接将`bucket_iterator`改为`bucket_const_iterator`，解决方案是重载该函数，一个const限定返回`bucket_const_iterator`，一个没有const限定，返回`bucket_iterator`，如下所示：

```c++
typedef std::pair<Key, Value> bucket_value;
typedef std::list<bucket_value> bucket_data;
typedef typename bucket_data::iterator bucket_iterator;
typedef typename bucket_data::const_iterator bucket_const_iterator;

bucket_data data;
mutable boost::shared_mutex mutex;

bucket_iterator find_entry_for(Key const& key)
{
	return std::find_if(data.begin(), data.end(),
		[&](bucket_value const& item){return item.first == key; });
}

bucket_const_iterator find_entry_for(Key const& key) const
{
	return std::find_if(data.begin(), data.end(),
		[&](bucket_value const& item) {return item.first == key; });
}
```

测试代码如下：

```c++
#include <iostream>
#include <cstdlib>
#include <thread>

#include "myLookupTable.h"

threadsafe_lookup_table<int,int> g_lookup_table;

void add_data()
{
	int loop_count = 10;
	for (int i = 0; i < loop_count; ++i)
	{
		g_lookup_table.add_or_update_mapping(i, i);
	}
}

void print_data()
{
	int loop_count = 10;
	for (int i = 0; i < loop_count; ++i)
	{
		std::cout << g_lookup_table.value_for(i, 0) << std::endl;
	}
}

int main()
{
	auto t1 = std::thread(add_data);
	auto t2 = std::thread(print_data);

	t1.join();
	t2.join();
}
```

`boost::thread`的库需要编译，步骤如下：

*	dos下进入到你下载的boost解压包文件夹，我的是`cd E:\boost_1_64_0\boost_1_64_0`；
*	`cd tools\build`；
*	`bootstrap.bat`；
*	复制生成的`bjam.exe`到`E:\boost_1_64_0\boost_1_64_0`下；
*	`cd E:\boost_1_64_0\boost_1_64_0`；
*	根据你的vs版本，你可以先不添加库编译，vs会告诉你需要的库及你的vs版本号，`bjam --toolset=msvc-14.1 --with-thread stage`；
*	`bjam --toolset=msvc-14.1 --with-date_time stage`；
*	最后需要的lib在`E:\boost_1_64_0\boost_1_64_0\stage\lib`下；
*	在debug/x86模式下添加库输入`libboost_date_time-vc141-mt-gd-1_64.lib`和`libboost_thread-vc141-mt-gd-1_64.lib`即可。

测试结果(测试程序可能需要修改，用于确保print发生在add之后)：

```text
0
1
2
3
4
5
6
7
8
9

```
