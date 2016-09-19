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
