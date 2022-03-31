---
title: C++11多线程
date: 2022-03-30 10:20
categories: [C++, C++11代码优化与工程级应用学习]
tags: [multi_thread]     # TAG names should always be lowercase
---

## 线程的创建
```c++
#include <thread>
void func() {
	// do some work
}
int main() {
	std::thread t(func);
	t.join();
	return 0;
}
```
在上例中，函数`func`会运行与线程对象`t`中，`join`函数将会阻塞线程，直到线程函数执行结束，如果线程函数有返回值，返回值将会被忽略。如果不希望线程被阻塞执行，可以调用线程的`detach()`方法，将线程和线程对象分离。通过detach，线程和线程对象分离里，让线程作为后台线程去执行，当前线程也不会阻塞了。但需要注意的是，detach之后就无法再和线程发生联系了，比如detach之后就不能再通过join来等待线程执行完，线程何时执行完我们也无法控制了。

需要注意的是，`std::thread`出了作用域之后将会析构，这时如果线程函数还没有执行完则会发生错误，因此，需要保证线程函数的生命周期在线程变量`std::thread`的生命周期之内。

线程不可以复制，但是可以移动
```c++
#include <thread>

void func () {
	//do some work
}

int main () {
	std::thread t(func);
	std::thread t1(std::move(t))
	t.join();
	t1.join();
}
```

线程被移动之后，线程对象t将不再代表任何线程了。

必须保证线程对象的生命周期在线程函数执行完时仍然存在。可通过以下三种方式：
1. 可以通过join的方式来阻塞等待线程函数执行完
2. 可以通过detach方式让线程在后台执行
3. 可以将线程对象保存到一个容器中，以保证线程对象的生命周期

```c++
#include <thread>

std::vector<std::thread> g_list;
std::vector<std::shared_ptr<std::thread>> g_list2;
void CreateThread () {
	std::thread t(func);
	g_list.push_back(std::move(t));

	g_list2.push_back(std::make_shared<std::thread>(func));
}
int main() {
	CreateThread();
	for (auto& thread: g_list) {
		thread.join();
	}
	for (auto& thread: g_list2) {
		thread->join();
	}
	return 0;
}
```

## 互斥量
`std::mutex`独占互斥量
``` c++
std::mutex mtx;
mtx.lock();
mtx.unlock();
```
使用`lock_guard`来简化写法，也更加安全。`lock_guard`在构造时会自动锁定互斥量，而在退出作用域后进行析构时就会自动解锁，从而保证了互斥量的正确操作，避免忘记unlock操作。尽量使用lock_guard。lock_guard使用了RAII技术。这种技术在类的构造函数中分配资源，在析构函数中释放资源，保证资源在出了作用域之后就释放。

```
void func() {
	std::lock_guard<std::mutex> locker(g_lock);
}
```

`std::recursive_mutex`递归锁，允许统一线程多次获得互斥量。尽量不使用递归锁。

## 条件变量
条件变量是另外一种用于等待的同步机制，它能阻塞一个或多个线程，直到收到另外一个线程发出的通知或者超时，才会唤醒当前阻塞的线程。条件变量需要和互斥量配合起来用。
1. `condition_variable`, 配合`std::unique_lock<std::mutex>`进行wait操作
2. `condition_variable_any`， 和任意可以搭配使用，灵活，但效率差。
条件变量的使用过程:
1. 拥有条件变量的线程获取互斥量
2. 循环检查某个条件，如果条件不满足，则阻塞直到条件满足；如果条件满足，则向下执行
3. 某个线程满足条件执行完之后调用notify_one或者notify_all唤醒其他等待线程
可以用条件变量来实现一个同步队列，同步队列作为一个线程安全的数据共享区，经常用于线程之间数据读取。