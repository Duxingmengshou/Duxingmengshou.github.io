---
title: 'mutex（互斥锁）与condition_variable（条件变量）'
date: 2023-12-14
permalink: /posts/2023/12/mutex（互斥锁）与condition_variable（条件变量）/
tags:
  - C++
---
# mutex（互斥锁）与condition_variable（条件变量）

## 无法避免的同步问题

如果多线程编程中各个线程不需要数据的交换，那么多线程的目的是什么呢，为何不直接分割为多个程序让操作系统进行管理呢。所以多线程编程中线程的数据交换，同步是无法避免的，而直接使用多线程对数据进行没有保护的使用，结果也是无法预知的。

```C++
//一个无法预知结果的加法运算
#include <mutex>
#include <thread>
#include <iostream>

int i = 0;

void f1() {
    for (int j = 0; j < 100000; j++) {
        i++;
    }
}
int main() {
    std::thread t1(f1);
    std::thread t2(f1);
    std::thread t3(f1);
    std::thread t4(f1);
    std::thread t5(f1);

    while (true) {
        std::this_thread::sleep_for(std::chrono::milliseconds(1000));
        std::cout << i << std::endl;
    }
    return 0;
}
//这段程序运行结果应该在[100000,500000]之间，很显然，两个极端都是很难达到的，我的运行结果在[150000,200000]中徘徊。
```

因为对共享数据没有保护的使用，所以出现了无法预知的结果。引入锁的概念保护共享区数据，使得同时对数据进行访问的线程只有一个。

```C++
#include <mutex>
#include <thread>
#include <iostream>
int i = 0;
std::mutex mx;
void f1() {
    for (int j = 0; j < 100000; j++) {
        mx.lock();
        i++;
        mx.unlock();
    }
}
int main() {
    std::thread t1(f1);
    std::thread t2(f1);
    std::thread t3(f1);
    std::thread t4(f1);
    std::thread t5(f1);

    while (true) {
        std::this_thread::sleep_for(std::chrono::milliseconds(1000));
        std::cout << i << std::endl;
    }
    return 0;
}
//结果是标准的500000
```

## 互斥量

| 互斥量类型                         | 功能                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| `std::mutex`                       | 最基本的互斥锁                                               |
| `std::timed_mutex`                 | 支持定时操作的锁                                             |
| `std::recursive_mutex`             | 允许同一线程多次获取同一锁                                   |
| `std::recursive_timed_mutex`       | 结合了定时锁和递归锁的特性                                   |
| `std::shared_mutex`（C++17）       | 允许多个线程读取，但是只允许一个线程写入，但是读和写都是独立的，读时不能写，写时不能读 |
| `std::shared_timed_mutex`（C++14） | 结合了`std::shared_mutex`和`std::timed_mutex`                |

```C++
#include <mutex>
#include <iostream>
#include <thread>

int i = 0;
std::mutex mx, mx1;

//使用lock成员函数的锁，如果获取不到会一直阻塞到获取到为止，保证流程一定正确执行下去
void f1() {
    for (int j = 0; j < 100000; j++) {
        mx.lock();
        i++;
        mx.unlock();
    }
}

//使用std::lock_guard<>的范围锁，lock_guard对象创建上锁，析构解锁
void f2() {
    for (int j = 0; j < 100000; j++) {
        std::lock_guard<std::mutex> guardLock(mx);
        i++;
    }
}

//使用std::unique_lock<>的范围锁，除了具有lock_guard的功能之外，还能中途解锁上锁
void f3() {
    for (int j = 0; j < 100000; j++) {
        std::unique_lock<std::mutex> uniqueLock(mx);
        i++;
        uniqueLock.unlock();
        std::this_thread::sleep_for(std::chrono::milliseconds(1));
        uniqueLock.lock();
    }
}

//使用std::scoped_lock<>的范围锁（C++17），同时使用一种方法来避免死锁
void f4() {
    for (int j = 0; j < 100000; j++) {
        std::scoped_lock<std::mutex> scopedLock(mx);
        //std::scoped_lock scopedLock1(mx, mx1);//同时给多个上锁，同样在C++17中，简单模板类型可以不用<>指定类型
        i++;
    }
}

//使用try_lock立即返回上锁结果，lock会阻塞等待到能上锁为止
void f5() {
    for (int j = 0; j < 100000; j++) {
        if (mx.try_lock()) {
            i++;
            mx.unlock();
        } else {
            std::cout << "try_lock失败，使用lock阻塞上锁。" << std::endl;
            mx.lock();
            i++;
            mx.unlock();
        }
    }
}

int main() {

    std::thread t1(f1);
    std::thread t2(f2);
    std::thread t3(f3);
    std::thread t4(f4);
    std::thread t5(f5);

    while (true) {
        std::cout << i << std::endl;
        std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    }

    return 0;
}
/*
结果，因为std::unique_lock<std::mutex>休眠了1ms，所以不会立即得到500000
0
try_lock失败，使用lock阻塞上锁。
...
try_lock失败，使用lock阻塞上锁。
400604
...
412512
...
499778
500000
*/
```

```C++
#include <mutex>
#include <iostream>
#include <thread>

int i = 0;
std::timed_mutex mx;

//使用try_lock_for成员函数的锁，即在1000ms的时间内持续去尝试获取锁，不像mutex的lock会一直阻塞等待，也不像它的try_lock立即返回
void f1() {
    for (int j = 0; j < 1000; j++) {
        if (mx.try_lock_for(std::chrono::milliseconds(1))) {
            i++;
            std::this_thread::sleep_for(std::chrono::milliseconds(2));
            mx.unlock();
        } else {
            std::cout << "f1 获取锁超时。" << std::endl;
            j--;//本次操作失败
        }
    }
}

//使用std::lock_guard<>的范围锁，lock_guard对象创建上锁，析构解锁
void f2() {
    for (int j = 0; j < 1000; j++) {
        if (mx.try_lock_for(std::chrono::milliseconds(2))) {
            std::lock_guard<std::timed_mutex> guard(mx, std::adopt_lock);
            i++;
        } else {
            std::cout << "f2 获取锁超时。" << std::endl;
            j--;
        }

    }
}

//使用try_lock_until，即直到某个时间点
void f3() {
    for (int j = 0; j < 1000; j++) {
        if (mx.try_lock_until(std::chrono::steady_clock::now() + std::chrono::milliseconds(3))) {
            std::lock_guard<std::timed_mutex> guard(mx, std::adopt_lock);
            i++;
        } else {
            std::cout << "f3 获取锁超时。" << std::endl;
            j--;
        }
    }
}

int main() {

    std::thread t2(f2);
    std::thread t1(f1);
    std::thread t3(f3);

    while (true) {
        std::cout << i << std::endl;
        std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    }
    return 0;
}
/*
结果：
0
f3 获取锁超时。
f2 获取锁超时。
f2 获取锁超时。
f3 获取锁超时。
...
f2 获取锁超时。
3000
3000
*/
```

```C++
#include <mutex>
#include <iostream>
#include <thread>

int i = 0;
std::recursive_mutex mx;

//使用lock_guard控制范围锁，递归互斥量支持同一线程同一互斥量多次上锁
void f1() {
    if (i == 1000) {
        return;
    }
    std::lock_guard lock(mx);
    i++;
    f1();
}

int main() {
    std::thread t1(f1);
    while (true) {
        std::cout << i << std::endl;
        std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    }
    return 0;
}
/*
结果：
0
1000
1000
...
*/
```

```C++
#include <iostream>
#include <thread>
#include <mutex>

std::recursive_timed_mutex rtmtx;

//递归等待互斥锁的使用
void timed_recursive_function(int depth) {
    if (depth > 0) {
        if (rtmtx.try_lock_for(std::chrono::seconds(1))) {
            std::cout << "Depth: " << depth << std::endl;
            timed_recursive_function(depth - 1);
            rtmtx.unlock();
        } else {
            std::cout << "Failed to lock within 1 second at depth: " << depth << std::endl;
        }
    }
}

int main() {
    std::thread t1(timed_recursive_function, 5);
    t1.join();
    return 0;
}
/*
结果：
Depth: 5
Depth: 4
Depth: 3
Depth: 2
Depth: 1
*/
```

```C++
#include <iostream>
#include <thread>
#include <shared_mutex>
//使用shared_mutex保证多个读线程以及避免了不上锁导致读时写问题
std::shared_mutex smtx;
int i = 0;

void reader(int id) {
    while (true) {
        std::shared_lock<std::shared_mutex> lock(smtx);
        std::cout << "Reader " << id << " is reading------------" << i << std::endl;
        lock.unlock();
        std::this_thread::sleep_for(std::chrono::milliseconds(50));
    }
}

void writer(int id) {
    while (true) {
        std::unique_lock<std::shared_mutex> lock(smtx);
        std::cout << "Writer " << id << " is writing------------w" << std::endl;
        i++;
        lock.unlock();
        std::this_thread::sleep_for(std::chrono::milliseconds (100));
    }
}

int main() {

    std::thread t1(reader, 1);
    std::thread t2(reader, 2);
    std::thread t3(writer, 1);

    while (true) {
        std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    }
    return 0;
}
/*
结果：每两个w之间的数据都是一致的
Writer 1 is writing------------w
Reader 2 is reading------------1
Reader 1 is reading------------1
Reader 2 is reading------------1
Reader 1 is reading------------1
Writer 1 is writing------------w
Reader 2 is reading------------2
Reader 1 is reading------------2
Reader 2 is reading------------2
Reader 1 is reading------------2
Writer 1 is writing------------w
Reader 2 is reading------------3
Reader 1 is reading------------3
Reader 2 is reading------------3
Reader 1 is reading------------3
Writer 1 is writing------------w
*/
/*
为什么要使用共享锁？为什么要使用锁？
读者写者问题中，读于读者，有三种情况：
1、读者不加锁，出现读时写
2、读者加互斥锁，只能一个读者
3、读者加共享锁，多个读者，写者也不能写
读者写者PV伪代码：
int count = 0;
信号量 busy = 1; // “读文件”和“写文件”的互斥锁
信号量 mutex = 1; // 变量 count 的互斥锁
Reader(){ // 读者进程
    while(1){
        P(mutex);
        count++; 
        if (count == 1){ 
            P(busy);
        }
        V(mutex);
        
        读文件; 
        
        P(mutex);
        count--; 
        if (count == 0){ 
            V(busy);
        }
        V(mutex);
    }
}
Writer(){ // 写者进程
    while(1){
        P(busy);
        写文件;
        V(busy);
    }
}
这里巧妙使用一个count变量以及它的互斥锁来解决多个读者的问题，而shared_mutex相当于封装了一个这样的功能。
*/
//上面的所有例子使用的i++都有点类似原子操作，下面模拟读时写
#include <iostream>
#include <thread>
#include <shared_mutex>

std::mutex mx;
int i = 0, j = 0;

void reader(int id) {
    while (true) {
        std::cout << "Reader " << id << " is reading------------" << i << "-";
        std::this_thread::sleep_for(std::chrono::milliseconds(50));
        std::cout << j << std::endl;
    }
}

void writer(int id) {
    while (true) {
        std::unique_lock<std::mutex> lock(mx);
        i++, j++;
        lock.unlock();
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
}

int main() {
    std::thread t1(reader, 1);
    std::thread t3(writer, 1);
    while (true) {
        std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    }
    return 0;
}
/*
结果：出现数据不一致问题
Reader 1 is reading------------1-1
Reader 1 is reading------------1-2
Reader 1 is reading------------2-2
Reader 1 is reading------------2-3
Reader 1 is reading------------3-3
Reader 1 is reading------------3-4
Reader 1 is reading------------4-4
Reader 1 is reading------------4-5
理论上讲，当然可以严格控制每一个读写都使用唯一锁，但是这样很显然读取的性能会有所消耗。
*/
```

```C++
#include <iostream>
#include <thread>
#include <shared_mutex>
//是对共享锁和等待锁的合并，不再赘述
std::shared_timed_mutex stm;

void try_shared_lock_for_example(int id) {
    if (stm.try_lock_shared_for(std::chrono::seconds(1))) {
        std::cout << "Reader " << id << " locked for shared access" << std::endl;
        std::this_thread::sleep_for(std::chrono::seconds(1));
        stm.unlock_shared();
    } else {
        std::cout << "Reader " << id << " failed to lock for shared access within 1 second" << std::endl;
    }
}

void try_lock_for_example(int id) {
    if (stm.try_lock_for(std::chrono::seconds(1))) {
        std::cout << "Writer " << id << " locked for exclusive access" << std::endl;
        std::this_thread::sleep_for(std::chrono::seconds(1));
        stm.unlock();
    } else {
        std::cout << "Writer " << id << " failed to lock for exclusive access within 1 second" << std::endl;
    }
}

int main() {
    std::thread t1(try_shared_lock_for_example, 1);
    std::thread t2(try_shared_lock_for_example, 2);
    std::thread t3(try_lock_for_example, 1);

    t1.join();
    t2.join();
    t3.join();

    return 0;
}
/*
结果：
Reader 1 locked for shared access
Writer 1 locked for exclusive access
Reader 2 failed to lock for shared access within 1 second
或
Reader 1 locked for shared access
Writer 1 failed to lock for exclusive access within 1 second
Reader 2 locked for shared access
...一切皆有可能
*/
```

```C++
//实现简陋的shared_mutex
#include <iostream>
namespace shared_mutex {
    class shared_mutex {
    private:
        unsigned int _count = 0;
        bool _mutex = false, _mutex_count = false;
    public:
        shared_mutex() {}

        void lock() {
            while (_mutex) {}
            _mutex = true;
        }

        void unlock() {
            if (_mutex) {
                _mutex = false;
            } else {
                std::cerr << "unlock unlock" << std::endl;
            }
        }

        void shared_lock() {
            while (_mutex_count) {}
            _mutex_count = true;
            _count++;
            if (_count == 1)
                lock();
            _mutex_count = false;
        }

        void shared_unlock() {
            while (_mutex_count) {}
            _mutex_count = true;
            _count--;
            if (_count == 0)
                unlock();
            _mutex_count = false;
        }
    };
}


shared_mutex::shared_mutex sm;
int i = 0;
#include <thread>
void reader(int id) {
    while (true) {
        sm.shared_lock();
        std::cout << "Reader " << id << " is reading------------" << i << std::endl;
        sm.shared_unlock();
        std::this_thread::sleep_for(std::chrono::milliseconds(50));
    }
}
void writer(int id) {
    while (true) {
        sm.lock();
        std::cout << "Writer " << id << " is writing------------w" << std::endl;
        i++;
        sm.unlock();
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
}
int main() {
    std::thread t1(reader, 1);
    std::thread t2(reader, 2);
    std::thread t3(writer, 1);
    while (true) {
        std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    }
    return 0;
}
```

## 条件变量

`std::condition_variable` 需要与 `std::unique_lock<std::mutex>` 一起使用。

`std::condition_variable_any`可以与任何锁一起使用。

| 成员函数                                                     | 功能                                       |
| ------------------------------------------------------------ | ------------------------------------------ |
| `void wait(std::unique_lock<std::mutex>& lock, Predicate pred );` | 让当前线程阻塞，直到被通知或者满足某个条件 |
| `void notify_one()`                                          | 唤醒一个等待的线程                         |
| `void notify_all()`                                          | 唤醒所有等待的线程                         |

```C++
//使用互斥量、条件变量完成的生产者消费者模型
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>

std::mutex mtx;
std::condition_variable cv;
std::queue<int> dataQueue;
bool done = false;

void producer() {
    for (int i = 0; i < 10; ++i) {
        {
            std::lock_guard<std::mutex> lock(mtx);
            dataQueue.push(i);
            std::cout << "Produced: " << i << std::endl;
        }
        cv.notify_one(); // 通知消费者
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
    {
        std::lock_guard<std::mutex> lock(mtx);
        done = true;
    }
    cv.notify_all(); // 通知所有消费者
}

void consumer() {
    while (true) {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, []{ return !dataQueue.empty() || done; });

        while (!dataQueue.empty()) {
            int value = dataQueue.front();
            dataQueue.pop();
            std::cout << "Consumed: " << value << std::endl;
        }

        if (done) break;
    }
}

int main() {
    std::thread prodThread(producer);
    std::thread consThread(consumer);

    consThread.join();
    prodThread.detach();

    return 0;
}
/*
结果：
Produced: 0
Consumed: 0
Produced: 1
Consumed: 1
Produced: 2
Consumed: 2
Produced: 3
Consumed: 3
Produced: 4
Consumed: 4
Produced: 5
Consumed: 5
Produced: 6
Consumed: 6
Produced: 7
Consumed: 7
Produced: 8
Consumed: 8
Produced: 9
Consumed: 9
*/
```

```C++
//条件变量核心是解决忙等，采用通知的策略，在上面自己实现的shared_mutex中（while (_mutex) {}），等待解锁的过程是忙等的，这样会消耗计算机资源。
//解决忙等，不是进行回调，进行通知，不代表一定会让出数据访问权限。

//1、不是进行回调，不一定让出数据访问权限
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>

std::mutex mx;
std::condition_variable cv;
bool ready = false;  // 用于通知 f2 准备好

void f1() {
    while (true) {
        {
            std::unique_lock<std::mutex> lock(mx);
            std::cout << "f1" << std::endl;
            std::this_thread::sleep_for(std::chrono::milliseconds(30));
            ready = true;  // 设置条件为真
        }
        cv.notify_all();
// std::this_thread::sleep_for(std::chrono::nanoseconds(1000));//等待一个时间，让出控制权
    }
}

void f2() {
    while (true) {
        std::unique_lock<std::mutex> lock(mx);
        cv.wait(lock, []() { return ready; });  // 等待条件为真
        std::cout << "f2" << std::endl;
        ready = false;  // 重置条件
    }
}

int main() {
    std::thread t1(f1);
    std::thread t2(f2);

    while (true) {
        std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    }
    return 0;
}
/*
加上注释，结果：
f1
f1
f1
f1
f1
f1
f1
...（条件变量只是通知其它线程应该注意了，并不代表其他线程一定能得到锁，锁的得到是随机的，抢占的，很显然早先已经获得了锁的线程如果不进行等待，获得锁的速度比其他线程快）
不加注释，结果：
f1
f2
f1
f2
f1
f2
f1
...（交替进行，说明等待时间弥补了被通知线程的劣势，被通知线程获得了锁，在我的电脑上尝试了几个值，1000ns以下弥补不了这个劣势，很显然，这是与CPU有关的）
*/

```

```C++
//重写读者写者问题，经过上诉描述，互斥量、条件变量、锁的概念已经有了清晰认识，返璞归真，重新模拟读者写者问题
#include <iostream>
#include <thread>
#include <shared_mutex>
#include <condition_variable>

class WRP {
private:
    std::shared_mutex _shared_mutex;
    unsigned int _i = 0;
    std::condition_variable_any cv;
public:
    void Reader(const int &id) {
        while (true) {
            std::shared_lock lock(_shared_mutex);
            cv.wait(lock);
            std::cout << "Reader " << id << " ------" << _i << std::endl;
            lock.unlock();
        }
    }

    void Writer() {
        while (true) {
            std::unique_lock lock(_shared_mutex);
            _i++;
            std::cout << "Writer   ------w" << std::endl;
            lock.unlock();
            cv.notify_all();
            std::this_thread::sleep_for(std::chrono::milliseconds(1));
        }
    }
};


int main() {
    WRP wrp;
    std::thread t1(&WRP::Reader, &wrp, 1);
    std::thread t2(&WRP::Reader, &wrp, 2);
    std::thread t3(&WRP::Writer, &wrp);

    while (true) {
        std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    }

    return 0;
}
/*
结果：
Writer   ------w
Reader 1 ------1
Writer   ------w
Reader 1 ------2
Reader 2 ------2
Writer   ------w
Reader 2 ------3
Reader 1 ------3
...
为什么第一次读只有Reader1呢，很显然，因为t1先被构建，不过无法解释Writer优先于Reader，哈哈哈哈，这是因为cv.wait(lock)的阻塞是不管lock是否被获取的，即使lock已经被当前线程获取了，它还是会进入阻塞，直到等到通知的到来。（得到锁了还要等待，这不是死锁了吗🤪）
template <class _Lock>
void wait(_Lock& _Lck) noexcept { // wait for signal
    const shared_ptr<mutex> _Ptr = _Myptr; // for immunity to *this destruction
    unique_lock<mutex> _Guard{*_Ptr};
    _Unlock_guard<_Lock> _Unlock_outer{_Lck};--------------- //它解锁了
    _Cnd_wait(_Mycnd(), _Ptr->_Mymtx());                   |
    _Guard.unlock();                                       |
} // relock _Lck                                           |
                                                           |
explicit _Unlock_guard(_Lock& _Mtx_) : _Mtx(_Mtx_) {-------|         
    _Mtx.unlock();
}
*/
```