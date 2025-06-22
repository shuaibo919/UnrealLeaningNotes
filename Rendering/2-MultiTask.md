## C++多线程
### 1.C++多线程关键字
thread_local: 线程局部变量，每个线程都有自己独立的变量副本，线程之间互不干扰。

atomic: 原子操作，保证操作的原子性，防止多线程竞争。

volatile: 禁止编译器优化，保证变量的可见性。

### 2.多线程使用

基础使用
```cpp
#include <iostream>
#include <thread>
#include <chrono>

void foo()
{
    // simulate expensive operation
    std::this_thread::sleep_for(std::chrono::seconds(1));
}

int main()
{
    std::cout << "starting thread...\n";
    std::thread t(foo); // 构造线程对象，且传入被执行的函数。

    std::cout << "waiting for thread to finish..." << std::endl;
    t.join(); // 加入主线程，使得主线程必须等待该线程执行完毕。

    std::cout << "done!\n";
}
```

调用线程和线程之间同步数据，使用promise和future
```cpp
#include <vector>
#include <thread>
#include <future>
#include <numeric>
#include <iostream>
 
void accumulate(std::vector<int>::iterator first,
                std::vector<int>::iterator last,
                std::promise<int> accumulate_promise)
{
    int sum = std::accumulate(first, last, 0);
    accumulate_promise.set_value(sum);  // Notify future
}
 
int main()
{
    // Demonstrate using promise<int> to transmit a result between threads.
    std::vector<int> numbers = { 1, 2, 3, 4, 5, 6 };
    std::promise<int> accumulate_promise;
    std::future<int> accumulate_future = accumulate_promise.get_future();
    std::thread work_thread(accumulate, numbers.begin(), numbers.end(),
                            std::move(accumulate_promise));
 
    // future::get() will wait until the future has a valid result and retrieves it.
    // Calling wait() before get() is not needed
    //accumulate_future.wait();  // wait for result
    std::cout << "result = " << accumulate_future.get() << '\n';
    work_thread.join();  // wait for thread completion
}
```


但是，std::thread的执行并不能保证是异步的，也可能是在当前线程执行。

如果需要强制异步，则可使用std::async。它可以指定两种异步方式：std::launch::async和std::launch::deferred，前者表示使用新的线程异步地执行任务，后者表示在当前线程执行，且会被延迟执行。使用范例：
```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <numeric>
#include <future>
#include <string>
#include <mutex>

std::mutex m;
struct X {
    void foo(int i, const std::string& str) {
        std::lock_guard<std::mutex> lk(m);
        std::cout << str << ' ' << i << '\n';
    }
    void bar(const std::string& str) {
        std::lock_guard<std::mutex> lk(m);
        std::cout << str << '\n';
    }
    int operator()(int i) {
        std::lock_guard<std::mutex> lk(m);
        std::cout << i << '\n';
        return i + 10;
    }
};

template <typename RandomIt>
int parallel_sum(RandomIt beg, RandomIt end)
{
    auto len = end - beg;
    if (len < 1000)
        return std::accumulate(beg, end, 0);

    RandomIt mid = beg + len / 2;
    // launch::async: 异步执行
    auto handle = std::async(std::launch::async,
        parallel_sum<RandomIt>, mid, end);
    int sum = parallel_sum(beg, mid);
    return sum + handle.get();
}

int main()
{
    std::vector<int> v(10000, 1);
    std::cout << "The sum is " << parallel_sum(v.begin(), v.end()) << '\n';

    X x;
    // Calls (&x)->foo(42, "Hello") with default policy:
    // may print "Hello 42" concurrently or defer execution
    auto a1 = std::async(&X::foo, &x, 42, "Hello");
    // Calls x.bar("world!") with deferred policy
    // prints "world!" when a2.get() or a2.wait() is called
    auto a2 = std::async(std::launch::deferred, &X::bar, x, "world!");
    // Calls X()(43); with async policy
    // prints "43" concurrently
    auto a3 = std::async(std::launch::async, X(), 43);
    a2.wait();                     // prints "world!"
    std::cout << a3.get() << '\n'; // prints "53"
} // if a1 is not done at this point, destructor of a1 prints "Hello 42" here
```

## UE多线程
UE的多线程实现上并没有采纳C++11标准库的那一套，而是自己从系统级做了封装和实现，包括系统线程、线程池、异步任务、任务图以及相关的通知和同步机制。

```cpp
// Engine\Source\Runtime\Core\Public\HAL\Runnable.h

class CORE_API FRunnable
{
public:
    virtual bool Init();    // 初始化, 成功返回True.
    virtual uint32 Run();    // 运行, 只有Init成功才会被调用.
    virtual void Stop();    // 请求提前停止.
    virtual void Exit();    // 退出, 清理数据.
};
```
FRunnable及其子类是可运行于多线程的对象，而与之对立的是只在单线程运行的类FSingleThreadRunnable：

```cpp
// 多线程禁用下的单线程运行的物体
class CORE_API FSingleThreadRunnable
{
public:
    virtual void Tick();
};
```
FRunnable的子类非常多。