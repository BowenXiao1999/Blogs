# C++ Concurrency in Action Notes

## 2 Thread Management
* Use RAII to ensure thread finish even Exception (Impl)
* Pass parameter safely (conversion happens in thread context, so do conversion in current thread) (Impl)
    * Use std::ref to pass real reference
    * execute a object function
* Transfer the ownership of thread. 
* Change thread number in run time.

## 3 Shared Data
* Use mutex to protect shared data.
* Do not pass pointer or reference.
* C++ STL API do not thread safe -> Implement a thread safe API
* Avoid Dead Lock
    * acquire lock in same order
    * When multiple mutex protect one instance (swap): use std::lock and std::adopt_lock (Impl)
    * Use hierachy lock (Impl)
* unique_lock more flexible.
* Reduce lock granularity.
* What to do when not use mutex lock
    * std::call_once() (Execute Once)
    * boost::shared_mutex (R/W Lock)
* Recursive Lock.
