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
  #include <iostream>
  #include <thread>

  class Hello
  {
  private:
  public:
  	Hello() = default;
  	~Hello() = default;

  	inline void funcHello() const
  	{
  		std::cout << "Hello concurrent world!\n";
  	}

  	void operator()() const
  	{
  		funcHello();
  	}
  };

  int main()
  {
  	const Hello test_hello;
  	std::thread t(test_hello);
  	t.join();

  	return EXIT_SUCCESS;
  }
  ```
  * In this case, the supplied function object is copied into the storage belonging to the newly created thread of execution and invoked from there. It’s therefore essential that the copy behave equivalently to the original, or the result may not be what’s expected
    * 代码中,提供的函数对象被复制到新线程类的存储空间中,函数对象的执行和调用都在线程的内存空间中进行,这个函数对象的副本应与原对象保持一致,否则结果可能差强人意;
* 当传入一个函数对象到线程构造函数中时
    * 如果传递的是一个临时变量,C++编译器会将其解析为函数声明,而不是thread对象的定义;
    ```C++
    std::thread my_thread(Hello());
    ```
    * 解决方法
      * the extra parentheses prevent interpretation as a function
declaration, thus allowing my_thread to be declared as a variable of type std::thread
        * 使用多组圆括号;
      * uses the new uniform initialization syntax with braces rather than parentheses, and thus would also declare a variable      
        * 使用新统一的初始化语法(花括号);
      ```C++
      std::thread my_thread((Hello()));
      std::thread my_thread{Hello()};
      ```
      * lambda表达式
        * This is a new feature from C++11 which essentially allows you to write a local function, possibly capturing some local variables and avoiding the need of passing additional arguments

      ```C++
      #include <iostream>
      #include <thread>

      inline void funcHello()
      {
      	std::cout << "Hello concurrent world!\n";
      }

      int main()
      {
      	std::thread t([]{ funcHello();});
      	t.join();

      	return EXIT_SUCCESS;
      }

      ```
* **一旦启动线程,就需要明确决定是要等待线程结束(join),还是让它自主运行(detach)**;    
  * 如果在`std::thread`对象销毁之前还没做出决定,程序就会终止,因为`std::thread`析构函数会调用`std::terminate()`;
  * 要确保线程被正确的join或detach,即使有异常存在也不例外;
  * the thread itself may well have finished long before you join with it or detach it, and if you detach it, then the thread may continue running long after the std::thread object is destroyed.
    * 如果在线程结束之后分离它,那么该线程可能会在`std::thread`对象销毁之后继续运行;
* 如果采用detach模式,你需要确保这个线程访问的数据是有效的,在它结束之前;下面这个case是极其危险的,它在some_local_state被销毁后依旧去调用它

```C++
  struct func
  {
    int& i;
    func(int& i_):i(i_){}
    void operator()()
    {
       for(unsigned j=0;j<1000000;++j)
       {
       do_something(i);
       }
    }
  };
  void oops()
  {
     int some_local_state=0;
     func my_func(some_local_state);
     std::thread my_thread(my_func);
     my_thread.detach();
  }
```

* 为使detach线程数据一直有效,你可以使该线程功能齐全,并将所有所需数据都复制到该线程存储空间中;
* it’s a bad idea to create a thread within a function that has access to the local variables in that function, unless the thread is guaranteed to finish before the function exits;
* The act of calling join() also cleans up any storage associated with the thread, so the std::thread object is no longer associated with the nowfinished
thread; it isn’t associated with any thread.
  * 调用join()清理了与线程相关的存储部分,所以这个`std::thread`对象不再与任何线程相关联直到重新赋值;

  ```C++
  #include <iostream>
  #include <thread>

  inline void funcHello()
  {
  	std::cout << "Hello concurrent world!\n";
  }

  int main()
  {
  	std::thread t([]{ funcHello();});
  	t.join();

  	if(!t.joinable())
  	{
  		std::cout << "thread t is unjoinable\n";
  	}

  	std::thread t1(funcHello);
  	t = std::move(t1);
  	if(t.joinable())
  	{
  		std::cout << "thread t is joinable!\n";
  		t.join();
  	}

  	return EXIT_SUCCESS;
  }
  ```

* 对于一个线程只能调用一次join(),这时调用joinable()会返回false;
* 如果要加入一个线程,需要注意调用join()的位置
  * 如果在线程启动之后、join()调用之前,线程抛出了一个异常,那么join()调用会被跳过;
  * 为了防止异常抛出之后程序终止,需要在异常处理过程中也调用join()

  ```C++
  struct func;
  void f()
  {
     int some_local_state=0;
     func my_func(some_local_state);
     std::thread t(my_func);
     try
     {
       do_something_in_current_thread();
     }
     catch(...)
     {
       t.join();
       throw;
     }
     t.join();
  }
  ```
  * One way of doing this is to use the standard Resource Acquisition Is Initialization (RAII) idiom and provide a class that does the join() in its destructor

  ```C++
  class thread_guard
  {
    std::thread& t;
  public:
     explicit thread_guard(std::thread& t_):
     t(t_)
     {}
     ~thread_guard()
     {
       if(t.joinable())
       {
         t.join();
       }
     }
     thread_guard(thread_guard const&)=delete;
     thread_guard& operator=(thread_guard const&)=delete;
  };
  struct func;
  void f()
  {
     int some_local_state=0;
     func my_func(some_local_state);
     std::thread t(my_func);
     thread_guard g(t);
     do_something_in_current_thread();
  }
  ```

*  This(call detach) breaks the association of the thread with the std::thread object and ensures that std::terminate() won’t be called when the std::thread object is destroyed, even though the thread is still running in the background.
  * 分离操作切断了线程与`std::thread`的关联,并且保证了在这个`std::thread`对象被销毁的时候不会调用`std::terminate()`,即使这个线程在后台运行;
* if a thread becomes detached, it isn’t possible to obtain a std::thread object that references it, so it can no longer be joined.
  * 调用了detach就不能再调用join()了;
* 分离线程也被称为守护线程(daemon threads);
* 调用join()或detach()之前,必须确保`std::thread`对象关联到一个线程,方法是检查joinable()的返回是否为true;
* 向线程函数传递参数
  * 如果参数需要转换才能匹配函数,最好使用显式转换,因为默认转换也许在没有转换成功之前住线程就结束了
    * 就算是显式转换,`std::thread`构造函数也智慧复制转换前的值到线程存储空间;

  ```C++
  #include <iostream>
  #include <thread>
  #include <string>

  inline void funcHello(std::string str)
  {
  	std::cout << "string = " << str << std::endl;
  }

  int main(int argc, char *argv[])
  {
  	if(2 == argc)
  	{
  		std::thread t(funcHello,std::string(argv[1]));
  		t.join();
  	}
  	else
  	{
  		std::cout << "please input two arguments!\n";
  	}

  	return EXIT_SUCCESS;
  }
  ```

  * __如果线程参数是引用类型,传递参数时,一定要使用`std::ref()`__;
    * 如果不使用,那么会编译错误或者会引用传递参数的拷贝;

  ```C++
  #include <iostream>
  #include <thread>
  #include <string>

  inline void funcHello(std::string &str)
  {
  	std::cout << "string = " << str << std::endl;
  }

  int main()
  {
  	std::string str("Hello concurrent world!");

  	std::thread t(funcHello,std::ref(str));
  	t.join();

  	return EXIT_SUCCESS;
  }
  ```

  * 如果线程函数是成员函数,那么需要传递一个合适的对象实例指针,后跟成员函数参数

  ```C++
  #include <iostream>
  #include <thread>
  #include <string>

  class Hello
  {
  private:
  public:
  	Hello() = default;
  	~Hello() = default;

  	inline void funcHello(std::string &str)
  	{
  		std::cout << "string = " << str << std::endl;
  	}
  };

  int main()
  {
  	Hello test_hello;

  	std::string str("Hello concurrent world!");

  	std::thread t(&Hello::funcHello,&test_hello,std::ref(str));
  	t.join();

  	return EXIT_SUCCESS;
  }
  ```

  * 如果传递的参数只能移动(如`std::unique_ptr指针`),那么需要调用`std::move()`

  ```C++
  void process_big_object(std::unique_ptr<big_object>);
  std::unique_ptr<big_object> p(new big_object);
  p->prepare_data(42);
  std::thread t(process_big_object,std::move(p));
  ```

* 传递线程所有权
  * This ownership can be transferred between instances, because instances of std::thread are movable, even though they aren’t copyable. This ensures that only one object is associated with a particular thread of execution at any one time while allowing programmers the option of transferring that ownership between objects.
    * `std::thread`对象只能移动,不可拷贝,因为它要确保在任何时间,一个执行线程只能被一个`std::thread`对象关联;

  ```C++
  void some_function();
  void some_other_function();
  std::thread t1(some_function);
  std::thread t2=std::move(t1);
  t1=std::thread(some_other_function);
  std::thread t3;
  t3=std::move(t2);
  t1=std::move(t3);
  ```
  * 不能赋一个新值给一个已经关联到一个线程的`std::thread`对象(如上述程序的最后一句),那样会调用析构函数中的`std::terminate()`;
  * 移动操作可以真实拥有线程的所有权,下面程序传递了临时`std::thread`对象的所有权

  ```C++
  class scoped_thread
  {
    std::thread t;
  public:
     explicit scoped_thread(std::thread t_):
     t(std::move(t_))
     {
       if(!t.joinable())
       throw std::logic_error(“No thread”);
     }
     ~scoped_thread()
     {
       t.join();
     }
     scoped_thread(scoped_thread const&)=delete;
     scoped_thread& operator=(scoped_thread const&)=delete;
  };
  struct func;
  void f()
  {
     int some_local_state;
     scoped_thread t(std::thread(func(some_local_state)));
     do_something_in_current_thread();
  }
  ```

  * The move support in std::thread also allows for containers of std::thread objects, if those containers are move aware (like the updated std::vector<>).
    * 可以用容器存储`std::thread`对象,如果容器是移动敏感的,如`std::vector()`;

  ```C++
  void do_work(unsigned id);
  void f()
  {
     std::vector<std::thread> threads;
     for(unsigned i=0;i<20;++i)
     {
       threads.push_back(std::thread(do_work,i));
     }
     std::for_each(threads.begin(),threads.end(),
                    std::mem_fn(&std::thread::join)); // Call join() on each thread in turn
  }
  ```

  * If f() were to return a value to the caller that depended on the results of the operations performed by these threads, then as written this return value would have to be determined by examining the shared data after the threads had terminated.
    * 如果f()函数有返回值,并且该返回值与启动的线程执行的结果有关,那么在写入返回值之前,程序会检查使用的共享数据的线程是否终止;
* `std::thread::hardware_concurrency()` returns an indication of the number of threads that can truly run concurrently for a given execution of a program.On a multicore system it might be the number of CPU cores, for example. This is only a hint, and the function might return 0 if this information is not available, but it can be a useful guide for splitting a task among threads.
  * `std::thread::hardware_concurrency()`返回能同时并发在一个程序中的线程数;
* 两种方式检索线程标识`std::thread::id`
  * 使用`std::thread`对象调用`get_id()`;
    * 如果未关联线程,那么将返回`std::threead::id`类的默认构造值,代表木有线程;
  * 调用`std::this_thread::get_id()`(这个函数同样包含在`<thread>`中);
* the only guarantee given by the standard is that thread IDs that compare as equal should produce the same output, and those that are not equal should give different output.  
  * 如果两个线程ID相同,那么它们应该有相同的输出,不同则应该有不同的输出;
* Objects of type std::thread::id can be freely copied and compared;
* Thread IDs could be used as keys into associative containers where specific data needs to be associated with a thread and alternative mechanisms such as thread-local storage aren’t appropriate;
* You can even write out an instance of std::thread::id to an output stream such as std::cout

```C++
std::cout<<std::this_thread::get_id();
```
