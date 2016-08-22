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
* C++标准库提供另个条件变量的实现
  * 头文件: `<condition_variable>`;
  * `std::condition_variable`;
    * 只能与`std::mutex`一起使用;
  * `std::condition_variable_any`;
    * can work with anything that meets some minimal criteria for being mutex-like, hence the `_any` suffix;
    * 比`std::condition_variable`更费资源
  * In both cases, they need to work with a mutex in order to provide appropriate synchronization;
    * 它们都需要一个互斥量进行同步;
  * `std::condition_variable`更受欢迎,除非灵活性需要;
  * 示例

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
