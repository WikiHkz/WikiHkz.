---
layout: post
title: 多线程hello world
date: 2023-07-31
categories: master
tags: recommend
keywords: cpp
description: 
---

## Code

```c++
#include <iostream>
#include <thread>
#include <condition_variable>
#include <chrono>

std::condition_variable cv;
std::mutex mtx;
bool paused = false;

void printHelloWorld() {
    while (true) {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, []() -> bool { return !paused; });

        std::cout << "Hello World" << std::endl;
        lock.unlock();

        std::this_thread::sleep_for(std::chrono::seconds(1));
    }
}

int main() {
    std::thread printingThread(printHelloWorld);

    char input;
    while (true) {
        std::cin >> input;
        if (input == '-') {
            std::unique_lock<std::mutex> lock(mtx);
            paused = true;
        } else {
            std::unique_lock<std::mutex> lock(mtx);
            paused = false;
            cv.notify_one();
        }
    }

    printingThread.join();
    return 0;
}
```

## 解析

```c++
std::condition_variable cv;
std::mutex mtx;
bool paused = false;
```

​	`cv`是一个全局的条件变量，`mtx`是锁，`paused`用来表明当前程序是否进入暂停状态。

### 线程

​	`void printHelloWorld()`是线程中执行的函数，使用c++标准库中的`std::thread`新建一个线程`printingThread`，并把需要执行的函数作为函数指针传递进去，下文中都称呼为子线程。

​	在`void printHelloWorld()`中执行的是一个死循环，输出hello world，但区别在于，程序可能会被键盘输入的字符`-`打断进入暂停态。在这段代码中使用了全局变量`paused`来表明程序当前的状态。

​	这是一个全局的状态，因此在多线程中会产生竞争关系，在对临界区进行读写时需要锁来进行保护。在子线程开头，先获取`mtx`锁，保证对临界区访问的独占性。

#### std::unique_lock\<std::mutex\>

​	这是标准库中的独占锁。

​	这个模板类的构造函数是获取锁(上锁)，析构函数是释放锁，也可以通过类成员函数unlock()手动释放锁。在离开当前作用域时，锁会被自动释放。

#### cv.wait()

​	再获取`mtx`锁以后，等待条件变量`cv`。

```c++
cv.wait(lock, []() -> bool { return !paused; });
// 声明如下
template<class Predicate>
void wait(std::unique_lock<std::mutex>& lock, Predicate pred);
```

​	**cv.wait()**的内容主要是：

1. 获取*std::unique_lock\<std::mutex\>& lock*锁
2. 获取了锁之后，检查等待条件`pred`，如果为true，直接执行下一行代码(**不会自动释放锁**，需要手动释放)，如果为false，则线程**释放锁**并且进入阻塞(wait)状态直到被唤醒
3. 通过其他线程调用了 `cv.notify_one()` 或 `cv.notify_all()`来唤醒当前线程
4. 唤醒当前线程可以理解为重新执行`cv.wait()`函数，下一行是否执行(真正意义上的唤醒线程)取决于`pred`.`cv.notify_one()` 或 `cv.notify_all()`相当于只是把当前线程扒拉起来，看看问问他有没有睡醒，睡醒了就起来干活，没睡醒就让他继续睡。醒没醒由`pred`决定。

如果当前的程序状态是**paused == false**，即当前程序不处于暂停状态，子线程在获取锁，条件变量wait后，输出helloworld，然后手动释放锁，并睡眠1秒后进行第二次循环。

### 主线程

```c++
std::thread printingThread(printHelloWorld);
```

​	主线程在子线程创立后，两个线程就是独立运行的了。

​	子线程会不断进行循环：

- 获取锁，保护临界区
- 在临界区读全局变量**paused** 是不是暂停状态
- 不是暂停状态，输出hello word，手动释放锁，继续循环
- 是暂停状态，线程进入阻塞状态(cv.wait())，等待主线程在合适的时候唤醒子线程继续执行

​	主线程会进入死循环，不断通过std::cin读取键盘的输入，并根据键盘的输入情况，获取锁进入临界区，在临界区操作全局变量**paused**。由于是使用的std::unique_lock\<std::mutex\>，因此在离开时无需手动释放锁，析构函数会自动释放锁。

