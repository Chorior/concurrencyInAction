# Chapter 3: Sharing Data between Threads
---
:joy:
---
* 多线程的一个关键优点(key benefit)是可以简单的直接共享数据,但如果有多个线程拥有修改共享数据的权限,那么就会很容易出现数据竞争(data race);
  * The C++ Standard also defines the term data race to mean the specific type of race condition that arises because of concurrent modification to a single object;
* The most basic mechanism for protecting shared data provided by the C++ Standard is the mutex;
  * C++标准保护共享数据最基本的技巧是使用互斥量(mutex);
    * Before accessing a shared data structure, you lock the mutex associated with that data, and when you’ve finished accessing the data structure, you unlock the mutex;
      * 访问共享数据前锁住互斥量,完成时释放互斥量;
    * The Thread Library then ensures that once one thread has locked a specific mutex, all other threads that try to lock the same mutex have to wait until the thread that successfully locked the mutex unlocks it;
      * 线程库确保,一旦一个线程已经锁住特定互斥量,那么其他想要锁住这个互斥量的线程需要等待那个成功锁住这个互斥量的线程释放才行;
* 在C++中使用互斥量
  * 创建互斥量: 建造一个`std::mutex`的实例;
  * 锁住互斥量: 调用成员函数`lock()`;
  * 释放互斥量: 调用成员函数`unlock()`;
* However, it isn’t recommended practice to call the member functions directly, because this means that you have to remember to call unlock() on every code path out of a function, including those due to exceptions;
  * 不建议直接调用成员函数,因为那样你必须记得在每个函数出口(包括异常)处调用`unlock()`;
* Instead, the Standard C++ Library provides the std::lock_guard class template, which implements that RAII idiom for a mutex; it locks the supplied mutex on construction and unlocks it on destruction, thus ensuring a locked mutex is always correctly unlocked;
  * C++标准库提供了实现了RAII的`std::lock_guard`类模板,它爱构造时锁住互斥量,析构时释放互斥量;
  * RAII(Resource Acquisition Is Initialization): 资源获取就是初始化,是C++语言的一种管理资源、避免泄漏的惯用法,C++标准保证任何情况下,已构造的对象最终会销毁,即它的析构函数最终会被调用;
* `std::mutex`和`std::lock_guard`都在头文件`<mutex>`中定义;

  ```C++
  #include <list>
  #include <mutex>
  #include <algorithm>

  std::list<int> some_list;
  std::mutex some_mutex;

  void add_to_list(int new_value)
  {
     std::lock_guard<std::mutex> guard(some_mutex);
     some_list.push_back(new_value);
  }

  bool list_contains(int value_to_find)
  {
     std::lock_guard<std::mutex> guard(some_mutex);
     return std::find(some_list.begin(),some_list.end(),value_to_find)
               != some_list.end();
  }
  ```

* if one of the member functions returns a pointer or reference to the protected data, then it doesn’t matter that the member functions all lock the mutex in a nice orderly fashion, because you’ve just blown a big hole in the protection;
  * Any code that has access to that pointer or reference can now access (and potentially modify) the protected data without locking the mutex;
  * 如果函数返回指针或引用,那么会破坏对共享数据的保护;
  * 下面代码在没有保护的情况下,调用了x.data->do_something();

  ```C++
  class some_data
  {
     int a;
     std::string b;
  public:
    void do_something();
  };
  class data_wrapper
  {
  private:
     some_data data;
     std::mutex m;
  public:
     template<typename Function>
     void process_data(Function func)
     {
       std::lock_guard<std::mutex> l(m);
       func(data);
     }
  };
  some_data* unprotected;
  void malicious_function(some_data& protected_data)
  {
    unprotected=&protected_data;
  }
  data_wrapper x;
  void foo()
  {
     x.process_data(malicious_function);
     unprotected->do_something();
  }
  ```
  * Don’t pass pointers and references to protected data outside the scope of the lock, whether by returning them from a function, storing them in externally visible memory, or passing them as arguments to user-supplied functions;
    * 这个示例告诉我们,不要将保护数据的指针或引用传递到互斥锁作用域外面的地方去;
* it’s still possible to have race conditions, even when data is protected with a mutex;
  * 即使使用了互斥锁,也有可能出现竞争;
* 互斥量保护的粒度不能太小,那样保护不完全;也不能太大,那样会降低性能;
* If you end up having to lock two or more mutexes for a given operation, there’s another potential problem lurking in the wings: deadlock;
  * 如果一个操作要锁住两个或者更多互斥量,那么可能会造成死锁(deadlock);
  * deadlock: rather than two threads racing to be first, each one is waiting for the other, so neither makes any progress;
* The common advice for avoiding deadlock is to always lock the two mutexes in the same order;
  * 解决死锁的经典办法是按顺序锁住互斥量;
  * 这个方法的一个问题是
    * 如果一个`swap()`函数交换两个类的实例,这时有两个线程调用这个`swap()`函数并且镜像执行相同的操作,那么也可能造成死锁(比如说A和B交换数据,A先锁定了自己,这个时候B在另一个线程也锁定了自己,就会造成死锁);
      * Thankfully, the C++ Standard Library has a cure for this in the form of std::lock — a function that can lock two or more mutexes at once without risk of deadlock.
        * 幸运的是,C++标准库提供了`std::lock`函数,可以一次锁住两个或多个互斥量,并不会有死锁风险;

        ```C++
        class some_big_object;
        void swap(some_big_object& lhs,some_big_object& rhs);
        class X
        {
        private:
           some_big_object some_detail;
           std::mutex m;
        public:
           X(some_big_object const& sd):some_detail(sd){}
           friend void swap(X& lhs, X& rhs)
           {
             if(&lhs==&rhs)
             return;
             std::lock(lhs.m,rhs.m);
             std::lock_guard<std::mutex> lock_a(lhs.m,std::adopt_lock);
             std::lock_guard<std::mutex> lock_b(rhs.m,std::adopt_lock);
             swap(lhs.some_detail,rhs.some_detail);
           }
        };
        ```

        * The std::adopt_lock parameter is supplied in addition to the mutex to indicate to the std::lock_guard objects that the mutexes are already locked, and they should just adopt the ownership of the existing lock on the mutex rather than attempt to lock the mutex in the constructor；
          * 参数`std::adopt_lock`告诉`std::lock_guard`对象,互斥量已经被锁住了,只需接收已存在的互斥量的锁定的所有权即可,不要试图构造时锁住它们;
        * std::lock provides all-or-nothing semantics with regard to locking the supplied mutexes;
          * `std::lock`要么锁住全部提供的互斥量,要么一个也没锁住;
* The guidelines for avoiding deadlock all boil down to one idea:don’t wait for another thread if there’s a chance it’s waiting for you;
  * 避免死锁的指南归结为一个: 当有机会时,不要去等待另一个线程;
* 几种建议识别和消除另外一个线程等待的可能
  * don’t acquire a lock if you already hold one;If you need to acquire multiple locks, do it as a single action with std::lock in order to acquire them without deadlock;
    * 当持有一个锁时,不要试图去获取另一个锁;
    * 要使用多个锁时,使用`std::lock`即可;
  * avoid calling user-supplied code while holding a lock;
    * 持有锁时,避免调用用户自定义的代码,因为也许会违背第一条建议;
  * If you absolutely must acquire two or more locks, and you can’t acquire them as a single operation with std::lock, the next-best thing is to acquire them in the same order in every thread;
    * 如果必须要多个锁嵌套时,按顺序获取它们;
  * use a lock hierarchy
    * 使用层级锁结构
      * When code tries to lock a mutex, it isn’t permitted to lock that mutex if it already holds a lock from a lower layer;
        * 当代码已经持有一个底层锁时,不允许对较高的互斥量上锁;
      * Deadlocks between hierarchical mutexes are thus impossible, because the mutexes themselves enforce the lock ordering;
        * 由于层级结构的锁自身会遵守一定的顺序,所以层级结构的互斥量是不可能产生死锁的;
      * This does mean that you can’t hold two locks at the same time if they’re the same level in the hierarchy
        * 同一层不能持有两个或两个以上的锁;
      * `hierarchical_mutex` is not part of the standard but is easy to write;it can be used with `std::lock_guard<>` because it implements the three member functions required to satisfy the mutex concept: `lock()`, `unlock()`, and `try_lock()`;
        * `hierarchical_mutex`不是标准库的一部分,它可以和`std::lock_guard<>`一起使用,因为它实现了三个满足mutex需要的成员函数: `lock()`, `unlock()`, and `try_lock()`;

        ```C++
        #include <iostream>
        #include <thread>
        #include <cstdlib> // throw
        #include <mutex>
        #include <climits> // ULONG_MAX

        class hierarchical_mutex
        {
           std::mutex internal_mutex;
           unsigned long const hierarchy_value;
           unsigned long previous_hierarchy_value;
           static thread_local unsigned long this_thread_hierarchy_value;

           void check_for_hierarchy_violation()
           {
        	    if(this_thread_hierarchy_value <= hierarchy_value)
        	    {
        	    	throw std::logic_error("mutex hierarchy violated");
        	    }
           }

           void update_hierarchy_value()
           {
        	    previous_hierarchy_value=this_thread_hierarchy_value;
        	    this_thread_hierarchy_value=hierarchy_value;
           }

        public:
           explicit hierarchical_mutex(unsigned long value):
           hierarchy_value(value),
           previous_hierarchy_value(0)
           {
           		std::cout << "hierarchical_mutex() called!\n";
           }
           void lock()
           {
        	   	std::cout << "lock() called!\n";
        	    check_for_hierarchy_violation();
        	    internal_mutex.lock();
        	    update_hierarchy_value();
           }
           void unlock()
           {
           		std::cout << "unlock() called!\n";
        	    this_thread_hierarchy_value=previous_hierarchy_value;
        	    internal_mutex.unlock();
           }
           bool try_lock()
           {
           		std::cout << "try_lock() called!\n";
        	    check_for_hierarchy_violation();
        	    if(!internal_mutex.try_lock())
        	    return false;
        	    update_hierarchy_value();
        	    return true;
           }
        };
        thread_local unsigned long
         hierarchical_mutex::this_thread_hierarchy_value(ULONG_MAX);
        ```

        * 简单示例

        ```C++
        #include "hierarchical_mutex.h"

        hierarchical_mutex high_level_mutex(10000);
        hierarchical_mutex low_level_mutex(5000);

        int do_low_level_stuff()
        {
        	std::cout << "do_low_level_stuff() called!\n";
        	return 0;
        }

        int low_level_func()
        {
        	std::lock_guard<hierarchical_mutex> lk(low_level_mutex);
        	std::cout << "low_level_func() called!\n";
        	return do_low_level_stuff();
        }

        void high_level_stuff(int some_param)
        {
        	std::cout << "high_level_stuff() called!\n";
        }

        void high_level_func()
        {
        	std::lock_guard<hierarchical_mutex> lk(high_level_mutex);
        	std::cout << "high_level_func() called!\n";
        	high_level_stuff(low_level_func());
        }

        void thread_a()
        {
        	high_level_func();
        }

        hierarchical_mutex other_mutex(100);

        void do_other_stuff()
        {
        	std::cout << "do_other_stuff() called!\n";
        }

        void other_stuff()
        {
        	std::cout << "other_stuff() called!\n";
        	high_level_func();
         	do_other_stuff();
        }

        void thread_b()
        {
        	std::lock_guard<hierarchical_mutex> lk(other_mutex);
        	other_stuff();
        }

        int main()
        {
        	thread_a();

        	return EXIT_SUCCESS;
        }
        ```

        * 执行结果

        ```
        hierarchical_mutex() called!
        hierarchical_mutex() called!
        hierarchical_mutex() called!
        lock() called!
        high_level_func() called!
        lock() called!
        low_level_func() called!
        do_low_level_stuff() called!
        unlock() called!
        high_level_stuff() called!
        unlock() called!
        ```

* `std::unique_lock` provides a bit more flexibility than `std::lock_guard` by relaxing the invariants; a `std::unique_lock` instance doesn’t always own the mutex that it’s associated with;
  * `std::unique_lock`比`std::lock_guard`更加灵活,一个`std::unique_lock`实例不一定总拥有关联它的互斥量?
* First off, just as you can pass `std::adopt_lock` as a second argument to the constructor to have the lock object manage the lock on a mutex, you can also pass `std::defer_lock` as the second argument to indicate that the mutex should remain unlocked on construction;
  * 传递`std::adopt_lock`作为第二个参数来管理互斥锁;传递`std::defer_lock`作为第二个参数来指明构造时互斥量应该保持释放状态;
* The lock can then be acquired later by calling lock() on the `std::unique_lock` object (not the mutex) or by passing the `std::unique_lock` object itself to `std::lock()`;
  * 然后你可以使用`std::unique_lock`实例来调用`lock()`,或者传递`std::unique_lock`实例给`std::lock()`来锁定互斥量;
* 下面程序使用`std::unique_lock` and `std::defer_lock`,而不是`std::lock_guard` and `std::adopt_lock`,它们的功能是一样的,但是
  * `std::unique_lock` takes more space and is a fraction slower to use than `std::lock_guard`;
  * The flexibility of allowing a `std::unique_lock` instance not to own the mutex comes at a price: this information has to be stored, and it has to be updated;
    * 允许`std::unique_lock`实例不持有互斥量的代价是: 信息已被存储，且已被更新;
* the `std::unique_lock` objects could be passed to `std::lock()` because `std::unique_lock` provides lock(), try_lock(), and unlock() member functions;
  * `std::unique_lock`实例能被传递给`std::lock()`是因为`std::unique_lock`提供lock(), try_lock(), and unlock()成员函数;
* These forward to the member functions of the same name on the underlying mutex to do the actual work and just update a flag inside the `std::unique_lock` instance to indicate whether the mutex is currently owned by that instance. This flag is necessary in order to ensure that unlock() is called correctly in the destructor. If the instance does own the mutex, the destructor must call unlock(), and if the instance does not own the mutex, it must not call unlock(). This flag can be queried by calling the owns_lock() member function;
  * 这些同名的成员函数在低层做着实际的工作，并且仅更新`std::unique_lock`实例中的标志，来确定该实例是否拥有特定的互斥量;这个标志是为了确保unlock()在析构函数中被正确调用;如果实例拥有互斥量，那么析构函数必须调用unlock();但当实例中没有互斥量时,析构函数就不能去调用unlock();这个标志可以通过owns_lock()成员变量进行查询;

```C++
class some_big_object;
void swap(some_big_object& lhs,some_big_object& rhs);
class X
{
private:
   some_big_object some_detail;
   std::mutex m;
public:
   X(some_big_object const& sd):some_detail(sd){}
   friend void swap(X& lhs, X& rhs)
   {
     if(&lhs==&rhs)
     return;
     std::unique_lock<std::mutex> lock_a(lhs.m,std::defer_lock);
     std::unique_lock<std::mutex> lock_b(rhs.m,std::defer_lock);
     std::lock(lock_a,lock_b);
     swap(lhs.some_detail,rhs.some_detail);
   }
};
```
* 3.2.7
