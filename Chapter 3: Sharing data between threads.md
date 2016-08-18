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
  * C++标准库提供了实现了RAII的`std::lock_guard`类模板,它在构造时锁住互斥量,析构时释放互斥量;
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
  * 这些同名的成员函数在低层做着实际的工作，并且仅更新`std::unique_lock`实例中的标志，来确定该实例是否拥有特定的互斥量;这个标志是为了确保unlock()在析构函数中被正确调用;如果实例拥有互斥量，那么析构函数必须调用unlock();但当实例中没有互斥量时,析构函数就不能去调用unlock();这个标志可以通过owns_lock()成员函数进行查询;

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

* Because `std::unique_lock` instances don’t have to own their associated mutexes, the ownership of a mutex can be transferred between instances by moving the instances around;
  * 由于`std::unique_lock`实例不需要持有它们相关的互斥量,所以可以通过移动实例来传递互斥量所有权;
* Fundamentally this depends on whether the source is an lvalue—a real variable or reference to one—or an rvalue—a temporary of some kind. Ownership transfer is automatic if the source is an rvalue and must be done explicitly for an lvalue in order to avoid accidentally transferring ownership away from a variable;
  * 如果源值是一个右值,那么所有权传递是自动的;
  * 如果源值是一个左值,那么必须明确调用`std::move()`传递所有权,为了避免所有权转移出错;
* `std::unique_lock` is an example of a type that’s movable but not copyable;
  * `std::unique_lock`是可移动但不可复制类型;

  ```C++
  /// unique_lock
  template<typename _Mutex>
  class unique_lock
  {
  public:
    typedef _Mutex mutex_type;

    unique_lock() noexcept
    : _M_device(0), _M_owns(false)
    { }

    explicit unique_lock(mutex_type& __m)
    : _M_device(std::__addressof(__m)), _M_owns(false)
    {
      lock();
      _M_owns = true;
    }

    unique_lock(mutex_type& __m, defer_lock_t) noexcept
    : _M_device(std::__addressof(__m)), _M_owns(false)
    { }

    unique_lock(mutex_type& __m, try_to_lock_t)
    : _M_device(std::__addressof(__m)), _M_owns(_M_device->try_lock())
    { }

    unique_lock(mutex_type& __m, adopt_lock_t)
    : _M_device(std::__addressof(__m)), _M_owns(true)
    {
        // XXX calling thread owns mutex
    }

    template<typename _Clock, typename _Duration>
  unique_lock(mutex_type& __m,
      const chrono::time_point<_Clock, _Duration>& __atime)
  : _M_device(std::__addressof(__m)),
  _M_owns(_M_device->try_lock_until(__atime))
  { }

    template<typename _Rep, typename _Period>
  unique_lock(mutex_type& __m,
      const chrono::duration<_Rep, _Period>& __rtime)
  : _M_device(std::__addressof(__m)),
  _M_owns(_M_device->try_lock_for(__rtime))
  { }

    ~unique_lock()
    {
      if (_M_owns)
      unlock();
    }

    unique_lock(const unique_lock&) = delete;
    unique_lock& operator=(const unique_lock&) = delete;

    unique_lock(unique_lock&& __u) noexcept
    : _M_device(__u._M_device), _M_owns(__u._M_owns)
    {
      __u._M_device = 0;
      __u._M_owns = false;
    }

    unique_lock& operator=(unique_lock&& __u) noexcept
    {
      if(_M_owns)
        unlock();

      unique_lock(std::move(__u)).swap(*this);

      __u._M_device = 0;
      __u._M_owns = false;

      return *this;
    }

    void
    lock()
    {
      if (!_M_device)
        __throw_system_error(int(errc::operation_not_permitted));
      else if (_M_owns)
        __throw_system_error(int(errc::resource_deadlock_would_occur));
      else
      {
        _M_device->lock();
        _M_owns = true;
      }
    }

    bool
    try_lock()
    {
      if (!_M_device)
        __throw_system_error(int(errc::operation_not_permitted));
      else if (_M_owns)
        __throw_system_error(int(errc::resource_deadlock_would_occur));
      else
      {
        _M_owns = _M_device->try_lock();
        return _M_owns;
      }
    }

    template<typename _Clock, typename _Duration>
  bool
  try_lock_until(const chrono::time_point<_Clock, _Duration>& __atime)
  {
    if (!_M_device)
      __throw_system_error(int(errc::operation_not_permitted));
    else if (_M_owns)
      __throw_system_error(int(errc::resource_deadlock_would_occur));
    else
    {
      _M_owns = _M_device->try_lock_until(__atime);
      return _M_owns;
    }
  }

    template<typename _Rep, typename _Period>
  bool
  try_lock_for(const chrono::duration<_Rep, _Period>& __rtime)
  {
    if (!_M_device)
      __throw_system_error(int(errc::operation_not_permitted));
    else if (_M_owns)
      __throw_system_error(int(errc::resource_deadlock_would_occur));
    else
    {
      _M_owns = _M_device->try_lock_for(__rtime);
      return _M_owns;
    }
  }

    void
    unlock()
    {
      if (!_M_owns)
        __throw_system_error(int(errc::operation_not_permitted));
      else if (_M_device)
      {
        _M_device->unlock();
        _M_owns = false;
      }
    }

    void
    swap(unique_lock& __u) noexcept
    {
      std::swap(_M_device, __u._M_device);
      std::swap(_M_owns, __u._M_owns);
    }

    mutex_type*
    release() noexcept
    {
      mutex_type* __ret = _M_device;
      _M_device = 0;
      _M_owns = false;
      return __ret;
    }

    bool
    owns_lock() const noexcept
    { return _M_owns; }

    explicit operator bool() const noexcept
    { return owns_lock(); }

    mutex_type*
    mutex() const noexcept
    { return _M_device; }

  private:
    mutex_type*	_M_device;
    bool		_M_owns; // XXX use atomic_bool
  };
  ```

  ```C++
  std::unique_lock<std::mutex> get_lock()
  {
     extern std::mutex some_mutex;
     std::unique_lock<std::mutex> lk(some_mutex);
     prepare_data();
     return lk;
  }
  void process_data()
  {
     std::unique_lock<std::mutex> lk(get_lock());
     do_something();
  }
  ```

  * One such usage is where the lock isn’t returned directly but is a data member of a gateway class used to ensure correctly locked access to some protected data. In this case, all access to the data is through this gateway class: when you wish to access the data, you obtain an instance of the gateway class (by calling a function such as get_lock() in the preceding example), which acquires the lock. You can then access the data through member functions of the gateway object. When you’re finished, you destroy the gateway object, which releases the lock and allows other threads to access the protected data;
    * 当函数不直接返回锁而是一个gateway calss用来确认一些保护数据是否被正确锁住的数据成员时,所有访问这些数据的操作都要经过这个gateway class,当你想访问这些数据时,你需要获得一个该gateway class的实例(就像上面调用get_lock()函数一样)来获取锁,然后通过gateway class的成员函数来访问这些数据;当你结束时,销毁这个gateway calss实例来释放锁,继而允许其它线程访问这些数据;
  * The flexibility of `std::unique_lock` also allows instances to relinquish their locks before they’re destroyed. You can do this with the unlock() member function, just like for a mutex: std::unique_lock supports the same basic set of member functions for locking and unlocking as a mutex does, in order that it can be used with generic functions such as `std::lock`;
    * `std::unique_lock`允许在销毁实例前释放锁,通过调用成员函数`unlock()`实现;
* Locking at an appropriate granularity
  * 以一个合适的粒度上锁;

  ```C++
  void get_and_process_data()
  {
     std::unique_lock<std::mutex> my_lock(the_mutex);
     some_class data_to_process=get_next_data_chunk();
     my_lock.unlock();
     result_type result=process(data_to_process);
     my_lock.lock();
     write_result(data_to_process,result);
  }
  ```

* the lock granularity is a hand-waving term to describe the amount of data protected by a single lock;
  * 锁的粒度指的是描述一个锁保护的数据的总量的hand-waving术语;
* In general, a lock should be held for only the minimum possible time needed to perform the required operations;
  * 通常,一个锁只需要持续能够完成需要的操作的最小可能时间即可;
* if you don’t hold the required locks for the entire duration of an operation, you’re exposing yourself to race conditions;
  * 如果在所需操作的整个持续时间内,你诶有持有所需要的锁,那么你会进入竞争状态;
  * 下面程序在获取lhs_value and rhs_value时,可能数据发生了改变

```C++
class Y
{
private:
   int some_detail;
   mutable std::mutex m;
   int get_detail() const
   {
     std::lock_guard<std::mutex> lock_a(m);
     return some_detail;
   }
public:
   Y(int sd):some_detail(sd){}
   friend bool operator==(Y const& lhs, Y const& rhs)
   {
     if(&lhs==&rhs)
     return true;
     int const lhs_value=lhs.get_detail();
     int const rhs_value=rhs.get_detail();
     return lhs_value==rhs_value;
   }
};
```

* the C++ Standard provides a mechanism purely for protecting shared data during initialization;
  * C++标准提供了一种纯粹在初始化过程中保护共享数据的机制;
* infamous Double-Checked Locking pattern: the pointer is first read without acquiring the lock, and the lock is acquired only if the pointer is NULL. The pointer is then checked again once the lock has been acquired c (hence the doublechecked part) in case another thread has done the initialization between the first check and this thread acquiring the lock;
  * 声名狼藉的双重检查锁: 第一次读读指针时不上锁,只当这个指针为空时,上锁;然后再次检查锁,看其它线程是否在第一次检查和获取锁期间对该指针做了初始化;
  * 代码如下

  ```C++
  void undefined_behaviour_with_double_checked_locking()
  {
     if(!resource_ptr)
     {
       std::lock_guard<std::mutex> lk(resource_mutex);
       if(!resource_ptr)
       {
         resource_ptr.reset(new some_resource);
       }
     }
     resource_ptr->do_something();
  }
  ```

  * Unfortunately, this pattern is infamous for a reason: it has the potential for nasty race conditions, because the read outside the lock isn’t synchronized with the write done by another thread inside the lock. This therefore creates a race condition that covers not just the pointer itself but also the object pointed to; even if a thread sees the pointer written by another thread, it might not see the newly created instance of some_resource, resulting in the call to do_something() operating on incorrect values;
    * 这样的问题是: 因为第一次读的时候没有与另一个线程的写同步,这样在指针不为空的时候,指针在程序执行期间可能会发生改变,即数据竞争;即使第一次验证为空,第二次检验发现有另一个线程对指针做了初始化,但新创建的实例可能不可见,那么调用do_something()的结果就会产生不正确的结果;
* Rather than locking a mutex and explicitly checking the pointer, every thread can just use `std::call_once`, safe in the knowledge that the pointer will have been initialized by some thread (in a properly synchronized fashion) by the time `std::call_once` returns;
  * 比起锁住互斥量并显式的检查指针,每个线程可以只使用`std::call_once`,当其返回时就能知道指针已经在其它线程初始化了;
* Use of `std::call_once` will typically have a lower overhead than using a mutex explicitly, especially when the initialization has already been done, so should be used in preference where it matches the required functionality;
  * 使用`std::call_once`比显式使用互斥量更省资源,尤其是在初始化已经完成之后,这个结论同样适合于引用;

  ```C++
  std::shared_ptr<some_resource> resource_ptr;
  std::once_flag resource_flag;
  void init_resource()
  {
    resource_ptr.reset(new some_resource);
  }
  void foo()
  {
     std::call_once(resource_flag,init_resource); // Initialization is called exactly once
     resource_ptr->do_something();
  }
  ```
  ```C++
  class X
  {
  private:
     connection_info connection_details;
     connection_handle connection;
     std::once_flag connection_init_flag;
     void open_connection()
     {
       connection=connection_manager.open(connection_details);
     }
  public:
      X(connection_info const& connection_details_):
     connection_details(connection_details_)
     {}
     void send_data(data_packet const& data)
     {
       std::call_once(connection_init_flag,&X::open_connection,this);
       connection.send_data(data);
     }
     data_packet receive_data()
     {
       std::call_once(connection_init_flag,&X::open_connection,this);
       return connection.receive_data();
     }
  };
  ```

  *  It’s worth noting that, like `std::mutex`, `std::once_flag` instances can’t be copied or moved, so if you use them as a class member like this, you’ll have to explicitly define these special member functions should you require them;
    * `std::mutex`和`std::once_flag`实例不能复制或者移动,如果把它们当作类成员来使用,你需要明确定义成员函数你要使用它们;
* This can be used as an alternative to `std::call_once` for those cases where a single global instance is required:
  * 当只要一个全局实例时,`std::call_once`的替代方案为:

  ```C++
  class my_class;
  my_class& get_my_class_instance()
  {
    static my_class instance;
    return instance;
  }
  ```

  * Multiple threads can then call `get_my_class_instance()` safely, without having to worry about race conditions on the initialization;
    * 多线程可以安全的调用`get_my_class_instance()`函数,而不用担心初始化过程中的条件竞争;
* Protecting data only for initialization is a special case of a more general scenario: that of a rarely updated data structure. For most of the time, such a data structure is read-only and can therefore be merrily read by multiple threads concurrently, but on occasion the data structure may need updating. What’s needed here is a protection mechanism that acknowledges this fact;
  * 初始化数据保护是下列情况的一个特例:
    * 对于很少有更新的数据结构,大多数情况下是只读的,所以并发不会有问题;但是一旦数据发生更新,就可能造成条件竞争;
* 保护很少进行更新的数据结构
  * 使用`boost::shared_mutex`代替使用`std::mutex`;
  * 当数据进行更新操作时,可以使用`std::lock_guard<boost::shared_mutex>`和`std::unique_lock<boost::shared_mutex>`进行锁定,这能保证单独访问;
  * 不更新数据结构时,可以使用`boost::shared_lock<boost::shared_mutex>`去访问共享数据;
  * The only constraint is that if any thread has a shared lock, a thread that tries to acquire an exclusive lock will block until all other threads have relinquished their locks, and likewise if any thread has an exclusive lock, no other thread may acquire a shared or exclusive lock until the first thread has relinquished its lock;
    * 这样做的唯一限制是
      * 当一个线程尝试获取独占锁时,它需要等待其它拥有共享锁的线程解锁;
      * 当一个线程拥有独占锁时,其它线程不能获取独占锁或共享锁;

  ```C++
  #include <map>
  #include <string>
  #include <mutex>
  #include <boost/thread/shared_mutex.hpp>
  class dns_entry;
  class dns_cache
  {
     std::map<std::string,dns_entry> entries;
     mutable boost::shared_mutex entry_mutex;
  public:
     dns_entry find_entry(std::string const& domain) const
     {
       boost::shared_lock<boost::shared_mutex> lk(entry_mutex);
       std::map<std::string,dns_entry>::const_iterator const it=
       entries.find(domain);
       return (it==entries.end())?dns_entry():it->second;
     }
     void update_or_add_entry(std::string const& domain,
     dns_entry const& dns_details)
     {
       std::lock_guard<boost::shared_mutex> lk(entry_mutex);
       entries[domain]=dns_details;
     }
  };
  ```

* With std::mutex, it’s an error for a thread to try to lock a mutex it already owns, and attempting to do so will result in undefined behavior;
  * 当线程尝试对一个已经锁定的互斥量上锁时,可能发生错误,多次尝试会发生未知行为;
* 使用`std::recursive_mutex`可以让一个互斥量在同一线程被多次上锁,但也需要相应次数的解锁;
  * Most of the time, if you think you want a recursive mutex, you probably need to change your design instead;
  * However, such usage is not recommended,because it can lead to sloppy thinking and bad design;
    * 递归锁定是不推荐的;
