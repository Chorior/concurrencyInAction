# Chapter 4: Synchronizing concurrent operations
---
:art:
---
* 等待一个事件完成
  * 持续检查完成标志,直至完成;
    * 消耗不必要的时间检查;
    * 上锁时,其它需要该锁的线程会处于等待状态,消耗系统资源;
  * 在等待完成期间,使用`std::this_thread::sleep_for()`函数进行周期时间间歇;
    * 难以确定合适的休息时间;

      ```C++
      bool flag;
      std::mutex m;
      void wait_for_flag()
      {
         std::unique_lock<std::mutex> lk(m);
         while(!flag)
         {
           lk.unlock();
           std::this_thread::sleep_for(std::chrono::milliseconds(100));
           lk.lock();
         }
      }
      ```

  * 使用Ｃ++标准库工具等待时间本身(最好的选择)
    * The most basic mechanism for waiting for an event to be triggered by another thread is the condition variable;
      * 等待事件被另一个线程触发最基本的措施是使用条件变量;
    * Conceptually, a condition variable is associated with some event or other condition, and one or more threads can wait for that condition to be satisfied. When some thread has determined that the condition is satisfied, it can then notify one or more of the threads waiting on the condition variable, in order to wake them up and allow them to continue processing;
      * 从概念上讲,一个条件变量与多个事件或其它条件相关,并且一个或更多线程会等待那个条件的达成;当某些线程确定那个条件达成时,它会通知一个或多个在条件变量上等待的线程,继而唤醒它们,继续工作;
* C++标准库提供两个条件变量的实现
  * 头文件: `<condition_variable>`;
  * `std::condition_variable`;
    * 只能与`std::mutex`一起使用;
  * `std::condition_variable_any`;
    * can work with anything that meets some minimal criteria for being mutex-like, hence the `_any` suffix;
    * 比`std::condition_variable`更费资源
  * In both cases, they need to work with a mutex in order to provide appropriate synchronization;
    * 它们都需要一个互斥量进行同步;
  * `std::condition_variable`更受欢迎,除非灵活性需要;
  * 示例(有问题么?)

    ```C++
    #include <iostream>
    #include <mutex>
    #include <thread>
    #include <condition_variable>
    #include <queue>
    #include <string>

    std::mutex mut;
    std::queue<std::string> data_queue;
    std::condition_variable data_cond;

    void data_preparation(std::string data)
    {
      std::cout << "data_preparation() called!\n";
      std::lock_guard<std::mutex> lk(mut);
      data_queue.push(data);
      data_cond.notify_one();
    }

    void data_processing()
    {
      std::cout << "data_processing() called!\n";
      while(true)
      {
        std::unique_lock<std::mutex> lk(mut);
        data_cond.wait(
        lk,[]{return !data_queue.empty();});
        std::string data = data_queue.front();
        data_queue.pop();
        lk.unlock();

        std::cout << "data = " << data << std::endl;

        if("quit" == data)
        {
          break;
        }
      }
    }

    void thread_a(std::string data)
    {
      std::thread t(data_preparation,data);
      t.join();
    }

    void thread_b()
    {
      std::thread t(data_processing);
      t.join();
    }

    int main()
    {
      while(true)
      {
        std::cout << "please input a string(quit to end): ";
        std::string data;
        getline(std::cin,data);
        thread_a(data);

        if("quit" == data)
        {
          break;
        }
      }
      thread_b();

      return EXIT_SUCCESS;
    }
    ```

  * 数据处理线程使用`std::unique_lock`是为了方便解锁和上锁;
  * `std::condition_variable`成员函数`wait()`
    * 接受一个锁类实例和一个bool函数;
    * 如果bool函数返回true,则`wait()`立即返回;
    * 如果bool函数返回false,则`wait()`会解锁传递的互斥量,并使线程阻塞或等待;
    * 当`std::condition_variable`成员函数`notify_one()`成员函数被调用时,线程醒来,重新获取锁,并再次检查检查bool函数,如未返回true,继续解锁等待;
    * 若等待线程不是响应另一个线程的通知而重新获取锁并检查bool函数时,称为伪唤醒
      * 由于伪唤醒的数量和频率不确定,所以不建议使用有副作用的函数做条件检查;
    * `notify_all()`通知所有等待线程检查条件;
    * calls the notify_one() member function on the std::condition_variable instance to notify the waiting thread (if there is one);
    * The implementation of wait() then checks the condition (by calling the supplied lambda function) and returns if it’s satisfied (the lambda function returned true ). If the condition isn’t satisfied (the lambda function returned false ), wait() unlocks the mutex and puts the thread in a blocked or waiting state. When the condition variable is notified by a call to notify_one() from the data-preparation thread, the thread wakes from its slumber (unblocks it), reacquires the lock on the mutex, and checks the condition again, returning from wait() with the mutex still locked if the condition has been satisfied. If the condition hasn’t been satisfied, the thread unlocks the mutex and resumes waiting;
    * During a call to wait() , a condition variable may check the supplied condition any number of times; however, it always does so with the mutex locked and will return immediately if (and only if) the function provided to test the condition returns true . When the waiting thread reacquires the mutex and checks the condition, if it isn’t in direct response to a notification from another thread, it’s called a spurious wake. Because the number and frequency of any such spurious wakes are by definition indeterminate, it isn’t advisable to use a function with side effects for the condition check. If you do so, you must be prepared for the side effects to occur multiple times;
* 使用期望等待一次性事件
  * If a thread needs to wait for a specific one-off event, it somehow obtains a future representing this event;
    * 如果一个线程需要等待一个特定的一次性事件,那么它需要通过一些方法获取这个事件未来的表现形式;
  * 这个线程会周期性的等待或检查事件是否触发,在检查间隙可以做其它的任务直到时间发生,然后等待期望状态变为"ready";
  * 期望不能被重置;
  * C++标准库实现了两种future模板
    * 头文件`<future>`;
    * unique futures(`std::future<>`);
    * shared futures(`std::shared_future<>`);
    * 分别仿照`std::unique_ptr`和`std::shared_ptr`;
    * 一个`std::future<>`实例是它关联事件的唯一实例;
    * 多个`std::shared_future<>`实例可以同时关联到同一個事件;这种情况下,所有实例会在同一时间变为ready状态,它们可能全部访问事件关联的数据;
    * 模板参数是关联数据的类型,void表示事件无关联数据;
    * 期望类不提供同步访问,如果多个线程需要访问同一个期望实例,那么它们需要通过互斥量或其它第三章的同步措施来保护访问;
* 从后台任务中返回值
  * 使用`std::async`启动一个不需要立即获取结果的异步任务;
    * `std::async`返回一个`std::future`对象,这个对象最后持有函数的返回值;
    * 对返回的future对象调用`get()`,线程阻塞直到future状态变为ready,然后返回结果;

      ```C++
      #include <future>
      #include <iostream>
      int find_the_answer_to_ltuae();
      void do_other_stuff();
      int main()
      {
         std::future<int> the_answer=std::async(find_the_answer_to_ltuae);
         do_other_stuff();
         std::cout<<"The answer is "<<the_answer.get()<<std::endl;
      }
      ```

    * `std::async`第一个参数为函数指针或可调用对象,若是成员函数,则最后一个参数为相应的类实例(指针,引用或拷贝),而后为函数参数列表;

      ```C++
      #include <string>
      #include <future>
      struct X
      {
         void foo(int,std::string const&);
         std::string bar(std::string const&);
      };
      X x;
      auto f1=std::async(&X::foo,&x,42,"hello"); // Calls p->foo(42,"hello") where p is &x
      auto f2=std::async(&X::bar,x,"goodbye"); // Calls tmpx.bar("goodbye") where tmpx is a copy of x
      struct Y
      {
        double operator()(double);
      };
      Y y;
      auto f3=std::async(Y(),3.141); // Calls tmpy(3.141) where tmpy is move-constructed from Y()
      auto f4=std::async(std::ref(y),2.718); // Calls y(2.718)
      X baz(X&);
      std::async(baz,std::ref(x)); // Calls baz(x)
      class move_only
      {
      public:
         move_only();
         move_only(move_only&&)
         move_only(move_only const&) = delete;
         move_only& operator=(move_only&&);
         move_only& operator=(move_only const&) = delete;
         void operator()();
      };
      auto f5=std::async(move_only()); // Calls tmp() where tmp is constructed from std::move(move_only())
      ```

    * 在‘std::sync’参数函数或可调用对象前添加额外参数
      * `std::launch::deferred`指明函数延迟直到`wait()`或`get()`被future调用;
        *  If the function call is deferred, it may never actually run;
      * `std::launch::async`指明函数必须在它自己的线程运行;
      * `std::launch::deferred | std::launch::async`指明由编译器选择;
      * 不添加时,默认为`std::launch::deferred | std::launch::async`;

        ```C++
        auto f6=std::async(std::launch::async,Y(),1.2); // Run in new thread
        auto f7=std::async(std::launch::deferred,baz,std::ref(x)); // Run in wait() or get()
        auto f8=std::async(
         std::launch::deferred | std::launch::async,
         baz,std::ref(x)); // Implementation chooses
        auto f9=std::async(baz,std::ref(x)); // // Implementation chooses
        f7.wait();  // Invoke deferred function
        ```

* 使用future关联任务
  * `std::packaged_task<>` ties a future to a function or callable object;
    * `std::packaged_task<>`绑定一个future到一个函数或一个可调用对象;
  * 当`std::packaged_task<>`对象被调用,它会调用关联的函数或可调用对象,并且使future状态变为ready,返回值也会被存储为关联数据;
  * 如果一个大的任务可以被划分为独立的小任务,那么每个小任务都可以被包含在一个`std::packaged_task<>`实例中;
  * `std::packaged_task<>`的模板参数是一个函数签名,void表示无返回值无参数函数, `int(std::string&,double*)`代表返回值为int,参数类型为`string &`,`double *`;
  * 构造`std::packaged_task<>`实例时,必须传入一个函数或可调用对象,这个函数或可调用对象需要能接收指定的参数和返回可转换为指定返回类型的值,可以不完全匹配;
    * 如`std::packaged_task<double(double)>`实例可以用float(int)函数构建;
  * 函数签名返回类型指定`std::future<>`调用`get_future()`成员函数的返回类型;
  * `std::packaged_task<>`对象是一个可调用对象
    * 它可以包含在`std::function`对象中,然后传递给`std::thread`作为线程函数;
    * 也可以传递给一个需要可调用对象的函数;
    * 或者直接调用;
  * 当`std::packaged_task<>`对象被当作一个函数对象调用时,返回值被存储在`std::future`中,通过`get_future()`获取;
  * `std::packaged_task<>`的部分实现

      ```C++
      template<>
      class packaged_task<std::string(std::vector<char>*,int)>
      {
      public:
         template<typename Callable>
         explicit packaged_task(Callable&& f);
         std::future<std::string> get_future();
         void operator()(std::vector<char>*,int);
      };
      ```

* 线程间传递任务

  ```C++
  #include <deque>
  #include <mutex>
  #include <future>
  #include <thread>
  #include <utility>

  std::mutex m;
  std::deque<std::packaged_task<void()> > tasks;
  bool gui_shutdown_message_received();
  void get_and_process_gui_message();

  void gui_thread()
  {
    while(!gui_shutdown_message_received())
    {
      get_and_process_gui_message();
      std::packaged_task<void()> task;
      {
      std::lock_guard<std::mutex> lk(m);
      if(tasks.empty())
        continue;
      task=std::move(tasks.front());
      tasks.pop_front();
      }
      task();
    }
  }
  std::thread gui_bg_thread(gui_thread);
  template<typename Func>
  std::future<void> post_task_for_gui_thread(Func f)
  {
    std::packaged_task<void()> task(f);
    std::future<void> res=task.get_future();
    std::lock_guard<std::mutex> lk(m);
    tasks.push_back(std::move(task));
    return res;
  }
  ```

* 使用 `std::promise`
  * `std::promise<T>` provides a means of setting a value (of type T ), which can later be read through an associated `std::future<T>` object. A `std::promise / std::future` pair would provide one possible mechanism for this facility; the waiting thread could block on the future, while the thread providing the data could use the promise half of the pairing to set the associated value and make the future ready;
    * `std::promise<T>`提供一种设置值的方法,这个值可以在设置之后被关联的`std::future<T>`对象读取;
    * 一个`std::promise / std::future`对可以实现上面的机制;
    * 等待的线程会在future上阻塞,提供数据的线程会根据promise去设置关联的值,并使future ready;
  * 可以通过调用`get_future()`成员函数获取一个给定的`std::promise`关联的`std::future`对象;
  * 在使用`set_value()`成员函数设定promise的值后,关联的future变为ready,且可以被用来获取存储的值;
  * 如果在设置promise值之前销毁`std::promise`,future会存储异常;

    ```C++
    #include <future>
    void process_connections(connection_set& connections)
    {
      while(!done(connections))
      {
        for(connection_iterator
        connection=connections.begin(),end=connections.end();
        connection!=end;
        ++connection)
        {
          if(connection->has_incoming_data())
          {
            data_packet data=connection->incoming();
            std::promise<payload_type>& p=
            connection->get_promise(data.id);
            p.set_value(data.payload);
          }
          if(connection->has_outgoing_data())
          {
            outgoing_packet data=
            connection->top_of_outgoing_queue();
            connection->send(data.payload);
            data.promise.set_value(true);
          }
        }
      }
    }
    ```

* 为 future 存储异常
  * 当`std::async`或`std::packaged_task`或`std::promise`调用的函数发生异常时,异常会存储在`std::future`中,通过相应获取future的方法来获取异常;
  * 在`std::promise`中,如果你想存储一个异常而不是值,可以调用`set_exception()`代替`set_value()`;

    ```C++
    extern std::promise<double> some_promise;
    try
    {
      some_promise.set_value(calculate_value());
    }
    catch(...)
    {
      some_promise.set_exception(std::current_exception());
    }
    ```

  * 上面使用`std::current_exception()`获取抛出的异常;
  * 可以使用`std::copy_exception()`直接存储一个新的异常而不用抛出

    ```C++
    some_promise.set_exception(std::copy_exception(std::logic_error("foo ")));
    ```

  * 在future中存储异常的另一个方式是
    * Another way to store an exception in a future is to destroy the `std::promise` or `std::packaged_task` associated with the future without calling either of the set functions on the promise or invoking the packaged task. In either case, the destructor of the `std::promise` or `std::packaged_task` will store a `std::future_error` exception with an error code of `std::future_errc::broken_promise` in the associated state if the future isn’t already ready;
    * 在调用set函数或包装人物前就相会了future关联的`std::promise`或`std::packaged_task`,如果这时future没有ready，那么析构函数就会存储一个与`std::future_errc::broken_promise`错误状态相关的`std::future_error`异常;
  * `std::future`的局限性
    * 只有一个线程能够等待future结果;
    * 当有很多线程等待同一事件时,需要使用`std::shared_future`;
* 多个线程的等待
  * `std::future` is only moveable;
  * `std::shared_future` instances are copyable;
  * Now, with `std::shared_future` , member functions on an individual object are still unsynchronized, so to avoid data races when accessing a single object from multiple threads, you must protect accesses with a lock. The preferred way to use it would be to take a copy of the object instead and have each thread access its own copy. Accesses to the shared asynchronous state from multiple threads are safe if each thread accesses that state through its own `std::shared_future` object;

    ```C++
    std::promise<int> p;
    std::future<int> f(p.get_future());
    assert(f.valid()); // The future f is valid
    std::shared_future<int> sf(std::move(f));
    assert(!f.valid()); // f is no longer valid
    assert(sf.valid()); // sf is now valid
    ```

    ![shared_future](http://o7s72jtji.bkt.clouddn.com/std::shared_future)

  * you can construct a std::shared_future directly from the return value of the `get_future()` member function of a `std::promise` object

    ```C++
    std::promise<std::string> p;
    std::shared_future<std::string> sf(p.get_future());
    // Implicit transfer of ownership
    ```

  * `std::future` has a share() member function that creates a new `std::shared_future` and transfers ownership to it directly

    ```C++
    std::promise< std::map< SomeIndexType, SomeDataType,
      SomeComparator,SomeAllocator>::iterator> p;
    auto sf=p.get_future().share();
    ```

* 超时等待
  * 基于持续时间(duration-based)的超时(如20s)有`_for`后缀;
  * 基于绝对时间(absolute)的超时(如2019.3.31)有`_until`后缀;
  * 稳定的时钟对于超时运算非常重要;  
  * 头文件`<chrono>`
    * `std::chrono::system_clock`不是稳定的;
    * `std::chrono::steady_clock.`是稳定的;
  * std::chrono::system_clock::now() will return the current time of the system clock;
  * The is_steady static data member of the clock class is true if the clock is steady and false otherwise;
* 持续时间(durations)用`std::chrono::duration<>`模板类进行处理(我发现使用英文记录比翻译要快多了,为什么要翻译呢,对吧)
  * The first template parameter is the type of the representation (such as int, long, or double);
  * the second is a fraction specifying how many seconds each unit of the duration represents;
  * example
    * a number of minutes stored in a short is `std::chrono::duration<short,std::ratio<60,1>>`, because there are 60 seconds in a minute;
    * a count of milliseconds stored in a double is `std::chrono::duration<double,std::ratio<1,1000>>`, because each millisecond is 1/1000 of a second;
  * The Standard Library provides a set of predefined typedefs in the `std::chrono` namespace for various durations
    * nanoseconds;
    * microseconds;
    * milliseconds;
    * seconds;
    * minutes;
    * hours
  * Conversion between durations is implicit where it does not require truncation of the value (so converting hours to seconds is OK, but converting seconds to hours is not).Explicit conversions can be done with `std::chrono::duration_cast<>`
    * The result is truncated rather than rounded, so s will have a value of 54 in this example

      ```C++
      std::chrono::milliseconds ms(54802);
      std::chrono::seconds s=
          std::chrono::duration_cast<std::chrono::seconds>(ms);
      ```

  * Durations support arithmetic
    * `5*seconds(1)` is the same as `seconds(5)` or `minutes(1) – seconds(55)`;
  * The count of the number of units in the duration can be obtained with the count() member function
      * `std::chrono::milliseconds(1234).count()` is 1234;
  * Duration-based waits are done with instances of `std::chrono::duration<>`;
    * wait for up to 35 milliseconds for a future to be ready

      ```C++
      std::future<int> f = std::async(some_task);
      if(f.wait_for(std::chrono::milliseconds(35)) == std::future_status::ready)
       do_something_with(f.get());
      ```
    * The wait functions all return a status to indicate whether the wait timed out or the waitedfor event occurred;
      * if you are waiting for a future
        * the wait function returns `std::future_status::timeout` if the wait times out;
        * `std::future_status::ready` if the future is ready;
        * `std::future_status::deferred` if the future’s task is deferred;
    * The time for a duration-based wait is measured using a steady clock internal to the library，so 35 milliseconds means 35 milliseconds of elapsed time, even if the system clock was adjusted (forward or back) during the wait;
* Time points
  * the time point for a clock is represented by an instance of the `std::chrono::time_point<>` class template
    * the first template parameter specifies which clock it refers to;
    * the second template parameter specifies the units of measurement(a specialization of `std::chrono::duration<>`);
  * the value of a time point is the length of time (in multiples of the specified duration) since a specific point in time called the epoch of the clock(时间段);
    * Typical epochs include 00:00 on January 1, 1970 and the instant when the computer running the application booted up;
    * The epoch of a clock is a basic property but not something that’s directly available to query or specified by the C++ Standard;
    * Although you can’t find out when the epoch is, you can get the `time_since_epoch()` for a given time_point . This member function returns a duration value specifying the length of time since the clock epoch to that particular time point;
  * `std::chrono::time_point<std::chrono::system_clock, std::chrono::minutes>`would hold the time relative to the system clock but measured in minutes as opposed to the native precision of the system clock (which is typically seconds or less);
  * You can add durations and subtract durations from instances of `std::chrono::time_point<>` to produce new time points
    * `std::chrono::high_resolution_clock::now() + std::chrono::nanoseconds(500)` will give you a time 500 nanoseconds in the future;

      ```C++
      auto start=std::chrono::high_resolution_clock::now();
      do_something();
      auto stop=std::chrono::high_resolution_clock::now();
      std::cout<<”do_something() took “
        <<std::chrono::duration<double,std::chrono::seconds>(stop-start).count()
        <<” seconds”<<std::endl;
      ```

  * When you pass the time point to a wait function that takes an absolute timeout, the clock parameter of the time point is used to measure the time;
  * below is the recommended way to wait for condition variables with a time limit, if you’re not passing a predicate to the wait
    * predicate?

    ```C++
    #include <condition_variable>
    #include <mutex>
    #include <chrono>
    std::condition_variable cv;
    bool done;
    std::mutex m;
    bool wait_loop()
    {
      auto const timeout= std::chrono::steady_clock::now()+
      std::chrono::milliseconds(500);
      std::unique_lock<std::mutex> lk(m);
      while(!done)
      {
        if(cv.wait_until(lk,timeout)==std::cv_status::timeout)
        break;
      }
      return done;
    }
    ```

* Functions that accept timeouts
  * `std::this_thread::sleep_for()`: the thread goes to sleep for the specified duration;
  * `std::this_thread::sleep_until()`: the thread goes to sleep until the specified point in time;
  * `std::timed_mutex` and `std::recursive_timed_mutex` support timeouts on locking
    * Both these types support `try_lock_for()` and `try_lock_until()` member functions;
  * table(Functions that accept timeouts)

    ![Functions that accept timeouts](http://o7s72jtji.bkt.clouddn.com/Functions%20that%20accept%20timeouts)

* Using synchronization of operations to simplify code
  * The term functional programming ( FP ) refers to a style of programming where the result of a function call depends solely on the parameters to that function and doesn’t depend on any external state;
    * it means that if you invoke a function twice with the same parameters, the result is exactly the same;
  * FP - STYLE Q UICKSORT

    ![FP-style recursive sorting](http://o7s72jtji.bkt.clouddn.com/FP-style%20recursive%20sorting)

    ```C++
    template<typename T>
    std::list<T> sequential_quick_sort(std::list<T> input)
    {
      if(input.empty())
      {
        return input;
      }
      std::list<T> result;
      result.splice(result.begin(),input,input.begin());

      T const& pivot = *result.begin();        
      auto divide_point=std::partition(input.begin(),input.end(),
        [&](T const& t){return t<pivot;});    

      std::list<T> lower_part;
      lower_part.splice(lower_part.end(),input,input.begin(),divide_point);    

      auto new_lower(
        sequential_quick_sort(std::move(lower_part)));
      auto new_higher(
        sequential_quick_sort(std::move(input)));

      result.splice(result.end(),new_higher);
      result.splice(result.begin(),new_lower);

      return result;        
    }
    ```

    * `std::partition()` rearranges the list in place and returns an iterator marking the first element that’s not less than the pivot value;
    * using `std::move` to avoid copying;
  * FP - STYLE PARALLEL Q UICKSORT
    ```C++
    template<typename T>
    std::list<T> parallel_quick_sort(std::list<T> input)
    {
      if(input.empty())
      {
        return input;
      }
      std::list<T> result;
      result.splice(result.begin(),input,input.begin());

      T const& pivot=*result.begin();
      auto divide_point=std::partition(input.begin(),input.end(),
        [&](T const& t){return t<pivot;});

      std::list<T> lower_part;
      lower_part.splice(lower_part.end(),input,input.begin(),divide_point);

      std::future<std::list<T> > new_lower(
        std::async(&parallel_quick_sort<T>,std::move(lower_part)));

      auto new_higher(
        parallel_quick_sort(std::move(input)));

      result.splice(result.end(),new_higher);
      result.splice(result.begin(),new_lower.get());

      return result;
    }
    ```

    * rather than using `std::async()` , you could write your own spawn_task() function as a simple wrapper around `std::packaged_task`

      ```C++
      template<typename F,typename A>
      std::future<std::result_of<F(A&&)>::type>
      spawn_task(F&& f,A&& a)
      {
        typedef std::result_of<F(A&&)>::type result_type;
        std::packaged_task<result_type(A&&)>
        task(std::move(f)));
        std::future<result_type> res(task.get_future());
        std::thread t(std::move(task),std::move(a));
        t.detach();
        return res;
      }
      ```

    * Functional programming isn’t the only concurrent programming paradigm that eschews shared mutable data; another paradigm is CSP (Communicating Sequential Processes), where threads are conceptually entirely separate, with no shared data but with communication channels that allow messages to be passed between them;

* Synchronizing operations with message passing
  * The idea of CSP is simple: if there’s no shared data, each thread can be reasoned about entirely independently, purely on the basis of how it behaves in response to the messages that it received;

    ![A simple state machine model for an ATM](http://o7s72jtji.bkt.clouddn.com/A%20simple%20state%20machine%20model%20for%20an%20ATM)

    code here<https://github.com/Chorior/commonCodes/tree/master/C%2B%2B/state%20machine>
