---
layout: post
title: 多线程基础——thread类
categories: C++ concrurrency
description: C++并发编程简介
keywords: c++, 并发编程,std::thread
---

# 多线程基础——thread类

- [参考：三种创建线程的方式](https://thispointer.com//c-11-multithreading-part-1-three-different-ways-to-create-threads/)
- [参考：join和detach线程](https://thispointer.com//c11-multithreading-part-2-joining-and-detaching-threads/)
- [参考：传递参数给线程](https://thispointer.com//c11-multithreading-part-3-carefully-pass-arguments-to-threads/)
- [参考：cpp11并发编程入门——futures](https://baptiste-wicht.com/posts/2017/09/cpp11-concurrency-tutorial-futures.html)

## 启动线程

C++11在<thread>头文件中提供了std::thread类在创建线程对象时启动线程。该类在创建对象的时候启动一个线程，其参数可以是以下三种：

- 函数指针
- lambda表达式
- 可调用的对象（仿函数——重载了operator()的类，bind之后的对象）


```cpp
class Cadd
{
public:
    int operator()(int a,int b){
        return a + b;
    }
};

int add(int a,int b)
{
    return a + b;
}

// 函数指针
// 注意：add是非类的成员函数时可以直接使用，thread会自动把他转换成函数指针
// 注意：如果add是类的成员函数，则需要使用取地址符号转换成函数指针
auto t = std::thread(add,1,2);

// 使用可调用对象
auto add_obj = Cadd();
auto t = std::thread(add_obj,1,2);

// lambda
std::thread my_thread([]{
  do_something();
  do_something_else();
});
```

**注意**：对于上面的Cadd类，如果直接使用其匿名对象传递给thread类将会被当做声明一个函数，而不是创建一个对象。可以使用括号将匿名对象包围起来，或者使用C++11新的统一初始化语句——中括号，如下：

```cpp
class op_class
{
public:
    void operator()()const{return;}
};

auto t = std::thread(Cadd(),1,2); // 不确定是否有问题，没验证~
std::thread my_thread(op_class());  // 这个是声明了一个函数，该函数接收一个op_class对象无返回值。

auto t = std::thread((Cadd()),1,2); // OK
auto t = std::thread((op_class));   // OK
```

## 传递线程参数

> Even if threadCallback accepts arguments as reference but still changes done it are not visible outside the thread.Its because x in the thread function threadCallback is reference to the temporary value copied at the new thread’s stack.

在传递线程函数参数的时候，如果是lambda表达式可以直接使用捕获，函数指针和可调用对象则可以在thread对象的第二个参数及以后进行指定。

**成员函数作为函数指针**

当成员函数作为函数指针创建线程时，需要在第二个参数指定使用哪个对象来调用该函数。第二个参数相当于指定了成员函数的this指针。示例如下：

```cpp
#include <thread>
#include <iostream>

class DummyClass {
public:
    DummyClass(const int a):m_count(a)
    {}
    DummyClass(const DummyClass & obj)
    {}
    void sampleMemberFunction(int x)
    {
        std::cout<<"Inside sampleMemberFunction "<<x<<std::endl;
        std::cout<<"Inside m_count "<< m_count <<std::endl;
    }
private:
    int m_count;
};

int main() {
 
    DummyClass dummyObj(99);
    int x = 10;
    std::thread threadObj(&DummyClass::sampleMemberFunction,&dummyObj, x);
    threadObj.join();
    return 0;
}
```

## 等待线程完成

- join():等待线程完成
- detach():分离线程，一般后台线程都可以使用detach进行分离，如守护线程
- joinable():是否可join，一般在调用join和detach之前进行合法性判断

**重点**

> If  neither join or detach is called with a std::thread object that has associated executing thread then during that object’s destruct-or it will terminate the program.Because inside the destruct-or it checks if Thread is Still Join-able then Terminate the program i.e.

> 如果一个线程既没有进行join，又没有进行detach，那么在这个线程相对应的线程对象析构的时候可能导致程序异常结束——因为如果一个线程对象在析构的时候如果任然是join-able的，那么它将会调用 terminate 结束程序！！！

**基于以上原因，不管是否在创建线程的时候进行了detach，都应该在持有thread对象类的析构函数中对该线程进行jionable判断及jion**

线程回调函数要保证其所使用到的数据是合法的。出现不合法访问的原因不外于访问的对象已经被析构或者释放，或者被意外的篡改。如果给线程通过引用传递局部变量的参数，那么在局部变量被析构之前应该要等待线程完成。
另外，如果不需要将线程分离，那么就需要在线程对象销毁的时候等待线程完成，一般只需要在类的析构函数中进行join即可，它既可以避免忘记join，又可以避免代码异常造成join被跳过的问题。以下为封装的线程的封装类：

```cpp
class thread_guard
{
  std::thread& t;
public:
  explicit thread_guard(std::thread& t_):
    t(t_)
  {}
  ~thread_guard()
  {
    if(t.joinable()) // 1
    {
      t.join();      // 2
    }
  }
  thread_guard(thread_guard const&)=delete;   // 3
  thread_guard& operator=(thread_guard const&)=delete;
};
```

## 线程所有权

### 线程所有权转移

thread类用于管理线程，每一个thread对象都拥有一个线程资源。thread对象只允许在对象之间转移，但是禁止在对象之间拷贝赋值，这样就保证了每一个thread对象拥有的线程都是独一无二的。虽然不可以拷贝，但是可以通过move将thread的资源进行转移。

**注意：**

- 不可以直接将一个thread对象赋值给另一个thread对象
- 如果要进行thread对象之间的赋值，那么可以使用move语义
- 如果一个thread对象本身已经有了一个关联的线程，再进行赋值的时候会造成程序崩溃。原因是系统直接调用std::terminate()终止程序继续运行。

**move语义**

- 对于匿名变量系统会自动使用move
- 显示的使用move需要实现operator=(class &&)操作

```cpp
auto t1 = std::thread(some_fun);    
auto t2 = std::thread(some_func);

auto t3 = std::move(t2);    // OK
t1 = std::move(t3);         // core dump,因为t1已经拥有了一个与之关联的线程

auto t4 = std::thread(some_fun);    
auto t5 = std::thread(some_func);
auto t6 = std::thread(some_func);
std::vector<std::thread> vecOfThreads;
vecOfThreads.push_back(std::move(t4));
vecOfThreads.push_back(std::move(t5));

//Destructor of already existing thread object will call terminate
vecOfThreads[1] = std::move(t6);
```

### 返回线程对象

前面说了，对于匿名对象，系统将自动调用move，因此除了显示的使用move返回thread对象，也可以通过匿名的对象返回thread对象。如下：

```cpp
std::thread create_thread_1()
{
    void fun();
    return std::thread(func);
}

std::thread create_thread_2()
{
    auto func = [](int a,int b){return a + b};
    auto t = std::thread(func,1,2);
    return move(t);
}
```

## 运行时线程数确定

对于多核CPU其能同时运行的最大线程数就是其CPU核心数目。C++11可以通过std::thread::hardware_concurrency()来获取能同时并发在一个程序中的线程数量，如果失败将返回0。

```c++
std::vector<std::thread> pool;
auto thread_count = std::thread::hardware_concurrency() == 0?4:std::thread::hardware_concurrency();
for(size_t i = 0; i < thread_count;++i)
{
  pool.emplace_back(std::thread(func,param_1,param_2));
}
```

## 管理当前线程

- std::this_thread: 这是一个对象，它实际就是一个static thread_local 的 thread 对象，因此可以获取当前处于调度状态的线程
- yield:让出处理器，重新调度各执行线程。yield主要用于频繁检查某一条件是否成立，如果不成立则放弃时间片等待下一次调度的情况。
- get_id:返回当前线程的线程 id
- sleep_for:使当前线程的执行停止指定的时间段
- sleep_until:使当前线程的执行停止直到指定的时间点

上面的几个函数都很好理解，这里重点介绍一下yield。在单核CPU上如果要执行多个程序主要依靠系统调度为每个进程分配CPU时间片，而CPU时间片的使用者就是线程。系统会根据当前系统的情况合理的为每个线程分配CPU时间片，只有被分配了时间片的线程才能执行CPU命令。如果一个线程需要满足某一条件之后才继续往下执行否则就继续等待，以前我们的实现一般就是在while里面进行条件检查，或者增加sleep、使用条件变量、事件等。这里我们又多了一个选择——yield。如下：

```cpp
while(!has_init_ok)
{
  this_thread::yield();
}

do_something();
```

当我们调用yield的时候就会放弃系统为该线程分配的CPU时间片，将当前CPU的时间片转让给其他线程。当这个时间片被消耗完毕之后，系统才会重新调度该线程。比如一个线程的时间片是执行N此++操作。

## 线程调度

这里说的线程调度不是指的系统对线程的调度情况，而是指的用户对当前线程的调度。一般的线程都是while循环执行的，直到某一条件才退出线程。在这个while循环中，可以根据业务的场景选择不同的方式来执行任务。

- 单消费者线程：单消费者线程中其他线程生产任务，该线程消费任务。一般可以使用条件变量进行通知。消费者线程等待条件，生产者添加任务的时候通过条件变化通知消费者。异步日志系统就是典型的单线程消费者情况。
- 定时任务线程：定时任务可以分为使用定时器的任务以及固定间隔的任务。对于固定间隔的任务最简单的方法
- 线程优雅退出：
- 多消费者线程：
- 加锁的粒度：


## thread_local 生存期对象

C++11引入了thread_local，具体作用可以参考《本地变量线程安全.md》