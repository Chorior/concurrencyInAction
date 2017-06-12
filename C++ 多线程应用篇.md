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
	*	[影响并发代码性能的因素](#factors_affecting_the_performance_of_concurrent_code)

<h2 id="design_concurrent_code">并发代码设计</h2>

**并发代码的设计比基本数据结构的设计和使用要频繁的多**，要将眼界放宽，才能构建出能有效工作的更大的架构。在写多线程代码时，你不仅需要考虑一般因素，如封装、继承、友元等等，你还需要考虑哪些数据需要被共享、如何同步这些数据、哪些线程需要等待哪些另外的线程，等等。

在代码设计时，你需要考虑要用多少个线程、每个线程做什么工作、是使用“全能”(什么工作都能做)的线程还是“专业”(只能做一种工作)的线程，还是混合着使用，这些选择将决定你代码的性能和清晰度。

<h3 id="data_based_work_division">基于数据的工作划分</h3>

最简单的并行算法是`std::for_each`，它对一个数据集中的每个元素执行相同的操作。为了并发这个算法，你可以为每个元素都分配一个处理线程。如何划分才能获得最大的优化性能很大程度上决定于数据结构的实现细节。最简单的方式是为每个线程分配最多N个元素，但是不管如何划分，每个线程都只管自己分配到的元素，直到操作完成前，没有任何与其他线程的通信。使用过[MPI(Message Passing Interface)](http://mpi-forum.org/)或[openMP](http://www.openmp.org/)架构的人应该非常熟悉：一个任务被划分为一个并行任务集，工作线程独立的执行这些任务，并将结果合并到最终的还原步骤中去。还原步骤就像`accumulate`一样，只不过`accumulate`是累加，结果的合并可能会有所不同。

有时候任务并不能被整洁的划分，因为数据可能只有在运行时才能被明确的划分，但这非常适用于递归算法，如快速排序。

快速排序的基本思想是：随意挑选一个元素，然后将集合中的所有元素按小于该元素和大于该元素分为两组，对两组元素分别再执行上诉操作，如此递归循环，最后将排序集合并，即为最终排序后的集合。你不能简单的划分数据来使其并行，因为分组是运行时才决定的，如果你一定要并行这种算法的话，很自然的你就会想到递归。由于不得不对前后两组分别进行排序，所以每次递归都会更多次的调用`quick_sort`函数，这些递归调用是完全独立的，因为它们访问的是不同元素的数据集，所以可以并发调用。

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

单线程中，当有多个任务需要在一段时间内连续不断的运行时，或者程序在其它任务正在进行的情况下仍然需要能够及时处理输入事件(如用户键盘输入、网络数据输入等)时，程序不得不使用单一责任原则(single responsibility principle)来处理这些冲突。在单线程世界里，要处理上面的场景，程序需要先处理一部分任务A，再处理一部分任务B，然后检查是否有用户输入、检查是否有网络数据输入，然后再循环回去继续执行另一部分的任务A...这意味着任务A会变得复杂化，因为它不得不保存它的状态信息，并周期性的返回到主循环中；如果你添加了太多任务到这个循环中，那么用户输入的响应就会变得很慢。有个特殊形式的这种情况相信很多人都见过：当你使程序做一项任务时，接口封锁直到任务完成。

这就是线程诞生的地方。如果你在不同线程中运行每个任务，那么操作系统会为你处理上面的问题。在任务A的代码里，你只需要专注于如何完成这个任务，而不用担心状态的保存、返回到主循环、更不用担心你做这个任务花费了多少时间。操作系统在何时的时候会在适当的时候帮你保存状态，然后切换到任务B或任务C(任务切换)，如果系统拥有多个内核或处理器的话，任务A和B也许可以达到真正的并发(硬件并发)。现在用户等到及时响应了，作为开发者你也简化了自己的代码，因为每个线程都可以专注于直接与其职责相关的操作，再也不必将控制分支与用户交互混合在一起了。

这听起来很美好，但是真的能够实现么？如果所有任务都是相互独立的，线程间也不必相互通信，那么它就是这么easy。然而不幸的是，现实并没有那么美好，这些美好的后台线程经常被要求做一些用户要求做的事情，并且他们需要通过以某种方式更新用户界面来让用户知道他们什么时候完成；或者用户可能想要取消或停止某个后台线程。这两个场景都需要小心的思考、设计并适当的同步，但是**关注的问题仍然是分离的(the concerns are still separate)**：用户接口线程仍然只需要处理用户输入，但是它可能在被其它线程要求做某件事时发生更新；同样的，后台任务仍然只需要关注其任务的实现，只是有时会发生“允许任务被其他线程停止”的情况。

你可能会分离成错误的关注点，线程间共享了大量的数据，或不同线程间相互等待，这两种情况都归结为线程间使用了太多的通信。**如果所有的通信都涉及到相同的问题，也许应该是一个线程的关键责任，并从引用它的所有线程中提取出来；或者如果两个线程通信很密切，但是与其它线程却通信很少，也许它们应该被合并在同一个线程中**；

**当使用任务类型来划分工作时，你不必将自己限制在完全独立的情况下，如果多组输入数据需要相同的操作顺序，你可以将序列中的操作分成多个阶段(stage)，然后分配给每个线程**。

#### DIVIDING A SEQUENCE OF TASKS BETWEEN THREADS

**如果你的任务包括将相同的操作序列应用于许多独立的数据项，则可以使用流水线(pipeline)来利用系统可用的并发性**。这好比一个物理管道：数据流从管道一端进入，在进行一系列操作后，从管道另一端出去。你可以为流水线的每个操作创建一个单独的线程：当一个操作完成之后，数据被放到一个队列中，然后被下一个线程获取。这允许序列中执行第一个操作的线程在开始做下一个数据项的时候，序列中执行第二个操作的线程正在做前一个数据项。这样的划分方式不仅适用于所有数据已知的情况，同样也适用于当操作开始后输入数据不是全部已知的情况。例如，数据可能通过网络进入，或者序列中的第一个操作可能用于扫描文件系统以识别要处理的文件。

通过在线程间划分工作而非数据，你改变了实现的属性。假设您有20个数据项要处理，在四个内核上，每个数据项需要四个步骤，每个步骤需要3秒钟。如果您在四个线程之间划分数据，那么每个线程都有5个数据项要处理，假设没有其他可能会影响时间的处理，12秒后，你将处理4个数据项，那么所有20个数据项一共需要60秒；如果你使用流水线(pipeline)的话，这四个步骤可以被分配到每个内核，现在第一个数据项会被每个内核处理，所以仍然需要12秒，然而12秒之后，一个数据项只需要3秒即可被处理，所以现在全部完成就需要3*19+12=69秒，发现多了9秒，原因是最后一个内核在开始处理第一个数据项时，不得不等待9秒。

**在一些情况下，更平稳，更固定的处理可能是有益的**。例如，考虑一个观看高分辨率数字视频的系统，为了让视频可以观看，通常需要每秒至少25帧或者更多，用户需要这些均匀间隔以给人持续运动的印象；每秒可以解码100帧的应用程序是没有在使用的，如果它暂停一秒钟，然后显示100帧，然后再暂停一秒钟，显示另外100帧；另一方面，用户可能很乐意在开始观看视频前延迟几秒钟。在这种情况下，使用以良好的稳定速率输出帧的管道进行并行化可能会更好。

<h3 id="factors_affecting_the_performance_of_concurrent_code">影响并发代码性能的因素</h3>

**处理器的数量是影响多线程程序性能的一个至关重要的因素**。如果你对目标硬件非常熟悉，那么你可以针对硬件进行设计，但是大多数情况下并不是这样--你可能在一个类似的系统上进行开发，所以与目标系统的差异就显得至关重要了。例如，你在一个双核或者四核的系统上进行开发，但是用户的系统却只有一个多核处理器(带有多个内核)、或者多个单核处理器、甚至多个多核处理器，**在不同平台上，并发程序的行为和性能特点就可能完全不同，所以你需要仔细考虑那些地方会被影响到，如果会被影响，就需要在不同平台上进行测试**。

一个16核处理器与4个四核处理器或16个单核处理器相同--可以同时运行16个线程。如果你想利用这一点，你的程序必须拥有至少16个线程；少于16个将会有空闲的处理器(不考虑其他程序运行的消耗)，但是多于16个将会浪费处理器的运算时间在线程间的任务切换上，这被称为超额认购(oversubscription)。为了允许应用程序根据硬件可以同时运行的线程数量来扩展线程数，C++11标准线程库提供了`std::thread::hardware_concurrency()`，本文最开始的示例也展示了对该函数的使用。

直接使用`std::thread::hardware_concurrency()`需要特别小心，因为代码不会考虑其它运行在该系统上的线程，除非你分享了这个信息。最坏的情况是：如果有多个线程同时调用了一个使用了`std::thread::hardware_concurrency()`的函数来扩展线程数，就会导致庞大的超额认购(oversubscription)。`std::async()`可以避免这个问题，因为标准库知道所有的调用，并能进行适当的安排；小心使用线程池也能避免这个问题。

但是，**即使你考虑到应用程序中运行的所有线程，仍然会受到同时运行的其他应用程序的影响**。一个选项是使用与`std::async()`类似的工具，来为所有执行异步任务的线程的数量做考虑;另一个选项是限制给定应用程序可以使用的处理核心数量。

一个问题的理想算法可能取决于该问题的规模与处理单元数的比值，如果你有一个具有许多处理单元的大规模并行系统，那么整体执行较多操作的算法可能会比执行较少操作的算法更快完成，因为每个处理器只执行很少的操作。

#### Data contention and cache ping-pong

如果两个线程在两个不同的处理器上同时运行，并且它们都在读相同的数据，这通常不会造成问题，因为数据可以被拷贝到各自的缓冲中去；然而，如果有线程修改数据的话，这个修改就需要被更新到其它内核的缓冲区中去，这需要消耗一些时间。根据两个线程上的操作的性质以及用于操作的存储顺序(memory order)，这样的修改可能会让第二个处理器停下来，等待硬件内存更新缓存中的数据。在CPU指令方面，这可能是一个非常慢的操作，相当于数百个单独的指令，尽管精确的时序主要取决于硬件的物理结构。

考虑如下代码：

```c++
std::atomic<unsigned long> counter(0);
void processing_loop()
{
	while (counter.fetch_add(1, std::memory_order_relaxed)<100000000)
	{
		do_something();
	}
}
```

counter是全局的，所以任何调用`processing_loop`的线程都在修改同一个变量，因此，处理器必须确保其缓存中的counter拷贝是最新的，然后修改这个值，并发布到其它处理器，即使你使用的是`std::memory_order_relaxed`。如果另一个处理器上的另一个线程在运行相同的代码，那么counter就必须在两个处理期间来回传递，这样当counter做递增操作时，两个处理器缓存中就会有最新的值；如果`do_something()`足够短，或者有太多处理器在运行这段程序，那么处理器可能会发现在相互等待：一个处理器正准备更新这个值，但是另一个处理器正在做这件事，所以它不得不等待另一个处理器完成其更新并发布出来，这被称为高竞争(high contention)；如果处理期间很少相互等待，那么称为低竞争(low contention)。

像上面的循环，counter会在缓存间来回传递多次，这被称为“乒乓缓存”(cache ping-pong)，它会严重影响应用程序的性能。当一个处理器因为等待缓存更新而停止运行时，这个处理器就不能做任何事情，即使同时有其他可以做有用的工作的线程在等待。这无疑是整个应用程序的坏消息。也许你在想这不可能发生在你的代码里面，因为你不会有上面那样的循环，但是如果你使用mutex呢，如果你在一个循环里面接受一个mutex，那么你的代码就回跟上面的循环出现一样的问题。

```c++
std::mutex m;
my_data data;
void processing_loop_with_mutex()
{
	while (true)
	{
		std::lock_guard<std::mutex> lk(m);
		if (done_processing(data)) break;
	}
}
```

如何避免“乒乓缓存”(cache ping-pong)呢？答案很好地与改善并发潜力的一般准则相联系：**尽可能减少两个线程竞争同一内存位置的潜力(do what you can to reduce the potential for two threads competing for the same memory location)**。然而即使某个内存位置只能由一个线程访问，由于伪共享(false sharing)的原因，你仍然能够获得“乒乓缓存”(cache ping-pong)。

#### False sharing

处理器缓存通常不会用来处理在单个内存位置，但其会用来处理称为缓存行(cache lines)的内存块。
