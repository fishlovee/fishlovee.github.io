---
layout: post
title: 高级并发接口:future、promise、 package_task 和 async
categories: cpp concurrency cpp_concurrency
description: C++并发编程简介
keywords: c++, 并发编程,std::future,std::package_task,std::promise,std::async
---

- [参考：C++11之获取线程返回值](https://thispointer.com//c11-multithreading-part-8-stdfuture-stdpromise-and-returning-values-from-thread/)
- [C++11多线程-异步运行(1)之std::promise](https://www.jianshu.com/p/7945428c220e)
- [C++11 并发指南四(<future> 详解一 std::promise 介绍)](https://www.cnblogs.com/haippy/p/3239248.html)
- [C++11 Multithreading – Part 8: std::future , std::promise and Returning values from Thread](https://thispointer.com/c11-multithreading-part-8-stdfuture-stdpromise-and-returning-values-from-thread/)
- [C++11 并发编程系列(四)：异步操作(future)](https://murphypei.github.io/blog/2019/04/cpp-concurrent-4)
- [csdn:C++11新特性之 std::future and std::async](https://blog.csdn.net/wangshubo1989/article/details/49872199)
- [C++11 Concurrency Tutorial - Part 5: Futures](https://baptiste-wicht.com/posts/2017/09/cpp11-concurrency-tutorial-futures.html)


在C++11中我们可以很方便的创建线程，如下：

```cpp
int32_t sum(std::vector<int> data)
{
    int32_t s = 0;
    for(const auto &d:data)
    {
        s+=d;
    }
    return s;
}

std::vector<int> data{1,2,3,4,5};
std::thread t(sum,data);
```

虽然我们让线程运行起来了，但是如何才能拿到我们想要的结果呢？答案是future。通过async、packege_tasks、promise我们可以得到一个future对象，从而得到线程的执行状态以及返回结果。而package_task则可以用来将一个可调用对象打包成一个可以异步调用的对象，利用packet_task可以很方便的写出接收任意任务的线程池，这部分在后面再讲。先看如何使用async、promise和future异步执行和获取异步结果。

## 使用promise获取异步结果

promise可以通过get_future返回一个future对象。通过future可以异步的等待或者获取promise在未来设置的结果。

> If std::promise object is destroyed before setting the value the calling get() function on associated std::future object will throw exception.A part from this, if you want your thread to return multiple values at different point of time then just pass multiple std::promise objects in thread and fetch multiple return values from thier associated multiple std::future objects.

- 如果在调用future::get之前promise对象已经销毁，则将会抛出异常
- 如果需要一个线程返回多个值，则可以通过传递多个promise给执行线程

```cpp
#include <iostream>       // std::cout
#include <functional>     // std::ref
#include <thread>         // std::thread
#include <future>         // std::promise, std::future

void power(int n,std::promise<long long> &pro) {
    long long ret = 1;
    for(int i = 0; i < n;++i)
    {
        ret *= 2;
    }
    pro.set_value(ret); // 设置共享状态的值, 此处和主线程保持同步.
}

int main ()
{
    std::promise<long long> prom; // 生成一个 std::promise<int> 对象.
    std::future<long long> fut = prom.get_future(); // 和 future 关联.
    std::thread t(print_int,3, std::move(prom)); // 将 future 交给另外一个线程t.
    
    std::cout << "power:" << fut.get();

    return 0;
}
```

## package_task

package_task可以将一个可调用对象打包为可以异步执行的对象，它实际上是对promise的封装。打包后的对象可有直接调用，对该对象的调用将返回future，通过future可以拿到原始函数执行的结果。要说明的是，被封装后的对象直接执行的时候是同步的，要异步执行你就需要将他放入async或者thread中执行。

```cpp
#include <iostream>
#include <cmath>
#include <thread>
#include <future>
#include <functional>

// unique function to avoid disambiguating the std::pow overload set
int f(int x, int y) { 
    sleep(2);
    std::cout << "wake up" << std::endl;
    return std::pow(x,y); 
}
 
void task_lambda()
{
    std::packaged_task<int(int,int)> task([](int a, int b) {
        return std::pow(a, b); 
    });
    std::future<int> result = task.get_future();
 
    task(2, 9);
 
    std::cout << "task_lambda:\t" << result.get() << '\n';
}
 
void task_bind()
{
    std::packaged_task<int()> task(std::bind(f, 2, 11));
    std::future<int> result = task.get_future();
 
    task();
    std::cout << "waiting ??" << std::endl;
    std::cout << "task_bind:\t" << result.get() << '\n';
}
 
void task_thread()
{
    std::packaged_task<int(int,int)> task(f);
    std::future<int> result = task.get_future();
 
    std::thread task_td(std::move(task), 2, 10);
    task_td.join();
 
    std::cout << "task_thread:\t" << result.get() << '\n';
}

int main()
{
    task_lambda();
    task_bind();
    task_thread();
}
```


## 使用async异步获取线程结果

std::async是对std::package_task的进一步封装。它先将异步操作用std::packaged_task包装起来，然后将异步操作的结果放到std::promise中，并返回std::future。外面再通过future.get/wait来获取线程异步执行的结果。

- std::launch::async:待执行的函数将在另外的线程中执行——异步
- std::launch::deferred:待执行的函数将延迟执行——同步
- std::launch::async | std::launch::deferred:默认的方式，可能异步，也可能同步，取决于系统


std::async() does following things:
- It automatically creates a thread (Or picks from internal thread pool) and a promise object for us.
- Then passes the std::promise object to thread function and returns the associated std::future object.
- When our passed argument function exits then its value will be set in this promise object, so eventually return value will be available in std::future object

std::async()完成以下事情：
- 自动创建一个线程或者使用一个内部的线程池，并且创建一个promise对象
- 将promise对象传递给线程函数，并且返回关联的future对象
- 当我们传递的函数执行完毕的时候，设置promise的值，这样我们在future上调用获取结果函数得到指定的结果


```cpp
async(std::launch::async | std::launch::deferred, f, args...);

std::future<int> future = std::async(std::launch::async, [](){ 
    std::this_thread::sleep_for(std::chrono::seconds(3));
    return 8;  
}); 

std::cout << "waiting...\n";
std::future_status status;
do {
    status = future.wait_for(std::chrono::seconds(1));
    if (status == std::future_status::deferred) {
        std::cout << "deferred\n";
    } else if (status == std::future_status::timeout) {
        std::cout << "timeout\n";
    } else if (status == std::future_status::ready) {
        std::cout << "ready!\n";
    }
} while (status != std::future_status::ready); 

std::cout << "result is " << future.get() << '\n';
```

```cpp
#include <thread>
#include <future>
#include <iostream>

int main(){
    auto future = std::async(std::launch::async, [](){
        std::cout << "I'm a thread" << std::endl;
    });

    future.get();

    return 0;
}

/*Nothing really special here. std::async will execute the task that we give it (here a lambda) and return a std::future. Once you use the get() function on a future, it will wait until the result is available and return this result to you once it is. The get() function is then blocking. Since the lambda, is a void lambda, the returned future is of type std::future<void> and get() returns void as well. It is very important to know that you cannot call get several times on the same future. Once the result is consumed, you cannot consume it again! If you want to use the result several times, you need to store it yourself after you called get().

Let's see with something that returns a value and actually takes some time before returning it:*/

#include <thread>
#include <future>
#include <iostream>
#include <chrono>

int main(){
    auto future = std::async(std::launch::async, [](){
        std::this_thread::sleep_for(std::chrono::seconds(5));
        return 42;
    });

    // Do something else ?

    std::cout << future.get() << std::endl;

    return 0;
}
This time, the future will be of the time std::future<int> and thus get() will also return an int. std::async will again launch a task in an asynchronous way and future.get() will wait for the answer. What is interesting, is that you can do something else before the call to future.

But get() is not the only interesting function in std::future. You also have wait() which is almost the same as get() but does not consume the result. For instance, you can wait for several futures and then consume their result together. But, more interesting are the wait_for(duration) and wait_until(timepoint) functions. The first one wait for the result at most the given time and then returns and the second one wait for the result at most until the given time point. I think that wait_for is more useful in practices, so let's discuss it further. Finally, an interesting function is bool valid(). When you use get() on the future, it will consume the result, making valid() returns :code:`false. So, if you intend to check multiple times for a future, you should use valid() first.

One possible scenario would be if you have several asynchronous tasks, which is a common scenario. You can imagine that you want to process the results as fast as possible, so you want to ask the futures for their result several times. If no result is available, maybe you want to do something else. Here is a possible implementation:

#include <thread>
#include <future>
#include <iostream>
#include <chrono>

int main(){
    auto f1 = std::async(std::launch::async, [](){
        std::this_thread::sleep_for(std::chrono::seconds(9));
        return 42;
    });

    auto f2 = std::async(std::launch::async, [](){
        std::this_thread::sleep_for(std::chrono::seconds(3));
        return 13;
    });

    auto f3 = std::async(std::launch::async, [](){
        std::this_thread::sleep_for(std::chrono::seconds(6));
        return 666;
    });

    auto timeout = std::chrono::milliseconds(10);

    while(f1.valid() || f2.valid() || f3.valid()){
        if(f1.valid() && f1.wait_for(timeout) == std::future_status::ready){
            std::cout << "Task1 is done! " << f1.get() << std::endl;
        }

        if(f2.valid() && f2.wait_for(timeout) == std::future_status::ready){
            std::cout << "Task2 is done! " << f2.get() << std::endl;
        }

        if(f3.valid() && f3.wait_for(timeout) == std::future_status::ready){
            std::cout << "Task3 is done! " << f3.get() << std::endl;
        }

        std::cout << "I'm doing my own work!" << std::endl;
        std::this_thread::sleep_for(std::chrono::seconds(1));
        std::cout << "I'm done with my own work!" << std::endl;
    }

    std::cout << "Everything is done, let's go back to the tutorial" << std::endl;

    return 0;
}
```

```cpp
#include <thread>
#include <future>
#include <iostream>
#include <chrono>
#include <vector>

int main(){
    std::vector<std::future<size_t>> futures;

    for (size_t i = 0; i < 10; ++i) {
        futures.emplace_back(std::async(std::launch::async, [](size_t param){
            std::this_thread::sleep_for(std::chrono::seconds(param));
            return param;
        }, i));
    }

    std::cout << "Start querying" << std::endl;

    for (auto &future : futures) {
      std::cout << future.get() << std::endl;
    }

    return 0;
}
```