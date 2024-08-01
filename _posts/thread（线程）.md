# thread（线程）

## thread状态

![C++thread](./image/C++thread.png)

thread是对线程的创建以及管理，线程一旦被创建，线程就开始运行，对于管理者，这时有三个选项，放任不管（反正线程已经开始运行），join（加入到现有进程，这不就是没有多线程吗），detach（分离，后台运行）。

- `放任不管`：那么如果创建的thread对象是局部变量，那么可能会自动析构，析构函数`if (joinable()) {std::terminate();}`会发现你是放任不管的，就抛出异常并终止线程。
- `join`：加入到现在的主进程，那么主进程就会阻塞等待这个线程完成，既然主线程已经需要等待这个线程完成了，那么主线程也无法对这个线程进行控制了，所以`joinable=false`无法回头。
- `detach`：分离线程到后台运行，对应于上面的放任不管，这个就相当于thread对象可以析构了，所以现在的thread对象已经失去对这个线程的管理权了，放任线程自流。

## 构造函数

| 函数类型               | 参数                                                         |
| ---------------------- | ------------------------------------------------------------ |
| 默认构造函数           | thread() noexcept;                                           |
| 初始化构造函数         | template <class Fn, class... Args><br/>explicit thread(Fn&& fn, Args&&... args); |
| 拷贝构造函数 [deleted] | thread(const thread&) = delete;                              |
| move 构造函数          | thread(thread&& x) noexcept;                                 |

- 默认构造函数，创建一个空的 `std::thread` 执行对象。
- 初始化构造函数，创建一个 `std::thread` 对象，该 `std::thread` 对象可被 `joinable`，新产生的线程会调用 `fn` 函数，该函数的参数由 `args` 给出。
- 拷贝构造函数(被禁用)，意味着 `std::thread` 对象不可拷贝构造。
- `move` 构造函数，调用成功之后 `x` 不代表任何 `std::thread` 执行对象，即`x`转移到`*this`了。

```C++
#include <iostream>
#include <thread>
void f1(int i) {
    while (true) {
        std::cout << i++ << std::endl;
        std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    }

}
void f2() {
    while (true) {
        std::cout << "f2" << std::endl;
        std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    }
}
int main() {
    std::cout << "Hello, World!" << std::endl;
    
    std::thread t1(f1, 1);
    std::thread t2(f2);
    std::thread t3{f1, 1000};
    std::thread t4{f2};
    std::thread t5(std::move(t2));

    while (true) {}
    return 0;
}
```

## 赋值

| 函数类型               | 参数                                       |
| ---------------------- | ------------------------------------------ |
| move 赋值操作          | thread& operator=(thread&& rhs) noexcept;  |
| 拷贝赋值操作 [deleted] | thread& operator=(const thread&) = delete; |

- `move`赋值操作：thread是线程的创建和管理，也管理不了什么内容，一个线程同时只能有一个thread对象进行管理，使用move赋值，删除拷贝赋值。

```C++
#include <iostream>
#include <thread>
void f1(int i) {
    while (true) {
        std::cout << i++ << std::endl;
        std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    }

}
int main() {
    std::cout << "Hello, World!" << std::endl;
    std::thread t1(f1, 1);
    std::thread t5;
    t5 = std::move(t1);
    if (t5.joinable())t5.join();
    return 0;
}
```

## 成员函数

| 函数                                                         | 功能                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| std::thread::id get_id()                                     | 返回线程ID                                                   |
| bool joinable()                                              | 检查线程是否能被join，默认构造函数生成的thread对象不能被join，线程任务执行完毕之后还没被join，是可以被join，因为线程还是被认为是活动的 |
| detach()                                                     | 分离线程，执行之后，joinable=false，get_id()=std::thread::id()，即为默认ID，因为现在这个thread对象已经失去管理权了，线程被放任自流 |
| join()                                                       | 加入线程，执行之后，joinable=false，线程被加入到主线程执行，主线程被阻塞，主线程是一个相对概念 |
| swap(thread)<br>std::swap(thread,thread)                     | 成员函数swap和标准库的swap都能完成两个线程对象的交换功能，因为这不违反同一时间只能有一个线程类管理同一个线程的思想，使用引用完成了交换。 |
| native_handle()                                              | 返回原生系统的线程句柄                                       |
| hardware_concurrency()<br>std::thread::hardware_concurrency() | 返回unsigned int类型，表示当前硬件系统最多支持的线程数量     |

```C++
#include <iostream>
#include <thread>
void f1(int i) {
    while (true) {
        std::cout << i++ << std::endl;
        std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    }

}
int main() {
    std::cout << "Hello, World!" << std::endl;
    std::thread t1(f1, 1);
    std::thread t5;
    t5.swap(t1);
    t1.swap(t5);
    std::swap(t1, t5);
    std::cout << "原生系统句柄：" << t5.native_handle() << std::endl;
    std::cout << "硬件线程数量：" << std::thread::hardware_concurrency() << std::endl;
    std::cout << "硬件线程数量：" << t5.hardware_concurrency() << std::endl;
    if (t5.joinable())t5.join();
    return 0;
}
//结果：
/*
Hello, World!
原生系统句柄：00000000000000B0
硬件线程数量：8
硬件线程数量：8
1
2
3
...
*/
```

## 参数传递

thread对象的构建是使用函数指针，并传递对应的参数，但是引用类型的参数传递就需要多加一层判断。

```C++
//普通函数
```

| 函数类型            | 线程构建                             |
| ------------------- | ------------------------------------ |
| void foo()          | std::thread(foo)<br>std::thread{foo} |
| void foo(int)       | std::thread(foo,int)                 |
| void foo(int,int)   | std::thread(foo,int,int)             |
| void foo(int &,int) | std::thread(foo,std::ref(int),int)   |

```C++
//类函数
class coo {
public:
    int i = 0;
    void run(int &j, int k) {
        std::cout << i++ << "-" << j++ << "-" << k++ << std::endl;
    }
};
int main() {
    int j=10,k=100;
    coo co;
    std::thread t1(&coo::run,&co,std::ref(j),k);
    t1.join();
    std::cout << co.i << "-" << j << "-" << k << std::endl;
    return 0;
}
```

```C++
//Lambda表达式
std::thread t1([](int &j, int k) {
        std::cout << j++ << "-" << k++ << std::endl;
    }, std::ref(j), k);
std::thread t1([j, &k] { return f1(j, k); });
```

```C++
//bind函数
//C++14 
#include<functional>
std::thread t1(std::bind(f1, j, std::ref(k)));
```

## std::this_thread中的相关辅助函数

```C++
//得到当前线程ID
std::thread::id this_id = std::this_thread::get_id();
//放弃获得的线程时间
std::this_thread::yield();
//休眠一段时间
std::this_thread::sleep_for(std::chrono::milliseconds(1000));
//休眠到指定时间点
std::this_thread::sleep_until(std::chrono::steady_clock::now()+std::chrono::milliseconds(1000));
```

## Boost asio库线程池

```C++
#include <boost/asio/thread_pool.hpp>
/// A simple fixed-size thread pool.
/**
 * For example:
 * @code void my_task()
 * {
 *   ...
 * }
 *
 * // 创建具有四个线程的线程池
 * boost::asio::thread_pool pool(4);
 *
 * // 发送函数到线程池
 * boost::asio::post(pool, my_task);
 *
 * // 发送lambda对象到线程池
 * boost::asio::post(pool,
 *     []()
 *     {
 *       ...
 *     });
 *
 * // 等待池中的所有任务完成
 * pool.join(); @endcode
 */
```

## 实现线程池

```C++
#pragma once

#include <thread>
#include <vector>
#include <queue>
#include <memory>
#include <functional>
#include <mutex>
#include <condition_variable>
#include <future>
#include <atomic>
#include <iostream>

namespace thread_pool {
    class thread_pool {
    public:
        using Task = std::function<void()>;

    private:
        std::vector<std::shared_ptr<std::thread>> _pool;
        std::queue<Task> _tasks;
        std::mutex _mutex;
        std::condition_variable _condition;
        std::atomic<bool> _stop;

    public:
        explicit thread_pool(const unsigned int &size) : _stop(false) {
            for (unsigned int i = 0; i < size; ++i) {
                _pool.emplace_back(std::make_shared<std::thread>(&thread_pool::worker, this));
            }
        }

        thread_pool(const thread_pool &) = delete;

        thread_pool &operator=(const thread_pool &) = delete;

        ~thread_pool() {
            {
                std::unique_lock<std::mutex> lock(_mutex);
                _stop = true;
            }
            _condition.notify_all();
            for (auto &thread: _pool) {
                if (thread->joinable()) {
                    thread->join();
                }
            }
        }

        template<class F, class... Args>
        auto post(F &&f, Args &&... args) -> std::future<typename std::result_of<F(Args...)>::type> {
            using ReturnType = typename std::result_of<F(Args...)>::type;
            auto task = std::make_shared<std::packaged_task<ReturnType()>>(
                    std::bind(std::forward<F>(f), std::forward<Args>(args)...)
            );
            std::future<ReturnType> result = task->get_future();
            {
                std::unique_lock<std::mutex> lock(_mutex);
                if (_stop) {
                    throw std::runtime_error("Thread pool has stopped");
                }
                _tasks.emplace([task]() { (*task)(); });
            }
            _condition.notify_one();
            return result;
        }

    private:
        void worker() {
            while (true) {
                Task task;
                {
                    std::unique_lock<std::mutex> lock(_mutex);
                    _condition.wait(lock, [this]() { return _stop || !_tasks.empty(); });
                    if (_stop && _tasks.empty()) {
                        return;
                    }
                    task = std::move(_tasks.front());
                    _tasks.pop();
                }
                task();
            }
        }
    };
}
```
