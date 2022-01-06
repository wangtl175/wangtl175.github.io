# 一个线程能访问另一个线程的栈吗

这是面试时遇到一个很有意思的问题。背过八股的同学都知道，一个进程里的线程共享代码段、全局数据段、BSS段、堆，只有栈是私有的，那么一个线程可以访问到另一个线程的栈吗？

一开始我以为线程栈既然是私有的，就可能有一些保护措施来避免被别的线程访问，但是面试官似乎对我这个回答并不满意。于是这里用实验探究一下，大体的思路就是使用一个指针执行一个线程里的局部变量，这个指针是一个全局变量，然后在另一个线程里通过这个指针去访问和修改局部变量。

```c++
#include <thread>
#include <cstdio>
#include <mutex>
#include <condition_variable>

std::condition_variable cv1, cv2;
std::mutex mtx1, mtx2;
bool cp1 = false, cp2 = false;

volatile int *ptr;  // 全局变量，fun1和fun2都可访问

void fun1() {
    volatile int localVal = 1234;
    ptr = &localVal;
    printf("fun1: %d\n", localVal);
    {
        std::unique_lock<std::mutex> lck1(mtx1);
        cp1 = true;
    }
    cv1.notify_one();  // 唤醒fun2

    std::unique_lock<std::mutex> lck2(mtx2);  // 等待fun2修改完成
    cv2.wait(lck2, [&]() { return cp2; });

    printf("fun1: %d\n", localVal);
}

void fun2() {
    std::unique_lock<std::mutex> lck1(mtx1); // 等待fun1取到地址
    cv1.wait(lck1, [&]() { return cp1; });
    printf("fun2: %d\n", *ptr);
    *ptr = 4321;
    printf("fun2: %d\n", *ptr);
    {
        std::unique_lock<std::mutex> lck2(mtx2);
        cp2 = true;
    }
    cv2.notify_one();  // 唤醒fun1
}

int main() {
    std::thread t1(fun1);
    std::thread t2(fun2);
    t1.join();
    t2.join();
}
```

结果

```
fun1: 1234
fun2: 1234
fun2: 4321
fun1: 4321
```

这里可以看出，其实一个线程可以访问整个进程空间的。
