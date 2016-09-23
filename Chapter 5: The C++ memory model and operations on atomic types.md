# The C++ memory model and operations on atomic types
---
:art:
---
* Memory model basics
  * There are two aspects to the memory model
    * the basic structural aspects,which relate to how things are laid out in memory;
    * the concurrency aspects;
  * All data in a C++ program is made up of objects;
  * Whatever its type, an object is stored in one or more memory locations;

    ![memory model basics](https://github.com/Chorior/concurrencyInAction/blob/master/picture/memory%20model%20basics.png)

  * There are four important things to take away from this
    * Every variable is an object, including those that are members of other objects;
    * Every object occupies at least one memory location;
    * Variables of fundamental type such as int or char are exactly one memory location,  
      whatever their size, even if they’re adjacent or part of an array;
    * Adjacent bit fields are part of the same memory location;

  *  everything hinges on those memory locations.
    * if two threads access separate memory locations,there’s no problem: everything works fine.
    * if two threads access the same memory location, then you have to be careful.
      * if neither thread is updating the memory location, you’re fine;  
        read-only data doesn’t need protection or synchronization;
      * if either thread is modifying the data, there’s a potential for a race condition;
  * In order to avoid the race condition, there has to be an enforced ordering between the accesses in the two threads;
    * use mutexes;
    * use the synchronization properties of atomic operations either on the same or other memory locations;
  * if more than two threads access the same memory location, each pair of accesses must have a defined ordering;
  * using atomic operations doesn’t prevent the race itself,  
    which of the atomic operations touches the memory location first is still not specified,  
    but it does bring the program back into the realm of defined behavior;
  * Every object in a C++ program has a defined modification order  
    composed of all the writes to that object from all threads in the program,  
    starting with the object’s initialization;
  * Although all threads must agree on the modification orders of each individual object in a program,  
    they don’t necessarily have to agree on the relative order of operations on separate objects;

* Atomic operations and types in C++
  * An atomic operation is an indivisible operation;
  * You can’t observe such an operation half-done from any thread in the system; it’s either done or not done;
  * The flip side of this is that a nonatomic operation might be seen as half-done by another thread;
  * In C++, you need to use an atomic type to get an atomic operation in most cases;
  * The standard atomic types
    * `<atomic>`;
    * All operations on such types are atomic,   
      and only operations on these types are atomic in the sense of the language definition,  
      although you can use mutexes to make other operations appear atomic;
    * the standard atomic types (almost) all have an `is_lock_free()` member function
      * `x.is_lock_free()` returns true
        * operations on a given type are done directly with atomic instructions;
      * `x.is_lock_free()` returns false
        * operations on a given type are done by using a lock internal to the compiler and library;
    * The only type that doesn’t provide an `is_lock_free()` member function is `std::atomic_flag`;
      * This type is a really simple Boolean flag;
      * operations on this type are required to be lock-free;
      * objects of type `std::atomic_flag` are initialized to clear,  
        and they can then either be queried and set (with the `test_and_set()` member function)  
        or cleared (with the `clear()` member function);
      * no assignment, no copy construction, no test and clear, no other operations at all;
    * The remaining atomic types are all accessed through specializations of the `std::atomic<>` class template  
      and are a bit more full-featured but may not be lockfree (as explained previously);

      ![Table 5.1](https://github.com/Chorior/concurrencyInAction/blob/master/picture/Table%205.1.png)

    * because of the history of how atomic types were added to the C++ Standard,  
      these alternative type names may refer either to the corresponding `std::atomic<>` specialization  
      or to a base class of that specialization.  
    * Mixing these alternative names with direct naming of `std::atomic<>` specializations in the same program  
      can therefore lead to nonportable code;

      ![Table 5.2](https://github.com/Chorior/concurrencyInAction/blob/master/picture/Table%205.2.png)

    * The standard atomic types are not copyable or assignable in the conventional sense,  
      in that they have no copy constructors or copy assignment operators;
    * each of the operations on the atomic types has an optional memory-ordering argument  
      that can be used to specify the required memory-ordering semantics;
    * the operations are divided into three categories
      * Store operations
        * `memory_order_relaxed`;
        * `memory_order_release`;
        * `memory_order_seq_cst`;
      * Load operations
        * `memory_order_relaxed`;
        * `memory_order_consume`;
        * `memory_order_acquire`;
        * `memory_order_seq_cst`;
      * Read-modify-write operations
        * `memory_order_relaxed`;
        * `memory_order_consume`;
        * `memory_order_acquire`;
        * `memory_order_release`;
        * `memory_order_acq_rel`;
        * `memory_order_seq_cst`;
    * The default ordering for all operations is `memory_order_seq_cst`;
    * `std::atomic_flag`
      * the simplest standard atomic type;
      * represents a Boolean flag;
      * Objects of this type can be in one of two states: set or clear;
      * Objects of this type must be initialized with `ATOMIC_FLAG_INIT`;
        * This initializes the flag to a clear state;
      * It’s the only atomic type to require such special treatment for initialization,  
        but it’s also the only type guaranteed to be lock-free;
      * If the `std::atomic_flag` object has static storage duration, it’s guaranteed to be statically initialized;
      * Once you have your flag object initialized, there are only three things you can do with it
        * destroy it;
        * clear it;
        * set it and query the previous value;
      * Both the `clear()` and `test_and_set()` member functions can have a memory order specified;
        * `clear()` is a store operation;
        * `test_and_set()` is a readmodify-write operation;
        * the default for both is `memory_order_seq_cst`;

          ```C++
          f.clear(std::memory_order_release);
          bool x=f.test_and_set();
          ```

        * the call to `clear()` explicitly requests that the flag is cleared with release semantics;
        * the call to `test_and_set()` uses the default memory ordering for setting the flag and retrieving the old value;
      * all operations on an atomic type are defined as atomic, and assignment and copy-construction involve two objects;
        * A single operation on two distinct objects can’t be atomic;
        * These are two separate operations on two separate objects, and the combination can’t be atomic;

          ```C++
          class spinlock_mutex
          {
            std::atomic_flag flag;

          public:

            spinlock_mutex():
              flag(ATOMIC_FLAG_INIT)
            {}

            void lock()
            {
              while(flag.test_and_set(std::memory_order_acquire));
            }

            void unlock()
            {
              flag.clear(std::memory_order_release);
            }
          };
          ```

    * `std::atomic<bool>`
      * The most basic of the atomic integral types;
      * you can construct it from a nonatomic bool;
      * you can also assign to instances of `std::atomic<bool>` from a nonatomic bool;

        ```C++
        std::atomic<bool> b(true);
        b=false;
        ```

      * the assignment operator from a nonatomic bool returns a bool with the value assigned;
      * writes (of either true or false) are done by calling `store()`(not `clear()`);
      * `exchange()`(not `test_and_set()`) allows you to replace the stored value  
        with a new one of your choosing and atomically retrieve the original value;
      * `std::atomic<bool>` also supports a plain nonmodifying query of the value  
        with an implicit conversion to plain bool or with an explicit call to `load()`;
      * `store()` is a store operation, whereas `load()` is a load operation, `exchange()` is a read-modify-write operation;

        ```C++
        std::atomic<bool> b;
        bool x=b.load(std::memory_order_acquire);
        b.store(true);
        x=b.exchange(false,std::memory_order_acq_rel);
        ```

      * storing a new value (or not) depending on the current value
        * this new operation is called compare/exchange;
        * it comes in the form of the `compare_exchange_weak()` and `compare_exchange_strong()` member functions;
        * The compare/exchange operation is the cornerstone of programming with atomic types
          * it compares the value of the atomic variable with a supplied expected value;
          * stores the supplied desired value if they’re equal;
          * else the expected value is updated with the actual value of the atomic variable;
          * The return type of the compare/exchange functions is a bool
            * true if the store was performed;
            * false otherwise;
        * `compare_exchange_weak()`
          * spurious failure: the store might not be successful even if the original value was equal to the expected value,  
            in which case the value of the variable is unchanged  
            and the return value is false while not modifying the expected value;
          * it must typically be used in a loop;

            ```C++
            bool expected=false;
            extern atomic<bool> b; // set somewhere else
            while(!b.compare_exchange_weak(expected,true) && !expected);
            ```

        * `compare_exchange_strong()`
          * it is guaranteed to return false only if the actual value wasn’t equal to the expected value;
        * The compare/exchange functions are also unusual in that they can take two memoryordering parameters;
          * This allows for the memory-ordering semantics to differ in the case of success and failure;
          * If you don’t specify an ordering for failure, it’s assumed to be the same as that for success,  
            except that the release part of the ordering is stripped
            * `memory_order_release` becomes `memory_order_relaxed`;
            * `memory_order_acq_rel` becomes `memory_order_acquire`;
          * If you specify neither, they default to `memory_order_seq_cst` as usual;
          * The following two calls to `compare_exchange_weak()` are equivalent

            ```C++
            std::atomic<bool> b;
            bool expected;
            b.compare_exchange_weak(expected,true,
             memory_order_acq_rel,memory_order_acquire);
            b.compare_exchange_weak(expected,true,memory_order_acq_rel);
            ```

    * `std::atomic<T*>`
      * The new operations provided by `std::atomic<T*>` are the pointer arithmetic operations;
        * `fetch_add()`: do atomic addition on the stored address;
        * `fetch_sub()`: do atomic subtraction on the stored address;
        * ++,--,+=,-=;
      * `fetch_add()` and `fetch_sub()` update the pointer but return the original value;

        ```C++
        class Foo{};
        Foo some_array[5];
        std::atomic<Foo*> p(some_array);
        Foo* x=p.fetch_add(2);
        assert(x==some_array);
        assert(p.load()==&some_array[2]);
        x=(p-=1);
        assert(x==&some_array[1]);
        assert(p.load()==&some_array[1]);
        ```

      * The function forms also allow the memory-ordering semantics to be specified as an additional function call argument

        ```C++
        p.fetch_add(3,std::memory_order_release);
        ```

      * both `fetch_add()` and `fetch_sub()` are read-modify-write operations;
      * Specifying the ordering semantics isn’t possible for the operator forms
        * because there’s no way of providing the information;
        * these forms therefore always have `memory_order_seq_cst` semantics;
    * standard atomic integral types
      * usual set
        * `load()`;
        * `store()`;
        * `exchange()`;
        * `compare_exchange_weak()`;
        * `compare_exchange_strong()`;
      * comprehensive set
        * `fetch_add()`;
        * `fetch_sub()`;
        * `fetch_and()`;
        * `fetch_or()`;
        * `fetch_xor()`;
      * compound-assignment forms
        * +=, -=, &=, |=, ^=
        * ++, --
      * additional operations can easily be done using `compare_exchange_weak()` in a loop, if required;
    * The `std::atomic<>` primary class template
      * `std::atomic<>` allows a user to create an atomic variant of a user-defined type
        * in order to use `std::atomic<UDT>` for some user-defined type UDT
          * this type must have a trivial copy-assignment operator
            * the type must not have any virtual functions or virtual base classes;
            * the type must use the compiler-generated copy-assignment operator;
          * every base class and non-static data member of a userdefined type must also have a trivial copy-assignment operator;
          * the type must be bitwise equality comparable
            * you must be able to copy an object of type UDT using `memcpy()`;
            * you must be able to compare instances for equality using `memcmp()`;
            * This guarantee is required in order for compare/exchange operations to work;
        * the compiler isn’t going to be able to generate lock-free code for `std::atomic<UDT>`
          * you will have to use internal lock for all the operations;
      * there are no atomic arithmetic operations on floating-point values;
      * If your UDT is the same size as (or smaller than) an int or a `void*`,  
        most common platforms will be able to use atomic instructions for `std::atomic<UDT>`;
      * Some platforms will also be able to use atomic instructions for user-defined types  
        that are twice the size of an int or `void*`;
        * These platforms are typically those that support a so-called double-word-compare-and-swap (DWCAS) instruction  
          corresponding to the `compare_exchange_xxx` functions;
    * Free functions for atomic operations
      * there are also equivalent nonmember functions for all the operations on the various atomic types;

        ![Table 5.3](https://github.com/Chorior/concurrencyInAction/blob/master/picture/Table%205.3.png)

      * Where there’s opportunity for specifying a memoryordering tag, they come in two varieties
        * one without the tag;
        * one with an `_explicit` suffix and an additional parameter or parameters for the memory-ordering tag or tags;
          * `std::atomic_store(&atomic_var,new_value)`;
          * `std::atomic_store_explicit(&atomic_var,new_value,std::memory_order_release)`;
          * `std::atomic_is_lock_free(&a)` returns the same value as `a.is_lock_free()` for an object of atomic type a;
          * `std::atomic_load(&a)` is the same as `a.load()`;
          * `a.load(std::memory_order_acquire)` equal to `std::atomic_load_explicit(&a, std::memory_order_acquire)`;
      * The operations on `std::atomic_flag` buck the trend, in that they spell out the “flag” part in the names
        * `std::atomic_flag_test_and_set()`;
        * `std::atomic_flag_clear()`;
        * `std::atomic_flag_test_and_set_explicit()`;
        * `std::atomic_flag_clear_explicit()`;
      * The C++ Standard Library also provides free functions for accessing instances of `std::shared_ptr<>`  
        in an atomic fashion, which taking a `std::shared_ptr<>*` as the first argument;

        ```C++
        std::shared_ptr<my_data> p;
        void process_global_data()
        {
          std::shared_ptr<my_data> local=std::atomic_load(&p);
          process_data(local);
        }
        void update_global_data()
        {
          std::shared_ptr<my_data> local(new my_data);
          std::atomic_store(&p,local);
        }
        ```

  * Synchronizing operations and enforcing ordering

    ```C++
    #include <vector>
    #include <atomic>
    #include <iostream>
    std::vector<int> data;
    std::atomic<bool> data_ready(false);
    void reader_thread()
    {
      while(!data_ready.load())
      {
        std::this_thread::sleep(std::milliseconds(1));
      }
      std::cout<<”The answer=”<<data[0]<<”\n”;
    }
    void writer_thread()
    {
      data.push_back(42);
      data_ready=true;
    }
    ```

    ![Figure 5.2](https://github.com/Chorior/concurrencyInAction/blob/master/picture/Figure%205.2.png)

    * memory model relations synchronizes-with
      * the synchronizes-with relationship is something that you can get only between operations on atomic types;
      * all operations on atomic types are suitably tagged by default;
      * if thread A stores a value and thread B reads that value,  
        there’s a synchronizes-with relationship between the store in thread A and the load in thread B;
    * memory model relations happens-before
      * The happens-before relationship is the basic building block of operation ordering in a program;
      * it specifies which operations see the effects of which other operations;
      * if one operation (A) occurs in a statement prior to another (B) in the source code, then A happens-before B;
      * code below will output “1,2” or “2,1”,because the order of the two calls to `get_num()` is unspecified

        ```C++
        #include <iostream>
        void foo(int a,int b)
        {
          std::cout<<a<<”,”<<b<<std::endl;
        }
        int get_num()
        {
          static int i=0;
          return ++i;
        }
        int main()
        {
          foo(get_num(),get_num());
        }
        ```

      * in general, operations within a single statement are nonsequenced,  
        and there’s no sequenced-before (and thus no happens-before) relationship between them;
      * if operation A on one thread inter-thread happens-before operation B on another thread, then A happens-before B;
      * inter-thread happens-before is relatively simple and relies on the synchronizes-with relationship
        * if operation A in one thread synchronizes-with operation B in another thread, then A inter-thread happensbefore B;
        * if A inter-thread happens-before B and B interthread happens-before C, then A inter-thread happens-before C;
      * Inter-thread happens-before also combines with the sequenced-before relation
        * if operation A is sequenced before operation B, and operation B inter-thread happensbefore operation C,  
          then A inter-thread happens-before C;
        * if A synchronizeswith B and B is sequenced before C, then A inter-thread happens-before C;
    * Memory ordering for atomic operations
      * There are six memory ordering options that can be applied to operations on atomic types
        * `memory_order_relaxed`;
        * `memory_order_consume`;
        * `memory_order_acquire`;
        * `memory_order_release`;
        * `memory_order_acq_rel`;
        * `memory_order_seq_cst`;
      * `memory_order_seq_cst` is the default memory-ordering option for all operations on atomic types;
      * six ordering options represent three models
        * sequentially consistent ordering
          * `memory_order_seq_cst`;
        * acquire-release ordering
          * `memory_order_consume`;
          * `memory_order_acquire`;
          * `memory_order_release`;
          * `memory_order_acq_rel`;
        * relaxed ordering
          * `memory_order_relaxed`;
      * These distinct memory-ordering models can have varying costs on different CPU architectures;
      * sequentially consistent ordering
        * Sequential consistency is the most straightforward and intuitive ordering;
        * but it’s also the most expensive memory ordering because it requires global synchronization between all threads;
        * If all operations on instances of atomic types are sequentially consistent,  the behavior of a multithreaded program  
          is as if all these operations were performed in some particular sequence by a single thread;
        * if your code has one operation before another in one thread, that ordering must be seen by all other threads;
        * you must use sequentially consistent operations on all your threads;

          ```C++
          #include <atomic>
          #include <thread>
          #include <assert.h>
          std::atomic<bool> x,y;
          std::atomic<int> z;
          void write_x()
          {
            x.store(true,std::memory_order_seq_cst);
          }
          void write_y()
          {
            y.store(true,std::memory_order_seq_cst);
          }
          void read_x_then_y()
          {
            while(!x.load(std::memory_order_seq_cst));
            if(y.load(std::memory_order_seq_cst))
            ++z;
          }
          void read_y_then_x()
          {
            while(!y.load(std::memory_order_seq_cst));
            if(x.load(std::memory_order_seq_cst))
            ++z;
          }
          int main()
          {
            x=false;
            y=false;
            z=0;
            std::thread a(write_x);
            std::thread b(write_y);
            std::thread c(read_x_then_y);
            std::thread d(read_y_then_x);
            a.join();
            b.join();
            c.join();
            d.join();
            assert(z.load()!=0);
          }
          ```

        * either the store to x B or the store to y must happen first, even though it’s not specified which;
        * for there to be a single total order
          * if one thread sees `x==true` and then subsequently sees `y==false`,  
            this implies that the store to x occurs before the store to y in this total order;
          * otherwise the opposite;

            ![Figure 5.3](https://github.com/Chorior/concurrencyInAction/blob/master/picture/Figure%205.3.png)

      * non-sequentially consistent memory orderings
