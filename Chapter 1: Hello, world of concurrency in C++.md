# Chapter 1: Hello, world of concurrency in C++
---
:art:
---
* 并发: 两个或更多独立的活动同时发生;
* 任务切换: 每个任务交织进行,但切换会有时间开销;
* 硬件并发(hardware concurrency): 真正的并行多个任务;
* 即便是具有真正硬件并发的系统，有时候也会需要任务切换;
* 使用并发
  * 将应用程序分为多个独立的进程,它们在同一时刻运行;
    * 缺点
      * 进程间通信设置复杂,速度慢,因为系统在进程间提供了很多保护措施用以保护数据;
      * 运行多个进程需要固定开销;
    * 优势
      * 更容易编写安全(safe)的并发代码;
      * 可以使用远程连接(可能需要联网)，在不同的机器上运行独立的进程;
  * 在单个进程中运行多个线程
    * 进程中的所有线程都共享地址空间;
    * 全局变量仍然是全局的，指针、对象的引用或数据可以在线程之间传递;
    * 使用多线程相关的开销远远小于使用多个进程;
    * 必须确保每个线程所访问到的数据是一致的;
* C++标准并未对进程间通信提供任何原生支持，所以使用多进程的方式实现，这会依赖与平台相关的API;
* 使用并发的原因
  * 关注点分离: separation of concerns(SOC);
  * 性能: performance;
    * 两种方式利用并发提高性能
      * 任务并行: 将一个单个任务分成几部分;
      * 数据并行: 一个线程执行算法的一部分，而另一个线程执行算法的另一个部分;或者每个线程在不同的数据部分上执行操作;
* 不使用并发的唯一原因: the benefit is not worth the cost;
* 简单的并发程序
  ```C++
  #include <iostream>
  #include <thread>

  void hello() { std::cout << "Hello Concurrent World!\n"; }
  int main()
  {
    std::thread t(hello);
    t.join();
  }
  ```
* 管理线程的函数和类在`<thread>`中声明，保护共享数据的函数和类在其他头文件中声明;
* If software is to take advantage of this increased computing power, it must be designed to run multiple tasks concurrently.
