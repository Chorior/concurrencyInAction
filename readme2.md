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
