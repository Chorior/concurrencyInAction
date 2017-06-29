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
	*	[为多线程性能设计数据结构](#designing_data_structures_for_multithreaded_performance)
	*	[多线程设计时的其它注意事项](#additional_considerations_when_designing_for_concurrency)
	*	[多线程改善响应能力](#improving_responsiveness_with_concurrency)
	*	[std::for_each](#std_for_each)
	*	[std::find](#std_find)
	*	[std::partial_sum](#std_partial_sum)

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

如何避免“乒乓缓存”(cache ping-pong)呢？答案很好地与改善并发潜力的一般准则相联系：**尽可能减少两个线程竞争同一内存位置(memory location)的潜力(do what you can to reduce the potential for two threads competing for the same memory location)**。然而即使某个内存位置只能由一个线程访问，由于伪共享(false sharing)的原因，你仍然能够获得“乒乓缓存”(cache ping-pong)。

#### False sharing

**处理器缓存通常不会用来处理在单个内存位置(memory location)，但其会用来处理称为高速缓存行(cache line)的内存块**。这些内存块大小通常为32或64字节，但具体细节取决于所使用的特定处理器型号。由于高速缓存硬件仅在高速缓存行(cache line)大小的内存块中进行处理，所以相邻内存位置(memory location)中的小数据项将位于相同的高速缓存行(cache line)中。有时候这是很好的：如果线程访问的数据集在同一个cache line中，那么程序性能会比数据集分布多个cache line中要好。然而，如果cache line中的数据项是不相关的，且需要被不同线程访问，这就会导致一个主要的性能问题。

假设你有一个int数组和一个线程集，每个线程都访问数组中的自己的条目，但重复执行，包括更新。由于int通常比缓存行小得多，所以相当数量的条目将在同一个cache line中，所以即使每个线程只访问自己的数组条目，但仍然发生了“乒乓缓存”(cache ping-pong)：每次访问条目0的线程A需要更新值时，需要将cache line的所有权传输到运行A的处理器，当访问条目1的线程B需要更新值时，cache line的所有权又被传输到运行B的处理器。虽然没有数据是共享的，但是cache line却是共享的，这就是术语false sharing的由来。这里的解决方案是**结构化数据，使得同一线程要访问的数据项在内存中相互靠近（从而更可能在同一个cache line中），而不同线程访问的数据项在内存中相互远离，因此更有可能在单独的cache line中**。

#### How close is your data?

false sharing是由于一个线程访问的数据太靠近另一个线程访问的数据引起的；另一个与数据布局相关的问题是数据接近度(proximity)：如果一个线程的访问的数据在内存中散列分布，那么这些数据很可能位于不同的cache line上，反之，如果数据在内存中相互靠近，那么这些数据就更可能位于相同的cache line上。所以，**如果数据散列分布，就需要将更多的cache line加载到处理器缓存上，这会增加内存的访问延迟，并且降低性能**。另外，如果数据散列分布，那么cache line中就会有更大的可能包含其它线程访问的数据，极端情况下，你所需要的数据甚至比你不关心的数据要少；这将浪费宝贵的缓存空间，从而增加处理器未命中的概率。

如果线程数量多于内核或处理器数量，操作系统可能会选择将一个线程安排给这个内核一段时间，之后再安排给另一个内核一段时间，这就需要将cache line从一个内核上转移到另一个内核上；**转移的cache line越多，耗费的时间就越多**。虽然操作系统尽量避免这样的情况发生，不过当其发生的时候，就会对性能有很大的影响。

#### Oversubscription and excessive task switching

**在多线程系统中，线程数通常比处理器数要多，除非你使用的是大型并发系统**。然而线程经常花费时间等待外部I/O、或等待mutex、或等待条件变量等等，所以拥有额外的线程使得程序能够执行有用的工作，而不是让处理器在等待时处于空闲状态。但是当线程数太多，以致等待运行的线程数超过了可用的处理器数时，系统就会开始任务切换，这也会增加时间开销，当一个任务重复产生新线程而不受控制时，可能会出现超额认购(oversubscription)。

<h3 id="designing_data_structures_for_multithreaded_performance">为多线程性能设计数据结构</h3>

**为多线程性能设计数据结构的关键点在于：竞争(contention)、伪共享(false sharing)、数据接近度(proximity)**。这三个点都能对性能造成巨大的影响，你通常可以修改数据布局、或更改为每个线程分配的数据来改善你的代码。

#### Dividing array elements for complex operations

假设你需要对两个超大的矩阵进行相乘，我们知道矩阵的乘法如下：

```text
如果 AB = C，其中A、B、C都是矩阵
那么 Cij = Ai0*B0j + Ai1*B1j + Ai2*B2j + ...
```

假设这两个超大的矩阵拥有上千行、上千列，那么使用多线程可以优化该乘法。通常，非稀疏矩阵在内存中用一个大数组表示，第二行的所有元素跟随在第一行的所有元素之后，以此类推。为了做矩阵乘法，你需要三个这样的大数组，两个用于相乘，一个用于结果。

你可以使用多种方法来划分工作。如果你的行列数超过了可用的处理器数，那么你可以让每个线程计算一定数量的结果元素，可以是几行、几列、或者一个子矩阵，都没问题。但是我们知道，**访问连续的元素会比访问分散的元素要好**，因为前者会减少缓存的使用量和false sharing的概率。

现在假设第一个矩阵A是N行M列，第二个矩阵B是M行K列，且每行元素使用一个cache line。如果每个线程计算结果C的一行元素，那么一个线程使用的cache line数为1+M+1，其中只有对B的数据的访问是分散的，各个线程使用的A和C的cache line都是不同的；如果每个线程计算结果C的一列元素，那么一个线程使用的cache line数为N+M+N，各个线程使用的cache line都是一样的，这不仅会增加缓存的使用量，还会增加false sharing的概率；如果每个线程计算结果C的一个PxQ子矩阵元素，那么一个线程使用的cache line数为P+Q+P，各个线程使用的cache line可能相同，但相对于第一种方法，可能会减少缓存的使用量。

考虑两个1000x1000的矩阵相乘，你有100个处理器。如果每个处理器处理结果的10行元素，那么需要访问第一个矩阵的10x1000个元素，第二个矩阵的1000x1000个元素，结果矩阵的10x1000个元素，一共需要访问1020000个元素；如果每个处理器处理结果的100x100子矩阵元素，那么需要访问第一个矩阵的100x1000个元素，第二个矩阵的100x1000个元素，结果矩阵的100x100个元素，一共需要访问210000个元素，是处理10行元素访问元素的五分之一，所以会更好的提升性能。

**将工作划分为小块可能会工作的更好，你可以根据源矩阵的大小和处理器的数量来动态的对块的大小进行调整**。

也许你在想这个例子到底是想说什么？答案就是：**很多情况下，你并不需要修改基本算法，你只需要简单的修改划分方式就能很好的提升性能了**。

#### Data access patterns in other data structures

从根本上来讲，当你尝试优化其它数据结构的数据访问模式时，需要考虑的与上面的数组差不多：

*	尝试调整数据在线程间的分布，使得同一线程使用的数据相互靠近；
*	尝试最小化每个线程的数据量；
*	尝试确保不同线程使用的数据相互远离，以避免false sharing。

>Of course, that’s not easy to apply to other data structures. For example, binary trees are inherently difficult to subdivide in any unit other than a subtree, which may or may not be useful, depending on how balanced the tree is and how many sections you need to divide it into. Also, the nature of the trees means that the nodes are likely dynamically allocated and thus end up in different places on the heap.
>
>
>Now, having data end up in different places on the heap isn’t a particular problem in itself, but it does mean that the processor has to keep more things in cache. This can actually be beneficial. If multiple threads need to traverse the tree, then they all need to access the tree nodes, but if the tree nodes only contain pointers to the real data held at the node, then the processor only has to load the data from memory if it’s actually needed. If the data is being modified by the threads that need it, this can avoid the performance hit of false sharing between the node data itself and the data that provides the tree structure.
>
>
>There’s a similar issue with data protected by a mutex. Suppose you have a simple class that contains a few data items and a mutex used to protect accesses from multiple threads. If the mutex and the data items are close together in memory, this is ideal for a thread that acquires the mutex; the data it needs may well already be in the processor cache, because it was just loaded in order to modify the mutex. But there’s also a downside: if other threads try to lock the mutex while it’s held by the first thread, they’ll need access to that memory. Mutex locks are typically implemented as a readmodify-write atomic operation on a memory location within the mutex to try to acquire the mutex, followed by a call to the operating system kernel if the mutex is already locked. This read-modify-write operation may well cause the data held in the cache by the thread that owns the mutex to be invalidated. As far as the mutex goes, this isn’t a problem; that thread isn’t going to touch the mutex until it unlocks it. However, if the mutex shares a cache line with the data being used by the thread, the thread that owns the mutex can take a performance hit because another thread tried to lock the mutex!

<h3 id="additional_considerations_when_designing_for_concurrency">多线程设计时的其它注意事项</h3>

虽然我们已经讨论了很多并发设计需要注意的事项，但是作为一个好的并发代码。还需要考虑异常安全和可扩展性(scalability)。**如果一段代码的性能随着内核数量的增加而增加(一般呈线性趋势，即100个内核运行该代码的性能是一个内核运行该代码的性能的100倍)，那么称该代码是可扩展的(scalable)**。单线程代码一定不是可扩展的。

#### 并发算法中的异常安全

异常安全是一个好代码必不可少的部分，并发代码也不例外，**实际上并发算法比一般的序列算法更需要注意异常安全**。

如果一个序列算法抛出异常，它只需要考虑自身资源的清理以及异常的处理，它也可以传回给调用者让调用者进行处理；但是并发算法中很多操作都是在不同线程中的，所以异常是不允许传回给调用者的，因为它们在一个错误的调用栈中。如果一个新线程中的函数异常退出，那么应用程序将会终止(terminate)。

查看如下代码：

```c++
#include <vector>
#include <algorithm>
#include <numeric>
#include <thread>
#include <random>
#include <cstdlib>
#include <iostream>

template<typename Iterator, typename T>
struct accumulate_block
{
	void operator()(Iterator first, Iterator last, T& result)
	{
		// 注意std::accumulate第三个参数是值传递
		result = std::accumulate(first, last, result);
	}
};

template<typename Iterator, typename T>
T parallel_accumulate(Iterator first, Iterator last, T init)
{
	unsigned long const length = std::distance(first, last);
	if (!length)
		return init;

	unsigned long const min_per_thread = 25;
	unsigned long const max_threads =
		(length + min_per_thread - 1) / min_per_thread;
	unsigned long const hardware_threads =
		std::thread::hardware_concurrency();
	unsigned long const num_threads =
		std::min(hardware_threads != 0 ? hardware_threads : 2, max_threads);

	unsigned long const block_size = length / num_threads;
	std::vector<T> results(num_threads);
	std::vector<std::thread> threads(num_threads - 1);

	Iterator block_start = first;
	for (unsigned long i = 0; i<(num_threads - 1); ++i)
	{
		Iterator block_end = block_start;
		std::advance(block_end, block_size);
		threads[i] = std::thread(
			accumulate_block<Iterator, T>(),
			block_start, block_end, std::ref(results[i]));
		block_start = block_end;
	}
	accumulate_block<Iterator, T>()(block_start, last, results[num_threads - 1]);

	// std::mem_fn生成一个包装对象的成员函数的指针，在C++14中被移除
	/*std::for_each(threads.begin(), threads.end(),
		std::mem_fn(&std::thread::join));*/

	for (unsigned i = 0; i < threads.size(); ++i)
	{
		threads[i].join();
	}
	
	return std::accumulate(results.begin(), results.end(), init);
}

int main()
{
	// 随机数生成
	std::vector<int> nums;
	std::uniform_int_distribution<unsigned> u(0, 1000);
	std::default_random_engine e;
	for (int i = 0; i < 10000; ++i) {
		nums.push_back(u(e));
	}

	auto begin = nums.begin();
	auto end = nums.end();

	int sum1 = 0, sum2 = 0;

	auto time_start = std::chrono::high_resolution_clock::now();
	accumulate_block<decltype(begin),int>()(begin, end, sum1);
	auto time_end = std::chrono::high_resolution_clock::now();
	std::cout << "accumulate_block tooks "
		<< std::chrono::duration<double, std::milli>(time_end - time_start).count()
		<< " ms\n";

	time_start = std::chrono::high_resolution_clock::now();
	sum2 = parallel_accumulate<decltype(begin), int>(begin, end, sum2);
	time_end = std::chrono::high_resolution_clock::now();
	std::cout << "parallel_accumulate tooks " 
		<< std::chrono::duration<double, std::milli>(time_end - time_start).count()
		<< " ms\n";

	std::cout << "sum1 = " << sum1
		<< ", sum2 = " << sum2
		<< std::endl;
}
```

结果：

```text
accumulate_block tooks 0.260436 ms
parallel_accumulate tooks 0.851227 ms
sum1 = 5026673, sum2 = 5026673

```

**当你调用一个你知道的可能会发生异常的函数，或者调用了一个用户自定义的函数时，可能发生异常**：

*	在`block_start`初始化之前的所有操作都是异常安全的，因为你除了初始化之外什么也没有做，而且这些操作全部都在调用线程上运行。
*	一旦创建的线程发生了异常，由于thread的析构函数在没有调用`detach`或`join`的情况下会调用`terminate`，所以程序将会终止；
*	在`accumulate_block`中的`std::accumulate`也可能抛出异常，因为没有做catch处理。

在基础篇我们知道，`std::future`是可以存储异常的，所以我们用`std::future`来重构上面的代码：

```c++
#include <vector>
#include <algorithm>
#include <numeric>
#include <thread>
#include <future>
#include <random>
#include <cstdlib>
#include <iostream>

template<typename Iterator, typename T>
struct accumulate_block
{
	T operator()(Iterator first, Iterator last)
	{
		return std::accumulate(first, last, T());
	}
};

template<typename Iterator, typename T>
T parallel_accumulate(Iterator first, Iterator last, T init)
{
	unsigned long const length = std::distance(first, last);

	if (!length)
		return init;

	unsigned long const min_per_thread = 25;
	unsigned long const max_threads =
		(length + min_per_thread - 1) / min_per_thread;

	unsigned long const hardware_threads =
		std::thread::hardware_concurrency();

	unsigned long const num_threads =
		std::min(hardware_threads != 0 ? hardware_threads : 2, max_threads);

	unsigned long const block_size = length / num_threads;

	std::vector<std::future<T> > futures(num_threads - 1);
	std::vector<std::thread> threads(num_threads - 1);

	Iterator block_start = first;
	T last_result = 0;
	try
	{
		for (unsigned long i = 0; i<(num_threads - 1); ++i)
		{
			Iterator block_end = block_start;
			std::advance(block_end, block_size);
			// !!!!!!特别注意，如果参数不加括号会被当做函数声明，你也可以使用{}来初始化
			std::packaged_task<T(Iterator, Iterator)> task(
				(accumulate_block<Iterator, T>()));
			futures[i] = task.get_future();
			threads[i] = std::thread(std::move(task), block_start, block_end);
			block_start = block_end;
		}
		last_result = accumulate_block<Iterator, T>()(block_start, last);

		std::for_each(threads.begin(), threads.end(),
			std::mem_fn(&std::thread::join));
	}
	catch (...)
	{
		for (unsigned long i = 0; i<(num_threads - 1); ++i)
		{
			if (threads[i].joinable())
				threads[i].join();
		}
		throw; // throw之后，后面的代码将不再运行
	}

	T result = init;
	for (unsigned long i = 0; i<(num_threads - 1); ++i)
	{
		// 如果工作线程抛出异常，将在主线程的这里重新抛出
		result += futures[i].get();
	}
	result += last_result;
	return result;
}

int main()
{
	// 随机数生成
	std::vector<int> nums;
	std::uniform_int_distribution<unsigned> u(0, 1000);
	std::default_random_engine e;
	for (int i = 0; i < 10000; ++i) {
		nums.push_back(u(e));
	}

	auto begin = nums.begin();
	auto end = nums.end();

	int sum1 = 0, sum2 = 0;

	auto time_start = std::chrono::high_resolution_clock::now();
	sum1 = accumulate_block<decltype(begin), int>()(begin, end);
	auto time_end = std::chrono::high_resolution_clock::now();
	std::cout << "accumulate_block tooks "
		<< std::chrono::duration<double, std::milli>(time_end - time_start).count()
		<< " ms\n";

	time_start = std::chrono::high_resolution_clock::now();
	try {
		sum2 = parallel_accumulate<decltype(begin), int>(begin, end, sum2);
	}
	catch (std::exception &e) {
		std::cout << e.what();
	}
	time_end = std::chrono::high_resolution_clock::now();
	std::cout << "parallel_accumulate tooks "
		<< std::chrono::duration<double, std::milli>(time_end - time_start).count()
		<< " ms\n";

	std::cout << "sum1 = " << sum1
		<< ", sum2 = " << sum2
		<< std::endl;
}
```

结果如下：

```text
accumulate_block tooks 0.519273 ms
parallel_accumulate tooks 1.22682 ms
sum1 = 5026673, sum2 = 5026673

```

上面的`parallel_accumulate`函数的try-catch语句太丑了，而且你在正常控制语句和catch控制语句中写了重复的代码，这往往是不好的，因为如果你要修改的话就需要修改多个地方。C++ 处理这种问题的惯用方法是将重复的代码写在析构函数里：

```c++
#include <vector>
#include <algorithm>
#include <numeric>
#include <thread>
#include <future>
#include <random>
#include <cstdlib>
#include <iostream>

template<typename Iterator, typename T>
struct accumulate_block
{
	T operator()(Iterator first, Iterator last)
	{
		return std::accumulate(first, last, T());
	}
};

class join_threads
{
	std::vector<std::thread>& threads;

public:
	explicit join_threads(std::vector<std::thread>& threads_) :
		threads(threads_)
	{}

	~join_threads()
	{
		for (unsigned long i = 0; i<threads.size(); ++i)
		{
			if (threads[i].joinable())
				threads[i].join();
		}
	}
};

template<typename Iterator, typename T>
T parallel_accumulate(Iterator first, Iterator last, T init)
{
	unsigned long const length = std::distance(first, last);

	if (!length)
		return init;

	unsigned long const min_per_thread = 25;
	unsigned long const max_threads =
		(length + min_per_thread - 1) / min_per_thread;

	unsigned long const hardware_threads =
		std::thread::hardware_concurrency();

	unsigned long const num_threads =
		std::min(hardware_threads != 0 ? hardware_threads : 2, max_threads);

	unsigned long const block_size = length / num_threads;

	std::vector<std::future<T> > futures(num_threads - 1);
	std::vector<std::thread> threads(num_threads - 1);
	join_threads joiner(threads);

	Iterator block_start = first;
	for (unsigned long i = 0; i<(num_threads - 1); ++i)
	{
		Iterator block_end = block_start;
		std::advance(block_end, block_size);
		// !!!!!!特别注意，如果参数不加括号会被当做函数声明，你也可以使用{}来初始化
		std::packaged_task<T(Iterator, Iterator)> task(
			(accumulate_block<Iterator, T>()));
		futures[i] = task.get_future();
		threads[i] = std::thread(std::move(task), block_start, block_end);
		block_start = block_end;
	}
	T last_result = accumulate_block<Iterator, T>()(block_start, last);

	std::for_each(threads.begin(), threads.end(),
		std::mem_fn(&std::thread::join));

	T result = init;
	for (unsigned long i = 0; i<(num_threads - 1); ++i)
	{
		// 如果工作线程抛出异常，将在主线程的这里重新抛出
		// 因为future.get()会阻塞直到其状态为ready，所以不用显式join
		result += futures[i].get();
	}
	result += last_result;
	return result;
}

int main()
{
	// 随机数生成
	std::vector<int> nums;
	std::uniform_int_distribution<unsigned> u(0, 1000);
	std::default_random_engine e;
	for (int i = 0; i < 10000; ++i) {
		nums.push_back(u(e));
	}

	auto begin = nums.begin();
	auto end = nums.end();

	int sum1 = 0, sum2 = 0;

	auto time_start = std::chrono::high_resolution_clock::now();
	sum1 = accumulate_block<decltype(begin), int>()(begin, end);
	auto time_end = std::chrono::high_resolution_clock::now();
	std::cout << "accumulate_block tooks "
		<< std::chrono::duration<double, std::milli>(time_end - time_start).count()
		<< " ms\n";

	time_start = std::chrono::high_resolution_clock::now();
	try {
		sum2 = parallel_accumulate<decltype(begin), int>(begin, end, sum2);
	}
	catch (std::exception &e) {
		std::cout << e.what();
	}
	time_end = std::chrono::high_resolution_clock::now();
	std::cout << "parallel_accumulate tooks "
		<< std::chrono::duration<double, std::milli>(time_end - time_start).count()
		<< " ms\n";

	std::cout << "sum1 = " << sum1
		<< ", sum2 = " << sum2
		<< std::endl;
}
```

结果如下：

```text
accumulate_block tooks 0.286739 ms
parallel_accumulate tooks 0.853801 ms
sum1 = 5026673, sum2 = 5026673

```

#### 使用`std::async`处理异常

在[基础篇](https://chorior.github.io/2017/04/24/C++-%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%9F%BA%E7%A1%80%E7%AF%87/#functional_program_with_future)中，我们知道**`std::async`确保线程数不会过载**，所以`std::async`很适合用来做并发递归：

```c++
#include <vector>
#include <algorithm>
#include <numeric>
#include <thread>
#include <future>
#include <random>
#include <cstdlib>
#include <iostream>

template<typename Iterator, typename T>
struct accumulate_block
{
	T operator()(Iterator first, Iterator last)
	{
		return std::accumulate(first, last, T());
	}
};

class join_threads
{
	std::vector<std::thread>& threads;

public:
	explicit join_threads(std::vector<std::thread>& threads_) :
		threads(threads_)
	{}

	~join_threads()
	{
		for (unsigned long i = 0; i<threads.size(); ++i)
		{
			if (threads[i].joinable())
				threads[i].join();
		}
	}
};

template<typename Iterator, typename T>
T parallel_accumulate(Iterator first, Iterator last, T init)
{
	unsigned long const length = std::distance(first, last);
	unsigned long const max_chunk_size = 25;
	if (length <= max_chunk_size)
	{
		return std::accumulate(first, last, init);
	}
	else
	{
		Iterator mid_point = first;
		std::advance(mid_point, length / 2);

		// 如果async线程发生异常，在get()时会重新抛出
		std::future<T> first_half_result =
			std::async(parallel_accumulate<Iterator, T>,
				first, mid_point, init);

		// 如果这里抛出异常，future的析构函数会等待其线程完成
		T second_half_result = parallel_accumulate(mid_point, last, T());
		return first_half_result.get() + second_half_result;
	}
}

int main()
{
	// 随机数生成
	std::vector<int> nums;
	std::uniform_int_distribution<unsigned> u(0, 1000);
	std::default_random_engine e;
	for (int i = 0; i < 10000; ++i) {
		nums.push_back(u(e));
	}

	auto begin = nums.begin();
	auto end = nums.end();

	int sum1 = 0, sum2 = 0;

	auto time_start = std::chrono::high_resolution_clock::now();
	sum1 = accumulate_block<decltype(begin), int>()(begin, end);
	auto time_end = std::chrono::high_resolution_clock::now();
	std::cout << "accumulate_block tooks "
		<< std::chrono::duration<double, std::milli>(time_end - time_start).count()
		<< " ms\n";

	time_start = std::chrono::high_resolution_clock::now();
	try {
		sum2 = parallel_accumulate<decltype(begin), int>(begin, end, sum2);
	}
	catch (std::exception &e) {
		std::cout << e.what();
	}
	time_end = std::chrono::high_resolution_clock::now();
	std::cout << "parallel_accumulate tooks "
		<< std::chrono::duration<double, std::milli>(time_end - time_start).count()
		<< " ms\n";

	std::cout << "sum1 = " << sum1
		<< ", sum2 = " << sum2
		<< std::endl;
}
```

结果：

```text
accumulate_block tooks 0.39162 ms
parallel_accumulate tooks 39.3281 ms
sum1 = 5026673, sum2 = 5026673

```

#### 可扩展性和 Amdahl 定律

可扩展性就是你的代码利用处理器数量改善性能的能力，单线程代码永远不能利用处理器的数量。理想情况下，每个线程每时每刻都在做有用的工作，但实际上这是不可能的，因为**线程间经常花费时间相互等待或等待I/O操作完成**。

一个简单的看待可扩展性的方法是：将程序划分为“串行”部分和“并行”部分，“串行”部分只有一个线程在做有用的工作，“并行”部分所有可用的处理器都在做有用的工作；所以如果程序运行在一个拥有较多处理器的系统上时，毫无疑问“并行”部分会完成的更加快速。假设“串行”部分占整个程序的比例为Fs，那么在带有N个处理器的系统上运行该程序的性能增益为：

```c++
P = 1/(Fs + (1 - Fs)/N)
```

上面的公式就是Amdahl 定律，**这个定律经常在讨论并发代码性能时被引用**，由该定律可知，**并发程序的性能绝对不会超过1/Fs**。

所以**减少“串行”部分的大小，或者减少线程等待的时间，就能更好的改善程序的性能**。如果我们在线程等待的时候做一些有用的事情，就能将这个等待“隐藏”掉。

阻塞的线程相当于什么都没做，但这浪费了CPU时间。**如果你知道一个线程可能花费相当长的时间等待，那么你可以通过运行一个或多个线程来利用和这个时间**。

例如，如果一个线程正在等待一个I/O操作完成，那么你可以使用异步I/O来进行这个操作，这样线程就可以执行其它有用的工作，同时后台运行着I/O操作；如果一个线程正在等待另一个线程完成一个操作，那么你可以尝试自己完成那个操作；如果一个线程正在等待一个任务被完成，但是这个任务还没有被启动，那么你可以尝试在本线程完成这个任务或者做其它没有开始的任务，就像[这里](#data_based_work_division)的`do_sort`一样。

**有时候添加线程并不是为了利用所有可用的处理器，而是为了对外部事件进行及时的响应**。

<h3 id="improving_responsiveness_with_concurrency">多线程改善响应能力</h3>

大多数图形化用户接口框架都是事件驱动型(event driven)，其主循环很可能与下面类似：

```c++
while (true)
{
	event_data event = get_event();
	if (event.type == quit)
		break;
	process(event);
}
```

为了确保及时响应用户输入，`get_event`和`process`必须以合理的频率被调用，不管程序在做什么。在[基于任务类型的工作划分](#task_type_based_work_division)第一节中，我们了解了单一线程及时处理用户输入是会大大增加代码的复杂性的，通过分离关注点，我们可以将处理用户输入与任务执行放在不同的线程中，下面的示例没有考虑线程数过载，且只具有参考价值：

```c++
std::thread task_thread;
std::atomic<bool> task_cancelled(false);
void gui_thread()
{
	while (true)
	{
		event_data event = get_event();
		if (event.type == quit)
			break;
		process(event);
	}
}

void task()
{
	while (!task_complete() && !task_cancelled)
	{
		do_next_operation();
	}
	if (task_cancelled)
	{
		perform_cleanup();
	}
	else
	{
		post_gui_event(task_complete);
	}
}

void process(event_data const& event)
{
	switch (event.type)
	{
	case start_task:
		task_cancelled = false;
		task_thread = std::thread(task);
		break;
	case stop_task:
		task_cancelled = true;
		task_thread.join();
		break;
	case task_complete:
		task_thread.join();
		display_results();
		break;
	default:
		//...
	}
}
```

<h3 id="std_for_each">std::for_each</h3>

`std::for_each`对范围内每个元素调用同一个函数，其并行版与序列版唯一的区别就是元素调用函数的顺序。要实现一个`std::for_each`的并行版本，只需要基于数据的工作划分，即每个线程做一定数量的元素的工作。

*	假设系统只有这一个多线程程序在运行，所以你可以使用`std::thread::hardware_concurrency()`来决定线程的数量；
*	你知道元素的数量在工作开始前就能得到，所以你可以在工作开始前就将数据划分好；
*	你也知道每个线程肯定是相互独立的，所以你可以使用连续的数据块来避免伪共享(false sharing)。

一个简单的实现如下，为了体现元素操作的顺序，借用[threadsafe_list](https://chorior.github.io/2017/05/24/C++-%E5%A4%9A%E7%BA%BF%E7%A8%8B%E8%AE%BE%E8%AE%A1-%E4%B8%80/#linked_list)来保存操作的顺序：

```c++
#include <vector>
#include <algorithm>
#include <numeric>
#include <thread>
#include <future>
#include <random>
#include <cstdlib>
#include <iostream>

#include "myLinkedList.h"

class join_threads
{
	std::vector<std::thread>& threads;

public:
	explicit join_threads(std::vector<std::thread>& threads_) :
		threads(threads_)
	{}

	~join_threads()
	{
		for (unsigned long i = 0; i<threads.size(); ++i)
		{
			if (threads[i].joinable())
				threads[i].join();
		}
	}
};

template<typename Iterator, typename Func>
void parallel_for_each(Iterator first, Iterator last, Func f)
{
	unsigned long const length = std::distance(first, last);
	if (!length)
		return;

	unsigned long const min_per_thread = 25;
	unsigned long const max_threads =
		(length + min_per_thread - 1) / min_per_thread;
	unsigned long const hardware_threads =
		std::thread::hardware_concurrency();
	unsigned long const num_threads =
		std::min(hardware_threads != 0 ? hardware_threads : 2, max_threads);
	unsigned long const block_size = length / num_threads;

	std::vector<std::future<void> > futures(num_threads - 1);
	std::vector<std::thread> threads(num_threads - 1);
	join_threads joiner(threads);

	Iterator block_start = first;
	for (unsigned long i = 0; i<(num_threads - 1); ++i)
	{
		Iterator block_end = block_start;
		std::advance(block_end, block_size);
		std::packaged_task<void(void)> task(
			[=]()
		{
			std::for_each(block_start, block_end, f);
		});
		futures[i] = task.get_future();
		threads[i] = std::thread(std::move(task)); // task是一个可调用对象
		block_start = block_end;
	}
	std::for_each(block_start, last, f);

	for (unsigned long i = 0; i<(num_threads - 1); ++i)
	{
		futures[i].get();
	}
}

int main()
{
	std::vector<int> nums;
	for (int i = 0; i < 51; ++i) {
		nums.push_back(i);
	}

	auto begin = nums.begin();
	auto end = nums.end();

	threadsafe_list<int> order;
	auto func = [&order](int &i)
	{ 
		order.push_front(i);
	};

	auto time_start = std::chrono::high_resolution_clock::now();
	std::for_each(begin, end, func);
	auto time_end = std::chrono::high_resolution_clock::now();
	std::cout << "for_each tooks "
		<< std::chrono::duration<double, std::milli>(time_end - time_start).count()
		<< " ms\n";

	order.for_each([](int i) {std::cout << i << std::ends; });
	std::cout << "\n";
	order.remove_if([](int const&) {return true; });

	time_start = std::chrono::high_resolution_clock::now();
	try {
		parallel_for_each<decltype(begin), decltype(func)>(begin, end, func);
	}
	catch (std::exception &e) {
		std::cout << e.what();
	}
	time_end = std::chrono::high_resolution_clock::now();
	std::cout << "parallel_for_each tooks "
		<< std::chrono::duration<double, std::milli>(time_end - time_start).count()
		<< " ms\n";

	order.for_each([](int i) {std::cout << i << std::ends; });
}
```

结果如下：

```text
for_each tooks 0.228043 ms
50 49 48 47 46 45 44 43 42 41 40 39 38 37 36 35 34 33 32 31 30 29 28 27 26 25 24 23 22 21 20 19 18 17 16 15 14 13 12 11 10 9 8 7 6 5 4 3 2 1 0
parallel_for_each tooks 0.829425 ms
50 33 49 32 48 31 47 30 46 29 45 28 44 27 43 26 42 25 41 24 40 23 39 22 38 21 37 20 36 19 35 18 34 17 16 15 14 13 12 11 10 9 8 7 6 5 4 3 2 1 0
```

再用`std::async`来实现一下：

```c++
template<typename Iterator, typename Func>
void parallel_for_each(Iterator first, Iterator last, Func f)
{
	unsigned long const length = std::distance(first, last);
	if (!length)
		return;

	unsigned long const min_per_thread = 25;
	if (length<(2 * min_per_thread))
	{
		std::for_each(first, last, f);
	}
	else
	{
		Iterator const mid_point = first + length / 2;
		std::future<void> first_half =
			std::async(&parallel_for_each<Iterator, Func>,
				first, mid_point, f);
		parallel_for_each(mid_point, last, f);
		first_half.get();
	}
}
```

结果如下：

```text
for_each tooks 0.307587 ms
50 49 48 47 46 45 44 43 42 41 40 39 38 37 36 35 34 33 32 31 30 29 28 27 26 25 24 23 22 21 20 19 18 17 16 15 14 13 12 11 10 9 8 7 6 5 4 3 2 1 0
parallel_for_each tooks 0.643719 ms
24 23 22 21 20 19 18 17 16 15 14 13 12 11 10 9 8 7 6 5 4 3 50 2 49 1 48 0 47 46 45 44 43 42 41 40 39 38 37 36 35 34 33 32 31 30 29 28 27 26 25
```

<h3 id="std_find">std::find</h3>

`std::find`并不一定对所有元素都做一次操作，所以如果已经找到目标，就应该中断其它线程。一种中断其它线程的方法是设置一个原子flag变量，然后在每次操作完成之后对其进行检查。为了返回值，以及传递可能发生的异常，你可以使用`std::packaged_task`或`std::promise`，两者的区别就是：`std::packaged_task`可以存储所有发生的异常，如果有一个线程发生了异常，其它线程还能继续执行，但是`std::promise`只能存储一个异常，只要有一个线程发生了异常，就会中断所有执行。

虽然`std::packaged_task`可以存储所有发生的异常，但实际上只能有一个异常能被抛出，所以我们选择`std::promise`来实现`std::find`的并行算法。这里一个比较重要的点是：如果找不到目标，那么`future.get()`将永远阻塞下去，所以在调用`get()`之前，需要等待所有线程结束，并对flag进行检查。

```c++
#include <vector>
#include <algorithm>
#include <numeric>
#include <thread>
#include <future>
#include <random>
#include <cstdlib>
#include <iostream>

class join_threads
{
	std::vector<std::thread>& threads;

public:
	explicit join_threads(std::vector<std::thread>& threads_) :
		threads(threads_)
	{}

	~join_threads()
	{
		for (unsigned long i = 0; i<threads.size(); ++i)
		{
			if (threads[i].joinable())
				threads[i].join();
		}
	}
};

template<typename Iterator, typename MatchType>
Iterator parallel_find(Iterator first, Iterator last, MatchType match)
{
	struct find_element
	{
		void operator()(Iterator begin, Iterator end,
			MatchType match,
			std::promise<Iterator>* result,
			std::atomic<bool>* done_flag)
		{
			try
			{
				for (; (begin != end) && !done_flag->load(); ++begin)
				{
					if (*begin == match)
					{
						// 如果有多个匹配的值，那么这里会发生竞争，
						// 但是没关系，不管哪个值被设置，都是正确的值
						// 同理，下面的异常设置
						result->set_value(begin);
						done_flag->store(true);
						return;
					}
				}
			}
			catch (...)
			{
				try
				{
					result->set_exception(std::current_exception());
					done_flag->store(true);
				}
				catch (...)
				{
				}
			}
		}
	};

	unsigned long const length = std::distance(first, last);
	if (!length)
		return last;

	unsigned long const min_per_thread = 25;
	unsigned long const max_threads =
		(length + min_per_thread - 1) / min_per_thread;
	unsigned long const hardware_threads =
		std::thread::hardware_concurrency();
	unsigned long const num_threads =
		std::min(hardware_threads != 0 ? hardware_threads : 2, max_threads);
	unsigned long const block_size = length / num_threads;

	std::promise<Iterator> result;
	std::atomic<bool> done_flag(false);
	std::vector<std::thread> threads(num_threads - 1);

	// 代码块的作用是控制join_threads的生命周期
	{
		join_threads joiner(threads);
		Iterator block_start = first;
		for (unsigned long i = 0; i<(num_threads - 1); ++i)
		{
			Iterator block_end = block_start;
			std::advance(block_end, block_size);
			threads[i] = std::thread(find_element(),
				block_start, block_end, match,
				&result, &done_flag);
			block_start = block_end;
		}
		find_element()(block_start, last, match, &result, &done_flag);
	}

	if (!done_flag.load())
	{
		return last;
	}
	return result.get_future().get();
}

int main()
{
	std::vector<int> nums;
	for (int i = 0; i < 1000; ++i) {
		nums.push_back(i);
	}

	auto begin = nums.begin();
	auto end = nums.end();
	int goal = 501;

	decltype(begin) result1 = end;
	decltype(begin) result2 = end;

	auto time_start = std::chrono::high_resolution_clock::now();
	result1 = std::find(begin, end, goal);
	auto time_end = std::chrono::high_resolution_clock::now();
	std::cout << "for_each tooks "
		<< std::chrono::duration<double, std::milli>(time_end - time_start).count()
		<< " ms\n";

	time_start = std::chrono::high_resolution_clock::now();
	try {
		result2 = parallel_find<decltype(begin), decltype(goal)>(begin, end, goal);
	}
	catch (std::exception &e) {
		std::cout << e.what();
	}
	time_end = std::chrono::high_resolution_clock::now();
	std::cout << "parallel_for_each tooks "
		<< std::chrono::duration<double, std::milli>(time_end - time_start).count()
		<< " ms\n";

	if (end != result1) { std::cout << *result1 << std::endl; }
	if (end != result2) { std::cout << *result2 << std::endl; }
}
```

结果：

```text
for_each tooks 0.014433 ms
parallel_for_each tooks 0.88844 ms
501
501

```

再用`std::async`实现：

```c++
template<typename Iterator, typename MatchType>
Iterator parallel_find_impl(Iterator first, Iterator last, MatchType match,
	std::atomic<bool>& done)
{
	try
	{
		unsigned long const length = std::distance(first, last);
		unsigned long const min_per_thread = 25;
		if (length<(2 * min_per_thread))
		{
			for (; (first != last) && !done.load(); ++first)
			{
				if (*first == match)
				{
					done = true;
					return first;
				}
			}
			return last;
		}
		else
		{
			Iterator const mid_point = first + (length / 2);
			std::future<Iterator> async_result =
				std::async(&parallel_find_impl<Iterator, MatchType>,
					mid_point, last, match, std::ref(done));
			Iterator const direct_result =
				parallel_find_impl(first, mid_point, match, done);
			return (direct_result == mid_point) ?
				async_result.get() : direct_result;
		}
	}
	catch (...)
	{
		done = true;
		throw;
	}
}

template<typename Iterator, typename MatchType>
Iterator parallel_find(Iterator first, Iterator last, MatchType match)
{
	std::atomic<bool> done(false);
	// 为了递归调用且有一个全局flag，所以需要另起一个函数
	return parallel_find_impl(first, last, match, done);
}
```

结果：

```text
for_each tooks 0.03047 ms
parallel_for_each tooks 3.08998 ms
501
501

```

<h3 id="std_partial_sum">std::partial_sum</h3>

```c++
template<class _InIt,
	class _OutIt,
	class _Fn2> inline
	_OutIt partial_sum(_InIt _First, _InIt _Last,
		_OutIt _Dest, _Fn2 _Func)
	{	// compute partial sums into _Dest, using _Func
	_DEPRECATE_UNCHECKED(partial_sum, _Dest);
	return (_Partial_sum_no_deprecate(_First, _Last, _Dest, _Func));
	}
```

查看`partial_sum`的[官方说明文档](http://en.cppreference.com/w/cpp/algorithm/partial_sum)，其功能简要的概括一下就是：对范围内的每个元素进行叠加，但是每加一个元素，就保存一个值，保存的开始位置在第三个参数中给出，第四个参数可以指定其他的二元算法。

要想实现该算法的并行版本，由于各个线程的数据不是相互独立的，例如第一个元素在每个线程都被使用了，所以就不能像上面的`for_each`和`find`一样进行简单的划分了。假设你有一组数据{1,2,3,4,5,6,7,8,9}，你将其分为三个小块，{1,2,3}、{4,5,6}、{7,8,9}，对这三个小块分别进行`partial_sum`，结果将是{1,3,6}、{4,9,15}、{7,15,24}，合并结果为{1,3,6,4,9,15,7,15,24}，很明显正确结果应该是{1,3,6,10,15,21,28,36,45}，怎样才能获得正确结果呢？如果能将第一小块的最后一个结果添加到第二个小块的所有结果上，再将第二个小块的最后一个结果添加到第三个小块的左右结果上，结果就正确了。

如果每个小块都先更新最后一个结果，那么计算每个小块剩余元素的线程就可以与下一个小块线程同时运行了。但是如果你的总元素个数太少或者处理器数量太多的话，后面的处理器肯定需要等待前面的处理器返回结果才能开始运行，这是不好的。换一种传递方式，你首先将相邻元素求和，结果为{1,3,5,7,9,11,13,15,17}，这个结果对于两个元素是完全正确的；然后将这些结果两个两个相加，结果为{1,3,6,10,14,15,22,26,30}，这个结果对于前四个元素也是完全正确的；在将结果四个四个相加，结果为{1,3,6,10,15,18,28,36,44}，该结果对于前八个元素是完全正确的；接着是{1,3,6,10,15,18,28,36,45}，该结果就是最终结果。

如果一共有N个元素、k个线程，那么使用第一种方法，每个线程需要做一次计算最后一个结果的操作，次数为N/k，然后再做一次值的传递操作，次数也是N/k；使用第二种方法需要做log2(N)次N个操作，所以第一种方法的复杂度是O(N)，第二种方法的复杂度是Nlog2(N)，然而如果你有N个处理器，那么第二种方法会使得每个处理器只有log2(N)个操作，而第一种方法在K变得足够大时，会序列化绝大多数操作。

下面实现了第一种算法：

```c++
#include <vector>
#include <algorithm>
#include <numeric>
#include <thread>
#include <future>
#include <random>
#include <cstdlib>
#include <iostream>
#include <iterator>

class join_threads
{
	std::vector<std::thread>& threads;

public:
	explicit join_threads(std::vector<std::thread>& threads_) :
		threads(threads_)
	{}

	~join_threads()
	{
		for (unsigned long i = 0; i<threads.size(); ++i)
		{
			if (threads[i].joinable())
				threads[i].join();
		}
	}
};

template<typename Iterator>
void parallel_partial_sum(Iterator first, Iterator last)
{
	typedef typename Iterator::value_type value_type;
	struct process_chunk
	{
		void operator()(Iterator begin, Iterator last,
			std::future<value_type>* previous_end_value,
			std::promise<value_type>* end_value)
		{
			try
			{
				Iterator end = last;
				++end;
				std::partial_sum(begin, end, begin);
				if (previous_end_value)
				{
					value_type addend = previous_end_value->get();
					*last += addend;
					if (end_value)
					{
						end_value->set_value(*last);
					}
					std::for_each(begin, last, [addend](value_type& item)
					{
						item += addend;
					});
				}
				else if (end_value)
				{
					end_value->set_value(*last);
				}
			}
			catch (...)
			{
				if (end_value)
				{
					end_value->set_exception(std::current_exception());
				}
				else
				{
					throw;
				}
			}
		}
	};

	unsigned long const length = std::distance(first, last);
	if (!length)
		return ;

	unsigned long const min_per_thread = 25;
	unsigned long const max_threads =
		(length + min_per_thread - 1) / min_per_thread;
	unsigned long const hardware_threads =
		std::thread::hardware_concurrency();
	unsigned long const num_threads =
		std::min(hardware_threads != 0 ? hardware_threads : 2, max_threads);
	unsigned long const block_size = length / num_threads;

	std::vector<std::thread> threads(num_threads - 1);
	std::vector<std::promise<value_type> > end_values(num_threads - 1);
	std::vector<std::future<value_type> > previous_end_values;

	// 分配至少能容纳num_threads - 1个元素的内存空间
	// 可以避免在push_back时重新分配内存，因为你知道需要多大的空间
	previous_end_values.reserve(num_threads - 1);

	join_threads joiner(threads);
	Iterator block_start = first;
	for (unsigned long i = 0; i<(num_threads - 1); ++i)
	{
		Iterator block_last = block_start;
		std::advance(block_last, block_size - 1);
		threads[i] = std::thread(process_chunk(),
			block_start, block_last,
			(i != 0) ? &previous_end_values[i - 1] : 0,
			&end_values[i]);
		block_start = block_last;
		++block_start;
		previous_end_values.push_back(end_values[i].get_future());
	}
	Iterator final_element = block_start;
	std::advance(final_element, std::distance(block_start, last) - 1);
	process_chunk()(block_start, final_element,
		(num_threads>1) ? &previous_end_values.back() : 0,
		0);
}

int main()
{
	std::vector<int> nums;
	for (int i = 0; i < 51; ++i) {
		nums.push_back(i);
	}

	auto begin = nums.begin();
	auto end = nums.end();
	std::vector<int> result(nums.size());

	auto time_start = std::chrono::high_resolution_clock::now();
	std::partial_sum(begin, end, result.begin());
	auto time_end = std::chrono::high_resolution_clock::now();
	std::cout << "partial_sum tooks "
		<< std::chrono::duration<double, std::milli>(time_end - time_start).count()
		<< " ms\n";

	time_start = std::chrono::high_resolution_clock::now();
	try {
		parallel_partial_sum<decltype(begin)>(begin, end);
	}
	catch (std::exception &e) {
		std::cout << e.what() << "\n";
	}
	time_end = std::chrono::high_resolution_clock::now();
	std::cout << "parallel_partial_sum tooks "
		<< std::chrono::duration<double, std::milli>(time_end - time_start).count()
		<< " ms\n";

	for (auto &i : result)
	{
		std::cout << i << " ";
	}
	std::cout << "\n";
	for (auto &i : nums)
	{
		std::cout << i << " ";
	}
}
```

结果：

```text
partial_sum tooks 0.027584 ms
parallel_partial_sum tooks 0.886516 ms
0 1 3 6 10 15 21 28 36 45 55 66 78 91 105 120 136 153 171 190 210 231 253 276 300 325 351 378 406 
435 465 496 528 561 595 630 666 703 741 780 820 861 903 946 990 1035 1081 1128 1176 1225 1275
0 1 3 6 10 15 21 28 36 45 55 66 78 91 105 120 136 153 171 190 210 231 253 276 300 325 351 378 406 
435 465 496 528 561 595 630 666 703 741 780 820 861 903 946 990 1035 1081 1128 1176 1225 1275
```

在对`previous_end_values`进行操作的时候，使用`push_back`而非以往的赋值操作，是为了让代码更加清晰，更容易理解，当然你也可以写成下面这样：

```c++
...
std::vector<std::future<value_type> > previous_end_values(num_threads - 1);
// previous_end_values.reserve(num_threads - 1);
...
for (unsigned long i = 0; i<(num_threads - 1); ++i)
{
	...
	previous_end_values[i] = end_values[i].get_future();
}
```

在做最后一块时，也可以简化为：

```c++
process_chunk()(block_start, last - 1,
		(num_threads>1) ? &previous_end_values.back() : 0,
		0);
```

第二种方法的实现：

```c++
#include <vector>
#include <algorithm>
#include <numeric>
#include <thread>
#include <future>
#include <random>
#include <cstdlib>
#include <iostream>
#include <iterator>

class join_threads
{
	std::vector<std::thread>& threads;

public:
	explicit join_threads(std::vector<std::thread>& threads_) :
		threads(threads_)
	{}

	~join_threads()
	{
		for (unsigned long i = 0; i<threads.size(); ++i)
		{
			if (threads[i].joinable())
				threads[i].join();
		}
	}
};

class barrier
{
	std::atomic<unsigned> count;
	std::atomic<unsigned> spaces;
	std::atomic<unsigned> generation;

public:
	explicit barrier(unsigned count_) :
		count(count_), spaces(count_), generation(0)
	{}

	void wait()
	{
		unsigned const my_generation = generation;
		if (!--spaces)
		{
			spaces = count.load();
			++generation;
		}
		else
		{
			while (generation == my_generation)
				std::this_thread::yield();
		}
	}

	void done_waiting()
	{
		--count;
		if (!--spaces)
		{
			spaces = count.load();
			++generation;
		}
	}
};

template<typename Iterator>
void parallel_partial_sum(Iterator first, Iterator last)
{
	typedef typename Iterator::value_type value_type;
	struct process_element
	{
		void operator()(Iterator first, Iterator last,
			std::vector<value_type>& buffer,
			unsigned i, barrier& b)
		{
			value_type& ith_element = *(first + i);
			bool update_source = false;
			for (unsigned step = 0, stride = 1; stride <= i; ++step, stride *= 2)
			{
				value_type const& source = (step % 2) ?
					buffer[i] : ith_element;
				value_type& dest = (step % 2) ?
					ith_element : buffer[i];
				value_type const& addend = (step % 2) ?
					buffer[i - stride] : *(first + i - stride);
				dest = source + addend;
				update_source = !(step % 2);
				b.wait();
			}
			if (update_source)
			{
				ith_element = buffer[i];
			}
			b.done_waiting();
		}
	};

	unsigned long const length = std::distance(first, last);
	if (length <= 1)
		return;

	std::vector<value_type> buffer(length);
	barrier b(length);
	std::vector<std::thread> threads(length - 1);
	join_threads joiner(threads);
	Iterator block_start = first;
	for (unsigned long i = 0; i<(length - 1); ++i)
	{
		threads[i] = std::thread(process_element(), first, last,
			std::ref(buffer), i, std::ref(b));
	}
	process_element()(first, last, buffer, length - 1, b);
}

int main()
{
	std::vector<int> nums;
	for (int i = 0; i < 51; ++i) {
		nums.push_back(i);
	}

	auto begin = nums.begin();
	auto end = nums.end();
	std::vector<int> result(nums.size());

	auto time_start = std::chrono::high_resolution_clock::now();
	std::partial_sum(begin, end, result.begin());
	auto time_end = std::chrono::high_resolution_clock::now();
	std::cout << "partial_sum tooks "
		<< std::chrono::duration<double, std::milli>(time_end - time_start).count()
		<< " ms\n";

	time_start = std::chrono::high_resolution_clock::now();
	try {
		parallel_partial_sum<decltype(begin)>(begin, end);
	}
	catch (std::exception &e) {
		std::cout << e.what() << "\n";
	}
	time_end = std::chrono::high_resolution_clock::now();
	std::cout << "parallel_partial_sum tooks "
		<< std::chrono::duration<double, std::milli>(time_end - time_start).count()
		<< " ms\n";

	for (auto &i : result)
	{
		std::cout << i << " ";
	}
	std::cout << "\n";
	for (auto &i : nums)
	{
		std::cout << i << " ";
	}
}
```

结果：

```text
partial_sum tooks 0.041375 ms
parallel_partial_sum tooks 13.122 ms
0 1 3 6 10 15 21 28 36 45 55 66 78 91 105 120 136 153 171 190 210 231 253 276 300 325 351 378 406 
435 465 496 528 561 595 630 666 703 741 780 820 861 903 946 990 1035 1081 1128 1176 1225 1275
0 1 3 6 10 15 21 28 36 45 55 65 78 91 105 120 136 153 171 190 210 231 253 276 300 325 351 377 406 
435 465 496 528 561 595 629 666 703 741 780 820 860 900 940 980 1020 1060 1100 1176 1225 1275
```
