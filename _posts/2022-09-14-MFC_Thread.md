---
title: MFC 多线程教程
author: SHKChan
date: 2022-09-14
category: C++ / C
layout: post
---
记录一下最近编程过程中, 需要要到的MFC简单多线程教程, 以搬运原文和附上修改后的例子为主.

MFC中有两类线程, 可以分别称为工作线程和用户界面线程, 两者的主要区别在于是否有消息机制. 工作线程没有消息机制, 通常用于执行后台计算和维护任务. 用户界面线程通常用于处理独立的用户输入, 响应用户以及系统所产生得事件和消息.

   在MFC中，一般用全局函数AfxBeginThread()来创建并初始化一个线程的运行，该函数有两种重载形式，分别用于创建**工作线程**和**用户界面线程**。两种重载函数原型和参数分别说明如下：

## 工作者线程

```C++

CWinThread* AfxBeginThread(AFX_THREADPROC pfnThreadProc,
           LPVOID pParam,
           nPriority=THREAD_PRIORITY_NORMAL,
           UINT nStackSize=0,
           DWORD dwCreateFlags=0,
           LPSECURITY_ATTRIBUTES lpSecurityAttrs=NULL);
```

- **PfnThreadProc**:指向工作者线程的执行函数的指针，线程函数原型必须声明：UINT ExecutingFunction(LPVOID pParam);
  请注意，ExecutingFunction()应返回一个UINT类型的值，用以指明该函数结束的原因。一般情况下，返回0表明执行成功。
- **pParam**: 传递给线程函数的一个32位参数，执行函数将用某种方式解释该值。它可以是数值，或是指向一个结构的指针，甚至可以忽略；
- **nPriority**: 线程的优先级。如果为0，则线程与其父线程具有相同的优先级；
- **nStackSize**: 线程为自己分配堆栈的大小，其单位为字节。如果nStackSize被设为0，则线程的堆栈被设置成与父线程堆栈相同大小；
- **dwCreateFlags**：如果为0，则线程在创建后立刻开始执行。如果为CREATE_SUSPEND，则线程在创建后立刻被挂起；
- **lpSecurityAttrs**：线程的安全属性指针，一般为NULL；

## 用户界面线程

```C++
CWinThread* AfxBeginThread(CRuntimeClass* pThreadClass,
           int nPriority=THREAD_PRIORITY_NORMAL,
           UINT nStackSize=0,
           DWORD dwCreateFlags=0,
           LPSECURITY_ATTRIBUTES lpSecurityAttrs=NULL);
```

- **pThreadClass** 是指向 CWinThread 的一个导出类的运行时类对象的指针，该导出类定义了被创建的用户界面线程的启动、退出等；其它参数的意义同形式1。使用函数的这个原型生成的线程有消息机制。

下面我们对CWinThread类的数据成员及常用函数进行简要说明：

- **m_hThread**：当前线程的句柄；
- **m_nThreadID**: 当前线程的ID；
- **m_pMainWnd**：指向应用程序主窗口的指针。

```c++
BOOL CWinThread::CreateThread(DWORD dwCreateFlags=0,UINT nStackSize=0,
      LPSECURITY_ATTRIBUTES lpSecurityAttrs=NULL);
```

该函数中的dwCreateFlags、nStackSize、lpSecurityAttrs参数和**API函数CreateThread**中的对应参数有相同含义，该函数执行成功，返回非0值，否则返回0。
   所以一般情况下，调用AfxBeginThread()来一次性地创建并启动一个线程，但是也可以通过两步法来创建线程。首先创建CWinThread类的一个对象，然后调用该对象的成员函数CreateThread()来启动该线程。

virtual BOOL CWinThread::InitInstance();
　　重载该函数以控制用户界面线程实例的初始化。初始化成功则返回非0值，否则返回0。用户界面线程经常重载该函数，工作者线程一般不使用InitInstance()。

virtual int CWinThread::ExitInstance();
　　在线程终结前重载该函数进行一些必要的清理工作。该函数返回线程的退出码，0表示执行成功，非0值用来标识各种错误。同InitInstance()成员函数一样，该函数也只适用于用户界面线程。

## 实例：工作线程

```c++
//
UINT  myproc(LPVOID  lParam)
{
CITTDlg *pWnd = (CITTDlg *)lParam;         //将窗口指针赋给无类型指针
pWnd->KMeansSegment();                         //要执行的函数
return 1;
}

 //真正的数据或任务处理函数
void CITTDlg::KMeansSegment()
{
// 主要处理函数在这里写
}

 //响应用户界面，开辟线程，执行相关后台任务
void CITTDlg::OnKMeansSegment()             //按钮点击执行
{

AfxBeginThread(myproc, (LPVOID)this);//启动新的线程

}
```

---

## 参考与来源：

1. [VC启动一个新线程的三种方法](https://blog.csdn.net/u014568921/article/details/44262645#)
2. [多线程之三：MFC多线程及实例](https://blog.csdn.net/zhandoushi1982/article/details/6041430)
