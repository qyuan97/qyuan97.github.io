---
title: Design Pattern
date: 2022-03-28 17:44
categories: [C++, Design Pattern]
tags: [design_pattern]     # TAG names should always be lowercase
---

## 单例模式
单例模式(Singleton)的运用场景: 数据池，用来缓存数据的数据结构，需要在一处写，多处读或写。

最简单的单例模式实现如下:
``` c++
class singleton{
private:
	singleton() {}
	static singleton *p;
public:
	static singleton *instance();
};

singleton *singleton::p = nullptr;

singleton *singleton::instance(){
	if(p == nullptr)
		p = new singleton();
	return p
}
```
将构造函数声明为private或者protected防止被外部函数实例化，内部使用一个静态的类指针保存唯一的实例，实例的实现由一个public的方法来实现，该方法返回该类的唯一实例。

这个实现单线程是正确的，但是对于多线程是不安全的。当多个线程同时首次调用`instance()`方法且同时检测到`p`是`nullptr`时，则两个线程会同时构造一个实例给`p`, 导致单例的错误。

### 单例分为两种实现方式
* 懒汉模式: 第一次用到类实例的时候才会去实例化
* 饿汉模式: 单例类定义的时候就进行实例化

饿汉模式实现如下:
```c++
class singleton{
private:
	singleton() {}
	static singleton *p;
public:
	static singleton *instance();
}

singleton *singleton::p = new singleton();
singleton* singleton::instance(){
	return p;
}
```
饿汉模式本身是线程安全的。懒汉模式为线程不安全的。

### 多线程加锁
```c++
class singleton{
private:
	singleton() {}
	static singleton *p;
	static mutex lock_;
public:
	static singleton *instance();
};

singleton *singleton::p = nullptr;

singleton *singleton::instance() {
	lock_guard<mutex> guard(lock_);
	if(p == nullptr)
		p = new singleton();
	return p;
}

```
对`instance()`函数加锁后，代码是线程安全的。但是加锁开锁的操作只有初始化时才是必要的。在`p`被初始化后，不管多少线程同时访问，都只是读操作，不需要加锁，没有线程安全问题。加了锁之后反而存在了性能问题。

### 双重检查锁模式
双重检查锁: 对于读操作是不存在线程安全的，所以只需要在第一次实例创建时加锁，以后不需要。
``` c++
class singleton{
private:
	singleton() {}
	static singleton *p;
	static mutex lock_;
public:
	singleton *instance();
};

singleton *singleton::p = nullptr;

singleton* singleton::instance() {
	if(p == nullptr){
		lock_guard<mutex> guard(lock_);
		if(p == nullptr){
			p = new singleton();
		}
	}
	return p;
}
```
双重检查锁(DCLP)关键在于，大多数对于`instance()`的调用会看到`p`是非空的，因此不会尝试去初始化。因此，DCLP在尝试获得锁之前检查`p`是否为空。只有当检查成功时，才会去获得锁，然后再次检测`p`是否为空。第二次检查是必要的，因为另一个线程可能第一次检查之后，获得锁成功之前初始化`p`。

这个代码存在：内存读写的乱序执行(编译器问题)。

`p = new singleton();`

这条语句包含三个步骤:
1. 分配能够存储`singleton`对象的内存;
2. 在被分配的内存中构造一个`singleton`对象;
3. 让`p`指向这块被分配的内存.

这三个步骤并不是顺序执行的。只能确定1是最先执行的，步骤2，3却不一定。所以可能存在以下情况。
> * 线程A调用`instance()`，执行`p`的测试，获得锁，按照1,3执行，然后被挂起。此时`p`的非空的，但是`p`指向的内存中还没有`singleton`对象被构造。
* 线程B调用`instance()`，判定`p`非空，将其返回给调用者。但此时2还没执行，所以会出错。

### 静态局部变量
```c++
singleton *singleton::instance(){
	static singleton p;
	return &p;
}
```
* 单线程下，正确。
* c++及以后的版本的多线程下，正确。
* c++之前的多线程下，不一定正确。

C++ 11之后是线程安全的，这是因为新的C++标准规定了当一个线程正在初始化一个变量的时候，其他线程必须得等到该初始化完成之后才能访问它。

## 生产者消费者模型
```c++
#include <iostream>
#include <vector>
#include <mutex>
#include <thread>
#include <condition_variable>
#include <unistd.h>

using namespace std;

const int MAX_NUM = 1000;

class BoundedBuffer {
private:
	std::vector<int> array_;
	size_t start_pos_;
	size_t end_pos_;
	size_t pos_;
	std::mutex mtx_;
	std::condition_variable not_full_;
	std::condition_variable not_empty_;
public:
	BoundedBuffer(size_t n) {
		array_.resize(n);
		start_pos_ = 0;
		end_pos_ = 0;
		pos_ = 0;
	}
	void Produce(int n){
		{
			std::unique_lock<std::mutex> lock(mtx_);
		// wait for not full
		not_full_.wait(lock, [=]{return pos_ < array_size();});

		usleep(1000);
		array_[end_pos_] = n;
		end_pos_ = (end_pos_ + 1) % array_size();
		++pos_;
		cout << "Produce pos: " << pos_ << endl;
		}
		not_empty_.notify_one();
	}

	int Consume() {
		std::unique_lock<std::muxtex> lock(mtx_);
		//wait for not empty
		not_empty_.wait(lock, [=]{return pos_ > 0;});

		usleep(1000);
		int n = array_[start_pos_];
		start_pos_ = (start_pos_ + 1) % array_size();
		--pos_;
		cout << "Consume pos: " << pos_ << endl;
		lock.unlock();
		not_full_.notify_one();
		return n;
	}
};

BoundedBuffer bb(10);

void Producer() {
	int n = 0;
	while(n < 100){
		bb.Produce(n);
		cout << "Producer: " << n << endl;
		n++;
	}
	bb.Produce(-1);
}

void Consumer() {
	std::thread::id thread_id = std::this_thread::get_id();
	int n = 0;
	do{
		n = bb.Consume();
		cout << "Consumer thread: " << thread_id << "=====>" << n << endl;
	}while(n != -1);
	bb.Produce(-1);
}

int main() {
	std::vector<<std::thread> t;
	t.push_back(std::thread(&Producer));
	t.push_back(std::thread(&Consumer));
	t.push_back(std::thread(&Consumer));
	t.push_back(std::thread(&Consumer));

	for(auto& one : t){
		one.join();
	}

	return 0;
}

```


> Reference: 
1. [https://github.com/Light-City/CPlusPlusThings/tree/master/design_pattern/singleton](https://github.com/Light-City/CPlusPlusThings/tree/master/design_pattern/singleton)
2. [hhttps://github.com/Light-City/CPlusPlusThings/tree/master/design_pattern/producer_consumer](https://github.com/Light-City/CPlusPlusThings/blob/master/design_pattern/producer_consumer/producer_consumer.cpp)