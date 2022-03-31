---
title: C++11智能指针
date: 2022-03-30 21:44
categories: [C++, C++11代码优化与工程级应用学习]
tags: [smart pointer]     # TAG names should always be lowercase
---

智能指针是存储指向动态分配(堆)对象指针的类，用于生存期控制，能够确保在离开指针所在作用域时，自动正确地销毁动态分配的对象，防止内存泄漏。
它的一种通用实现技术是使用引用计数。每使用它一次，内部的引用计数加1，每析构一次，内部引用计数减1，减为0时，删除所指向的堆内存。C++11提供三种
智能指针: `std::shared_ptr`, `std::unique_ptr` 和 `std::weak_ptr`，使用时需要引用头文件`<memory>`。

## shared_ptr共享的智能指针
`std::shared_ptr` 使用引用计数，每一个`std::shared_ptr`的拷贝都指向相同的内存，在最后一个`shared_ptr`析构的时候，内存才会被释放。

可以通过构造函数，`std::make_shared<T>`辅助函数和`reset`方法来初始化`shared_ptr`，代码如下：
```c++
std::shared_ptr<int> p(new int(1));
std::shared_ptr<int> p2 = p;
std::shared_ptr<int> ptr;
ptr.reset(new int(1));

std::shared_ptr<T> sp{new T}; 
std::shared_ptr<T> sp = std::make_shared<T>();
```
优先使用`make_shared`来构造智能指针，更高的高效.
> Always use std::make_unique<T>()! \\
>> If we don't use make_shared, then we're allocating memory twice (once for sp, and once for new T)!

不能将一个原始指针直接赋值给一个智能指针，比如以下：
```
std::shared_ptr<int> p = new int(1); //编译报错，不允许直接赋值
```
对于一个未初始化的智能指针，可以用过`reset`方法来初始化，当智能指针中有值的时候，调用`reset`会使引用计数减1。另外，智能指针可以通过重载的`bool`类型操作符来判断智能指针中是否为空(未初始化)。

当需要获取原始指针时，可以通过`get()`方法来返回原始指针，如下:
```c++
std::shared_ptr<int> ptr(new int(1));
int* p = ptr.get();
```

使用`shared_ptr`需要注意的问题:
1. 不可以一个原始指针初始化多个`shared_ptr`，例如以下这是错误的:
```c++
int* ptr = new int;
shared_ptr<int> p1(ptr);
shared_ptr<int> p2(ptr);
```
2. 不可以在函数事参中创建`shared_ptr`:

```c++
// wrong
function(shared_ptr<int>(new int), g());
// true
shared_ptr<int> p(new int());
f(p, g());
```
3. `shared_ptr`的最大问题是需要避免循环引用，循环引用会导致内存泄漏。如下是典型场景:

```c++
struct A;
struct B;

struct A {
	std::shared_ptr<B> bptr;
	~A() { cout << "A is deleted!" << endl; }
};

struct B {
	std::shared_ptr<A> aptr;
	~B() { cout << "B is deleted!" << endl; }
};

void TestPtr() {
	{
		std::shared_ptr<A> ap (new A);
		std::shared_ptr<B> bp (new B);
		ap->bptr = bp;
		bp->aptr = ap;
	}
}
```

## unique_ptr独占的智能指针
`unique_ptr`是独占型的智能指针，不能通过将一个`unique_ptr`赋值给另一个`unique_ptr`, C++14后支持`make_unique<T>`来创建。独占智能指针不能复制，但可以通过函数返回给其他的`unique_ptr`，还可以通过`std::move`来转移到其他的指针，这样它本身就不再拥有原来指针的所有权了。

```c++
unique_ptr<T> myPtr(new T);
//wrong
unique_ptr<T> myOtherPtr = myPtr;
//true
unique_ptr<T> myOtherPtr = std::move(myPtr);
```

## weak_ptr
`weak_ptr`用来监视`shared_ptr`,纯粹作为一个旁观者来监视`shared_ptr`中管理的资源是否存在。
```c++
std::weak_ptr<T> wp = sp;
// can only be copy/move constructed (or empty)!
```
