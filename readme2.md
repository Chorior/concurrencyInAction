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

*	同步(synchronizes-with)：一个变量X的适当标记了的原子写操作W，如果同步到一个对X的适当标记了的原子读操作，那么该读操作读取到的值一定是W写入的值；默然情况下，所有原子类型都被适当标记过，所以你可以认为：当一个线程写数据，一个线程读数据时，就会发生同步(synchronizes-with)关系；
*	发生在之前(happen-before)：在单线程中，如果一条语句在另一条之前，那么这条语句发生在另一条之前；但如果两个操作发生在同一条语句中，那么这两个操作的顺序根据编译器的不同是不确定的。多线程中，如果一个线程中的操作A比另一个线程中的操作B先发生，那么称A线程间发生在B之前(inter-thread happens-before)。

**当一个线程中的操作A同步到(synchronizes-with)另一个线程中的操作B时，那么A线程间发生在B之前(inter-thread happens-before)**。

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
