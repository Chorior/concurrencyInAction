# Chapter 2: Managing threads
---
:art:
---
* 每个程序至少有一个线程,执行main()函数的线程;
* 当线程执行完入口函数之后,会退出线程;
* starting a thread using the C++ Thread Library always boils down to constructing a std::thread object
  * 使用C++线程库开启一个线程通常归结为构造一个`std::thread`类
  ```C++
  void do_some_work();
  std::thread my_thread(do_some_work);
  ```
* As with much of the C++ Standard Library, std::thread works with any callable type, so you can pass an instance of a class with a function call operator to the std::thread constructor instead
  * 和大多数C++标准库一样,`std::thread`可以用callable类型调用,因此你可以使用带有函数调用符的类的实例传入`std::thread`类中,替换默认的构造函数
  ```C++
  class background_task
  {
  public:
    void operator()() const
    {
      do_something();
      do_something_else();
    }
  };  
  background_task f;
  std::thread my_thread(f);
  ```
  * In this case, the supplied function object is copied into the storage belonging to the newly created thread of execution and invoked from there. It’s therefore essential that the copy behave equivalently to the original, or the result may not be what’s expected
    * 代码中,提供的函数对象被复制到新线程类的存储空间中,函数对象的执行和调用都在线程的内存空间中进行,这个函数对象的副本应与原对象保持一致,否则结果可能差强人意;
* 当传入一个函数对象到线程构造函数中时
    * 如果传递的是一个临时变量,C++编译器会将其解析为函数声明,而不是thread对象的定义;
    ```C++
    std::thread my_thread(background_task());
    ```
    * 解决方法
      * the extra parentheses prevent interpretation as a function
declaration, thus allowing my_thread to be declared as a variable of type std::thread
        * 使用多组圆括号;
      * uses the new uniform initialization syntax with braces rather than parentheses, and thus would also declare a variable      
        * 使用新统一的初始化语法(花括号);
      ```C++
      std::thread my_thread((background_task()));
      std::thread my_thread{background_task()};
      ```
      * lambda表达式
        * This is a new feature from C++11 which essentially allows you to write a local function, possibly capturing some local variables and avoiding the need of passing additional arguments
      ```C++
      std::thread my_thread([]{
        do_somthing();
        do_something_else();
      });
      ```
*      
