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
