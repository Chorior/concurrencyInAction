# Chapter 4: Synchronizing concurrent operations
---
:art:
---
* 等待一个事件完成
  * 持续检查完成标志,直至完成;
    * 消耗不必要的时间检查;
    * 上锁时,其它需要该锁的线程会处于等待状态,消耗系统资源;
  * 在等待完成期间,使用`std::this_thread::sleep_for()`函数进行周期时间间歇;

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
    * 难以确定合适的休息时间;
  * 使用Ｃ++标准库工具等待时间本身(最好的选择)
    * The most basic mechanism for waiting for an event to be triggered by another thread is the condition variable;
      * 等待事件被另一个线程触发最基本的措施是使用条件变量;
    * Conceptually, a condition variable is associated with some event or other condition, and one or more threads can wait for that condition to be satisfied. When some thread has determined that the condition is satisfied, it can then notify one or more of the threads waiting on the condition variable, in order to wake them up and allow them to continue processing;
      * 从概念上讲,一个条件变量与多个事件或其它条件相关,并且一个或更多线程会等待那个条件的达成;当某些线程确定那个条件达成时,它会通知一个或多个在条件变量上等待的线程,继而唤醒它们,继续工作;
