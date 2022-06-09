---
title: Thread and mutex
date: '2021-06-28'
---
# 大二OS课程小作业
# 前言
今天是第18周的周三，上周末一听说操作系统被分配到“使用线程与条件变量实现面包师问题”，以前没学过任何C++并行的知识（毕竟太菜），就立刻去看了看C++文档，发现线程、条件变量、互斥量库一应俱全，这下就开心了。现在课设也做的差不多了，写些感悟要写到博客上。源代码：https://github.com/CirnoTox/OS_project
# 问题说明
## 原题 面包师问题
面包师有很多面包和蛋糕，由 n 个销售人员销售。每个顾客进店后先取一个号，并且等着叫号。当一个销售人员空闲下来，就叫下一个号。请分别编写销售人员和顾客进程的程序，要求使用线程与条件变量实现。
## 形式化分析
给定已有面包x个、蛋糕y个；销售人员个数n；进店顾客数量c,每个顾客购买随机个面包和蛋糕；要求销售人员们根据顾客队列进行销售，如果只剩少量面包或蛋糕，则拒绝过量购买意愿的顾客，销售结束后返回剩余面包restx与蛋糕数量resty。

# 源码说明
* 三个主文件`customer.cpp`,`OS_project.cpp`,`saler.cpp`
* 一个辅助类`pcout.h`

## 队列、向量与线程创建
`deque`和`vector`有着`emplace_back()`这样的优秀成员函数，为他们传入`thread`模板参数后，`emplace_back()`可以帮助coder在向`deque`和`vector`添加数据的同时启动线程。
```cpp
// 以i为参数，启动customer个customerThread线程
for (auto i = 0; i < customer; ++i) {
   customerThreadQ.emplace_back(customerThread,i);
}
```

当然，创建线程以后一定要记得`join()`，不过就算忘了，C++也会通过报错提醒你。

```cpp
for (thread& t : salerThreadP) {
   t.join();
}
```
## 随机数生成
在customerThread线程中，顾客生成数据我使用了随机数，并且让这随机数服从以(1,4)为参数伽玛分布`gamma_distribution<>`，具体是这样调用的

```cpp
random_device rd;   
mt19937 gen(rd());  
gamma_distribution<> dist(1, 4);
size_t needCake = size_t(dist(gen));
```

C++11开始，随机数有更好的调用方法，不过这不是这次课设的核心，随机数能用就行啦。
## 数据上锁
这里涉及到多线程地向队列存取数据，所以使用锁`unique_lock<>`来保证数据的顺序与安全性。

```cpp
//首先要有一个公共的互斥量mutex
extern mutex customerDataMutex;
//上锁的基本框架
static void customerThread(size_t number)
{
   //用公共的互斥量实例化一个unique_lock
   unique_lock<mutex> dataLock(customerDataMutex);
   //实例化结束后就立刻锁上了！
   //如果不想立刻上锁，可以在实例化时加上一个defer_lock参数，之后再用成员函数lock()上锁
   ...//生成数据
   //向队列存数据
   dataQ.push(vector<size_t>{number,needBread,needCake});
   //解锁
   dataLock.unlock();
   ...//do other things
}
```

值得注意的是，`unique_lock`在执行构析函数时也会自动解锁，所以一个简单的互斥线程框架是：

```cpp
mutex m;//公共互斥量
void thread()//并行的线程
{
   unique_lock<mutex>lock;
   ...//do othert things, 期间一直上锁，直到最终执行完毕
}
```

### 并行中的标准输出
并行标准输出类`struct pcout`的原理就是借用了上述的原理。

```cpp
static inline mutex cout_mutex;//公共互斥量
~pcout() {
   lock_guard<mutex> l{ cout_mutex };//上锁
   cout << rdbuf();//做输出
   cout.flush();//刷新到屏幕
}
```

## 条件变量
至少使用一次的条件变量，使用在了`salerThread`和`customerThread`两个线程的通信中。`customerThread`在生成完数据后会进入条件变量的等待wait，`salerThread`线程操作数据结束后，会提醒notify`customerThread`结束。

```cpp
//公共互斥量，条件变量，条件
extern mutex customerSaleFinishMutex;
extern condition_variable customerSaleFinishCV;
extern int customerIdSaleFinished;

static void customerThread(size_t number)
{
   ...//生成数据部分
   //条件变量必须与mutex和lock共同使用
   unique_lock<mutex> saleFinishLock(customerSaleFinishMutex);
   customerSaleFinishCV.wait(saleFinishLock,
   //等待customerIdSaleFinished为变为当前线程序号
      [&]() { return customerIdSaleFinished==number; });
   pcout{} << "Customer " << number << " leave.\n";
}

static void salerThread(size_t number)
{
   while (1) {
      ...//do some things
      unique_lock<mutex> salerLock(salerMutex);//上锁
      ...//处理数据
      //提醒顾客离开
      //因为处理数据时有上锁，就不用担心数据访问混乱
      customerIdSaleFinished = customerId;
      //notify所有顾客线程，确认条件customerIdSaleFinished情况
		customerSaleFinishCV.notify_all();
      salerLock.unlock();//解锁
   }
}
```

# Linux
用git获取代码，然后各个文件都要多包含一些头文件

```cpp
#include <iostream>
#include <vector>
#include <thread>
#include <mutex>
#include <queue>
#include<condition_variable>
#include <sstream> //pcout.h下
```

G++编译指令：

```
g++ -o exefile.o OS_project.cpp saler.cpp customer.cpp pcout.h -fpermissive -pthread
```

# 拓展阅读与参考
* https://github.com/xiaoweiChen/CPP-17-STL-cookbook 并行详见第9章
* https://docs.microsoft.com/en-us/cpp/standard-library/cpp-standard-library-reference
* https://en.cppreference.com/w/