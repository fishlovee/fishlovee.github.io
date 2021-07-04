---
layout: post
title: 16-使用future、packaged_task实现的任务线程池
categories: cpp_concurrency
description: C++并发编程简介
keywords: c++, 并发编程,线程池，异步,std::future,std::packaged_task
---

- [使用std::future和std::packaged_task实现的异步threadpool](https://gitee.com/wikall/sylar-clone/blob/master/src/base/threadpool.h)
- [使用C++封装pthread](https://gitee.com/wikall/sylar-clone/blob/master/src/base/pthread.h)


在c++11中实现一个线程池是很简单的。但是要实现一个可以进行异步等待获取结果的线程池则不是那么简单，在实现过程中除了需要使用C++11的一些新特性，还需要注意一下细微的问题：

- 线程池的完美退出
- 线程池的调度：任务推送、任务获取、工作线程唤醒
- 多线程变量生命周期管理：这里的多线程生命周期管理指的是被push到线程池中的其他线程中的变量的生命周期，比如被lambda捕获的变量，或者被bind绑定的对象
- 将普通的任务封装成packaged_task，并返回future对象，方便异步获取任务执行情况
- 线程池状态的判定：任务数量，线程数量，空闲线程数，总的线程数等


**线程池代码**


```cpp
#pragma once
#include <thread>
#include <functional>
#include <vector>
#include <random>
#include <atomic>
#include <future>
#include "base/thread.h"
#include "base/pthread.h"
#include "base/taskpool.h"
#include "base/mutex.h"
#include "log/log.h"

namespace fisher
{
    inline unsigned int hardware_concurrency(const unsigned int max_concurrency)
    {
        auto ret = std::thread::hardware_concurrency();
        if (ret == 0)
        {
            return max_concurrency;
        }
        return std::min(ret, max_concurrency);
    }

    // 使用 Pthread 代替 std::thread。
    class thread_pool
    {
        using thread_type = fisher::Pthread;

    public:
        explicit thread_pool(const int32_t thread_count, const std::string &pool_name = "Unknown ") : m_idle_count(thread_count), m_stop(false)
        {
            G_LOGD << "to create threads...";
            for (int i = 0; i < thread_count; ++i)
            {
                //std::shared_ptr<thread_type> thread(new Pthread(std::bind(&thread_pool::thread_func, this), "thread_pool " + std::to_string(i)));
                m_threads.emplace_back(std::make_shared<thread_type>(std::bind(&thread_pool::thread_func, this), pool_name + std::to_string(i)));
            }

            G_LOGD << "to start threads...";
            for (auto &thread : m_threads)
            {
                thread->start_thread();// fx--> 不应该在构造函数中启动线程 2021-06-24
            }
        }

        ~thread_pool()
        {
            // 析构，设置结束标记位，并等待线程结束
            m_stop.store(true);
            m_cv.notify_all();
            for (auto &t : m_threads)
            {
                //G_LOGD << "to join thread:" << t->thread_info();
                if (t->joinable())
                {
                    t->join();
                }
            }
        }

        auto commit(std::function<void()> f) -> std::future<decltype(f())>
        {
            using ret_type = decltype(f());
            auto task = std::make_shared<std::packaged_task<ret_type()>>(std::bind(f));
            //auto task = std::make_shared<std::packaged_task<void()>>(std::packaged_task<void()>(f));
            auto future = task->get_future();
            {
                std::lock_guard<std::mutex> lock(m_mutex);
                m_task_pool.emplace_back([=]()
                                         { (*task)(); });
            }

            m_cv.notify_one();
            return future;
        }

        std::vector<std::future<void>> commit(std::vector<std::function<void()>> tasks)
        {
            std::vector<std::future<void>> futures;
            {
                std::lock_guard<std::mutex> lock(m_mutex);
                for (auto &f : tasks)
                {
                    using ret_type = decltype(f());
                    auto task = std::make_shared<std::packaged_task<ret_type()>>(std::bind(f));
                    auto future = task->get_future();
                    {
                        m_task_pool.emplace_back([=]()
                                                 { (*task)(); });
                    }

                    m_cv.notify_one();
                }
            }
            return futures;
        }

        int32_t idle_count() const
        {
            return m_idle_count;
        }

        int32_t task_count() const
        {
            return m_task_pool.size();
        }

    private:
        void thread_func()
        {
            while (!this->m_stop.load())
            {
                std::function<void()> task;
                {
                    std::unique_lock<std::mutex> ulock(this->m_mutex);
                    this->m_cv.wait(ulock, [this]()
                                    {
                                        // G_LOGD << "this->m_stop=" << this->m_stop;
                                        // G_LOGD << "this->m_task_pool.empty()=" << this->m_task_pool.empty();
                                        return this->m_stop.load() || !this->m_task_pool.empty();
                                    });

                    //G_LOGD << "thread pool wake up,stoped = " << this->m_stop.load() << ",task size = " << this->task_count();
                    if (this->m_stop.load() && this->m_task_pool.empty())
                    {
                        break;
                    }

                    task = std::move(this->m_task_pool.front());
                    this->m_task_pool.pop_front();
                }

                this->m_idle_count--;
                task();
                this->m_idle_count++;
            }

            G_LOGD << "thread exit..." << Pthread::Thread_Info();
            this->m_idle_count--;
        }

    private:
        std::atomic<int32_t> m_idle_count;
        std::atomic<bool> m_stop;
        std::deque<std::function<void()>> m_task_pool;
        std::vector<std::shared_ptr<thread_type>> m_threads;
        std::mutex m_mutex;
        std::condition_variable m_cv;
    };
} // namespace fisher

```