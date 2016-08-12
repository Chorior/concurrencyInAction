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
* 3.2.5
