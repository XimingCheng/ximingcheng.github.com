<!-- 
.. link: 
.. description: 
.. tags: ZThread,thread,encapsulation
.. date: 2014/01/20 10:16:04
.. title: ZThread源码剖析——Win32线程封装
.. slug: zthread-source-Win32-thread-encapsulation
-->

##ZThread 线程最简使用##

想必大家对C++ 03中对线程库的标准支持缺失很是烦恼，C++11出现的时间太晚，ZThread是一个跨平台的线程库，可以使用面向对象的线程模型，API接口设计的很像Java，最简单的实现一个线程莫过于下面的代码：

```CPP
#include <zthread/Runnable.h>
#include <zthread/Thread.h>
#include <iostream>

using namespace std;
using namespace ZThread;

class TestRun : public Runnable
{
public:
  void run()
  {
    cout<<"hello world"<<endl;
  }
};

void testfun()
{
  Thread t0(new TestRun());
  Thread t1(new TestRun());
}

int main(int argc, char* argv[])
{
  testfun();
  return 0;
}
```

从上面可以明显的看出，只要实现一个Runnable的继承类，重新实现Runnable的纯虚函数run方法就可以实现出一个简单的线程类。使用的时候创建线程对象然后把自己实现好的对象指针传入进去即可。这个时候线程就可以正常运行，不过没有对线程异常进行捕捉和处理。

##Runnable Task的简单初窥##

Runnable这个class实现的非常简单，我这里就不详细复述了，他是一个抽象类，实现了一个纯虚函数的接口叫做run，所有的线程处理函数几乎都在这个函数下面大放光彩。

```CPP
virtual void run() = 0;
```

ZThread中需要把Runable的对象包装成Task，即任务之说。
Task的实现是一个智能指针，关于ZThread的智能指针，我以后还会详细分析。
Task的实现也非常的简单，重要的是它同时提供了仿函数的接口，即重载了()：

```CPP
void operator()() {
  (*this)->run();
}
```
##ThreadOps线程操作##
ThreadOps是对线程的一个封装，我这里详细的分析一下ThreadOps的Win32实现：
###dispatch###
dispatch函数是一个static函数，并且声明的时候标明了stdcall，这个是Win API的常见写法，他的作用是给线程分发任务，作为线程的回调函数来使用。

```CPP
  // 线程句柄
  HANDLE _hThread;
  // 线程ID
  DWORD _tid;
```

这两个是ThreadOps唯一存在的两个私有的成员属性

###self###
返回当前线程对应的ThreadOps对象
代码后面的几个函数例如激活线程对象，判断ThreadOps对象是否对应当前线程，==的重载，优先级读取和设置等等很容易理解，这里就不在说明了。

###线程的创建###
使用spawn函数创建线程，当然为了保证线程拥有自己独立的全局存储区，Win32下面如果发现可以使用_beginthread和_endthread宏的话，就尽量使用这连个宏来控制线程，_beginthread实际上底层调用的就是CreateThread，但是他给线程封装了更为安全的全局存储区，在你是用一些C语言库函数对一些全局错误标志的修改可以起到一定的保护作用。
在创建线程的时候，传递dispatch的函数指针作为线程的回调函数，同时传递一个Runnable实例化好的指针作为线程函数的参数。
```CPP
bool ThreadOps::spawn(Runnable* task) {

// Start the thread. 创建线程 注意用的是 _beginthreadex 把task作为参数传递进去
#if defined(HAVE_BEGINTHREADEX)
  _hThread = (HANDLE)::_beginthreadex(0, 0, &_dispatch, task, 0, (unsigned int*)&_tid);
#else
  _hThread = CreateThread(0, 0, (LPTHREAD_START_ROUTINE)&_dispatch, task, 0, (DWORD*)&_tid);
#endif

  return _hThread != NULL;

}
```
###线程join###
线程等待结束使用WaitForSingleObjectEx，使用无限长等待，当等待结束线程句柄被关闭。
```CPP
bool ThreadOps::join(ThreadOps* ops) {

  assert(ops);
  assert(ops->_tid != 0);
  assert(ops->_hThread != NULL);
  
  // 等待线程结束
  if(::WaitForSingleObjectEx(ops->_hThread, INFINITE, FALSE) != WAIT_OBJECT_0) 
    return false;

  ::CloseHandle(ops->_hThread);
  ops->_hThread = NULL;

  return true;

}
```
###线程让步和YieldOps函数对象（仿函数）###
线程让步在Windows下面调用SwitchToThread API进行线程让步，但是SwitchToThread属于Platform API，其代码实现被封装在Kernel32.dll中，所以ZThread使用了YieldOps封装了一个仿函数以供外围接口调用。
```CPP
class YieldOps {

  typedef BOOL (*Yield)(void);
  Yield _fnYield; // yield函数指针

public:

  YieldOps() : _fnYield(NULL) {

    OSVERSIONINFO v;
    v.dwOSVersionInfoSize = sizeof(OSVERSIONINFO);

    // NT 内核 找到SwitchToThread函数入口
    if(::GetVersionEx(&v) && (v.dwPlatformId == VER_PLATFORM_WIN32_NT)) {
       HINSTANCE hInst = ::GetModuleHandle("Kernel32.dll");
       if(hInst != NULL)
         _fnYield = (Yield)::GetProcAddress(hInst, "SwitchToThread");

       // REMIND: possibly need to use _T() macro for these strings
    }

  }

  // 仿函数 括号重载 系统调用失败 尝试用sleep(0)来yield线程
  bool operator()() {
  
    // Attempt to yield using the best function available
    if(!_fnYield || !_fnYield()) 
      ::Sleep(0);  

    return true;

  }

};
```
实际的ThreadOps::yield()只是初始化了一个静态的static YieldOps yielder：
```CPP
bool ThreadOps::yield() {

  // 注意是静态的 只初始化了一次
  static YieldOps yielder;

  yielder();

  return true;

}
```
###dispatch虚函数发威###
dispatch是类的静态成员，是没有this指针的，同时Win回调函数是不能使用class的成员函数的，所以在spawn函数中传递给了线程回调函数一个Runable指针参数，线程函数直接使用暴力强制类型转换得到正确的Runable对象指针
后续就是执行run方法，然后是线程资源回收
```CPP
// 分配线程任务 线程函数
unsigned int __stdcall ThreadOps::_dispatch(void *arg) {

  // 参数传递进入 强制类型转化成为线程执行的task
  Runnable* task = reinterpret_cast<Runnable*>(arg);
  assert(task);

  // Run the task from the correct context
  // 线程处理函数 执行线程run的虚函数
  task->run();

  // Exit the thread
#if defined(HAVE_BEGINTHREADEX)
  ::_endthreadex(0);
#else
  ExitThread(0);
#endif

  return 0;
  
}
```