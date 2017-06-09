---
title:      "C++ 多线程应用篇"
subtitle:   "Concurrency In Action"
date:       2017-06-8 20:20:00 +0800
header-img: "img/stock-photo-7.jpg"
tags:
    - C++
    - thread
---

本文知识来自[C++ Concurrency In Action](https://chenxiaowei.gitbooks.io/cpp_concurrency_in_action/content/)，介绍有关C++11多线程(multithreading)相关的代码设计、管理及调试的知识。

#   本文结构

*   [并发代码设计](#design_concurrent_code)
	*	[基于数据的工作划分](#data_based_work_division)
	*	[基于任务类型的工作划分](#task_type_based_work_division)

<h2 id="design_concurrent_code">并发代码设计</h2>

**并发代码的设计比基本数据结构的设计和使用要频繁的多**，要将眼界放宽，才能构建出能有效工作的更大的架构。在写多线程代码时，你不仅需要考虑一般因素，如封装、继承、友元等等，你还需要考虑哪些数据需要被共享、如何同步这些数据、哪些线程需要等待哪些另外的线程，等等。

在代码设计时，你需要考虑要用多少个线程、每个线程做什么工作、是使用“全能”(什么工作都能做)的线程还是“专业”(只能做一种工作)的线程，还是混合着使用，这些选择将决定你代码的性能和清晰度。

<h3 id="data_based_work_division">基于数据的工作划分</h3>

最简单的并行算法是`std::for_each`，它对一个数据集中的每个元素执行相同的操作。为了并发这个算法，你可以为每个元素都分配一个处理线程。如何划分才能获得最大的优化性能很大程度上决定于数据结构的实现细节。最简单的方式是为每个线程分配最多N个元素，但是不管如何划分，每个线程都只管自己分配到的元素，直到操作完成前，没有任何与其他线程的通信。使用过[MPI(Message Passing Interface)](http://mpi-forum.org/)或[openMP](http://www.openmp.org/)架构的人应该非常熟悉：一个任务被划分为一个并行任务集，工作线程独立的执行这些任务，并将结果合并到最终的还原步骤中去。还原步骤就像`accumulate`一样，只不过`accumulate`是累加，结果的合并可能会有所不同。

有时候任务并不能被整洁的划分，因为数据可能只有在运行时才能被明确的划分，但这非常适用于递归算法，如快速排序。

快速排序的基本思想是：随意挑选一个元素，然后将集合中的所有元素按小于该元素和大于该元素分为两组，对两组元素分别再执行上诉操作，如此递归循环，最后将排序集合并，即为最终排序后的集合。你不能简单的划分数据来使其并行，因为分组是运行时才决定的，如果你一定要并行这种算法的话，很自然的你就会想到递归。由于不得不对前后两组分别进行排序，所以每次递归都会更多次的调用`quick_sort`函数，这些递归调用时完全独立的，因为它们访问的是不同元素的数据集，所以可以并发调用。

在[基础篇](https://chorior.github.io/2017/04/24/C++-%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%9F%BA%E7%A1%80%E7%AF%87/#functional_program_with_future)中，我们实现了一个并发的快速排序，使用`std::async`可以要求C++标准库来决定何时在一个新线程上执行任务，以及何时同步执行它。这是非常重要的，因为太多线程可能会导致你的应用程序变得很慢。在递归函数中使用这样的方式是非常好的，你只需要注意线程的数量，而`std::async`可以帮你处理这个问题，但这不是唯一的方式。

一个替代的方案就是使用`std::thread::hardware_concurrency()`函数来选择线程的数量，然后，除了为每个递归调用开启一个新线程，你可以将要排序的数据块push到[一个线程安全stack](https://chorior.github.io/2017/05/24/C++-%E5%A4%9A%E7%BA%BF%E7%A8%8B%E8%AE%BE%E8%AE%A1-%E4%B8%80/#stack)中去，如果一个线程没有事做，它要么已经完成了交给它的数据块的排序，要么正在等待数据块进行排序，它可以从stack中获取一个数据块，并对它们进行排序。

```c++
#include <random>
#include <cstdlib>

#include "myQuickSort.h"

int main()
{
	// 随机数生成
	std::list<int> myList;
	std::uniform_int_distribution<unsigned> u(0, 1000);
	std::default_random_engine e;
	for (int i = 0; i < 1000; ++i) {
		myList.push_back(u(e));
	}

	auto start = std::chrono::high_resolution_clock::now();
	auto res1 = sequential_quick_sort<int>(myList);
	auto end = std::chrono::high_resolution_clock::now();
	std::cout << "seq: " << std::chrono::duration<double, std::milli>(end - start).count() << "ms\n";

	start = std::chrono::high_resolution_clock::now();
	auto res2 = parallel_quick_sort<int>(myList);
	end = std::chrono::high_resolution_clock::now();
	std::cout << "par: " << std::chrono::duration<double, std::milli>(end - start).count() << "ms\n";

	/*for (auto &i : res2)
	{
		std::cout << i << std::ends;
	}*/
}
```

```c++
// myQuickSort.h
#pragma once

#include <iostream>
#include <list>
#include <vector>
#include <atomic>
#include <algorithm>
#include <future>
//#include <boost\smart_ptr\shared_ptr.hpp>

#include "myStack.h"

template<typename T>
struct sorter
{
	struct chunk_to_sort
	{
		std::list<T> data;
		std::promise<std::list<T> > promise;
	};

	threadsafe_stack<chunk_to_sort> chunks;
	std::vector<std::thread> threads;
	unsigned const max_thread_count;
	std::atomic<bool> end_of_data;

	sorter() :
		max_thread_count(std::thread::hardware_concurrency() - 1),
		end_of_data(false)
	{}

	~sorter()
	{
		end_of_data = true;
		for (unsigned i = 0; i<threads.size(); ++i)
		{
			threads[i].join();
		}
	}

	void try_sort_chunk()
	{
		try {
			auto chunk = chunks.pop();
			if (chunk)
			{
				sort_chunk(chunk);
			}
		}
		catch (empty_stack &e)
		{
			//std::cout << e.what();
			e.what(); // 删除会跳出warning: 未引用的局部变量
		}
	}

	std::list<T> do_sort(std::list<T>& chunk_data)
	{
		if (chunk_data.empty())
		{
			return chunk_data;
		}

		std::list<T> result;
		// l.splice(iterator pos,list& x, iterator i)
		// 将x中i指向的元素移动插入到l中pos指向的位置之前
		result.splice(result.begin(), chunk_data, chunk_data.begin());
		T const& partition_val = *result.begin();

		// std::partition(iterator beg, iterator end, func)
		// 将[beg,end)中的元素按func分为两组，第一组使func返回true，第二组使func返回false
		// 返回分组后指向第二组的第一个元素的迭代器，不保证原有元素的顺序
		auto divide_point =
			std::partition(chunk_data.begin(), chunk_data.end(),
				[&](T const& val) {return val < partition_val; });

		chunk_to_sort new_lower_chunk;
		// l.splice(iterator pos, list& x, iterator beg, iterator end)
		// 将x中[beg,end)范围内元素移动插入到l中pos指向的位置之前
		new_lower_chunk.data.splice(new_lower_chunk.data.end(),
			chunk_data, chunk_data.begin(), divide_point);

		std::future<std::list<T> > new_lower =
			new_lower_chunk.promise.get_future();
		chunks.push(std::move(new_lower_chunk)); // 将new_lower_chunk压入到stack

		// 如果当前线程数小于允许的最大线程数，就继续开启一个排序线程
		if (threads.size()<max_thread_count)
		{
			threads.push_back(std::thread(&sorter<T>::sort_thread, this));
		}

		std::list<T> new_higher(do_sort(chunk_data));
		// l.splice(iterator pos, list& x)
		// 将x中所有元素移动插入到l中pos指向的位置之前
		result.splice(result.end(), new_higher);

		// 等待lower part完成排序。如果lower part还未完成或者还没开始，
		// 反正没事做，不如尝试在本线程内把别人的或自己的lower part也做了
		// 如果chunks为空，但是lower part还未完成，chunks.pop就会抛出empty_stack异常
		while (new_lower.wait_for(std::chrono::seconds(0)) !=
			std::future_status::ready)
		{
			try_sort_chunk();
		}
		result.splice(result.begin(), new_lower.get());
		return result;
	}

	void sort_chunk(std::shared_ptr<chunk_to_sort > const& chunk)
	{
		chunk->promise.set_value(do_sort(chunk->data));
	}

	void sort_thread()
	{
		while (!end_of_data)
		{
			try_sort_chunk();
			std::this_thread::yield();
		}
	}
};

template<typename T>
std::list<T> parallel_quick_sort(std::list<T> input)
{
	if (input.empty())
	{
		return input;
	}
	sorter<T> s;
	return s.do_sort(input);
}

template<typename T>
std::list<T> sequential_quick_sort(std::list<T> input)
{
	if (input.empty())
	{
		return input;
	}
	std::list<T> result;
	// l.splice(iterator pos,list& x, iterator i)
	// 将x中i指向的元素移动插入到l中pos指向的位置之前
	result.splice(result.begin(), input, input.begin());
	T const& pivot = *result.begin();

	// std::partition(iterator beg, iterator end, func)
	// 将[beg,end)中的元素按func分为两组，第一组使func返回true，第二组使func返回false
	// 返回分组后指向第二组的第一个元素的迭代器，不保证原有元素的顺序
	auto divide_point = std::partition(input.begin(), input.end(),
		[&](T const& t) {return t<pivot; });

	std::list<T> lower_part;
	// l.splice(iterator pos,list& x, iterator beg, iterator end)
	// 将x中[beg,end)范围内元素移动插入到l中pos指向的位置之前
	lower_part.splice(lower_part.end(), input, input.begin(), divide_point);

	auto new_lower(
		sequential_quick_sort(std::move(lower_part)));
	auto new_higher(
		sequential_quick_sort(std::move(input)));

	// l.splice(iterator pos,list& x)
	// 将x中所有元素移动插入到l中pos指向的位置之前
	result.splice(result.end(), new_higher);
	result.splice(result.begin(), new_lower);
	return result;
}
```

```c++
// myStack.h
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

结果：

```text
seq: 78.2769ms
par: 53.8802ms

```

分析：

*	上面程序中，`threadsafe_stack`很简单，对每个成员函数上锁然后进行操作；
*	`sequential_quick_sort`也很容易理解，一般的递归程序，关键部分也写了注释(注意splice的功能是移动而不是复制)；
*	比较难懂的就是`parallel_quick_sort`了，它首先调用了sorter的`do_sort`函数，该函数与`sequential_quick_sort`类似，但是将lower部分交给了另一个线程，如果不考虑线程的启动时间和线程数的限制的话，理论上会比`sequential`快一倍；与使用`std::async`不同，该实现使用`std::thread::hardware_concurrency()`来控制线程的最大数量，而不是标准库自行决定；当`do_sort`完成之后，`parallel_quick_sort`会进行return，所以`sorter<T> s`会进行析构，停止所有的工作线程(`end_of_data = true`)，并对thread进行清理(join)；一个潜在的问题就是：线程管理及线程间通讯，要解决这两个问题就要增加代码相当大的复杂程度；另外一个问题是：所有的添加和移除都是通过访问stack来完成的，这造成了强烈的竞争，并且会降低性能，也许你想使用无锁的stack会不会解决这个问题，答案是no。

上面的方案就是一个特殊版本的线程池(thread pool)--所有线程的任务都来源于一个等待列表，然后线程会去完成任务，完成任务后会再来列表提取任务。这样的线程池有一些潜在的问题(包括对等待列表的竞争)，如何解决这些问题，将在后面进行讲解。

现在我们有两种方式来划分数据了：一种是在处理开始前、一种是在数据长度固定的情况下进行递归划分。但是并不是所有情况都能这样解决，当数据是动态生成的、或者数据是从外部输入的时候，就不能这样做了，这种情况就需要基于任务类型的划分方式了。

<h3 id="task_type_based_work_division">基于任务类型的工作划分</h3>

基于数据的工作划分是在假设每个线程在分配的数据块上做相同的操作的情况下进行的；另一种划分工作的方式就是使线程变得“专业化”，每个线程执行完全不同的操作，可能会有多个线程工作在同一个数据块上，但是却执行完全不同的操作；这是使用并发分离关注点的结果，每个线程都有一个不同的任务，它独立于其他线程执行；也许偶尔会有其他线程给它传数据，或者触发它需要处理的事件，但是一般来说，每个线程都着重于做一件事情，每段代码只对自己的部分负责。

#### DIVIDING WORK BY TASK TYPE TO SEPARATE CONCERNS
