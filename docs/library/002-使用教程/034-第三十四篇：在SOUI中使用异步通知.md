原文链接：[《第三十四篇：在SOUI中使用异步通知》](http://www.cnblogs.com/setoutsoft/p/5642056.html)

### 概述
异步通知是客户端开发中常见的需求，比如在一个网络处理线程中要通知UI线程更新等等。

通常在Windows编程中，为了方便，我们一般会向UI线程的窗口句柄`Post/Send`一个窗口消息从而达到将非UI线程的事件切换到UI线程处理的目的。

在SOUI引入通知中心以前要在SOUI中处理非UI线程事件我也推荐用上面的方法。

使用窗口消息至少有以下两个不足：

1. 需要在线程中持有一个窗口句柄。

2. 发出的消息只能在该窗口句柄的消息处理函数里处理。

### SNotifyCenter
最新的SOUI引入了一个新的单例对象：`SNotifyCenter`，专门用来处理这类问题。

新的SNotifyCenter解决了窗口消息存在的上面的两个问题：

1. 通过使用全局单例，`SNotifyCenter`可以在代码任意位置获取它的指针（前提当然是要在它的生命周期内）；

2. 使用`SNotifyCenter`产生的通知采用SOUI的事件系统来派发，通过结合SOUI的事件订阅系统，用户可以在任意位置处理发出的事件。

在介绍如何使用`SNotifyCenter`前，先看一下`NotifyCenter.h`的代码：

```c++
#pragma once

#include <core/SSingleton.h>

namespace SOUI
{
    template<class T>
    class TAutoEventMapReg
    {
        typedef TAutoEventMapReg<T> _thisClass;
    public:
        TAutoEventMapReg()
        {
            SNotifyCenter::getSingleton().RegisterEventMap(Subscriber(&_thisClass::OnEvent,this));
        }

        ~TAutoEventMapReg()
        {
            SNotifyCenter::getSingleton().UnregisterEventMap(Subscriber(&_thisClass::OnEvent,this));
        }

    protected:
        bool OnEvent(EventArgs *e){
            T * pThis = static_cast<T*>(this);
            return !!pThis->_HandleEvent(e);
        }
    };

    class SOUI_EXP SNotifyCenter : public SSingleton<SNotifyCenter>
                        , public SEventSet
    {
    public:
        SNotifyCenter(void);
        ~SNotifyCenter(void);

        /**
        * FireEventSync
        * @brief    触发一个同步通知事件
        * @param    EventArgs *e -- 事件对象
        * @return    
        *
        * Describe  只能在UI线程中调用
        */
        void FireEventSync(EventArgs *e);

        /**
        * FireEventAsync
        * @brief    触发一个异步通知事件
        * @param    EventArgs *e -- 事件对象
        * @return    
        *
        * Describe  可以在非UI线程中调用，EventArgs *e必须是从堆上分配的内存，调用后使用Release释放引用计数
        */
        void FireEventAsync(EventArgs *e);


        /**
        * RegisterEventMap
        * @brief    注册一个处理通知的对象
        * @param    const ISlotFunctor &slot -- 事件处理对象
        * @return    
        *
        * Describe 
        */
        bool RegisterEventMap(const ISlotFunctor &slot);

        /**
        * RegisterEventMap
        * @brief    注销一个处理通知的对象
        * @param    const ISlotFunctor &slot -- 事件处理对象
        * @return    
        *
        * Describe 
        */
        bool UnregisterEventMap(const ISlotFunctor & slot);
    protected:
        void OnFireEvent(EventArgs *e);

        void ExecutePendingEvents();

        static VOID CALLBACK OnTimer( HWND hwnd,
            UINT uMsg,
            UINT_PTR idEvent,
            DWORD dwTime
            );

        SCriticalSection    m_cs;            //线程同步对象
        SList<EventArgs*>    *m_evtPending;//挂起的等待执行的事件
        DWORD                m_dwMainTrdID;//主线程ID
        
        UINT_PTR            m_timerID;    //定时器ID，用来执行异步事件

        SList<ISlotFunctor*>    m_evtHandlerMap;
    };
}
```

在这个文件中提供了两个类，一个就是`SNotifyCenter`，另一个是`TAutoEventMapReg<T>`。

可以看到`SNotifyCenter`中除构造外只有4个public方法: 

`FireEventSync`, `FireEventAsync`用来触发事件。

`RegisterEventMap`，`UnregisterEventMap`则用来提供事件处理订阅。

### 如何使用SNotifyCenter?

1. 在main中实例化`SNotifyCenter`。（不明白可以参考demo)

2. 定义需要通过通知中心分发的事件类型，首先定义事件，然后向通知中心注册，参见下面代码：

```c++
void CMainDlg::OnBtnStartNotifyThread()
{
    if(IsRunning()) return;
    SNotifyCenter::getSingleton().addEvent(EVENTID(EventThreadStart));
    SNotifyCenter::getSingleton().addEvent(EVENTID(EventThreadStop));
    SNotifyCenter::getSingleton().addEvent(EVENTID(EventThread));

    EventThreadStart evt(this);
    SNotifyCenter::getSingleton().FireEventSync(&evt);
    BeginThread();    
}

void CMainDlg::OnBtnStopNotifyThread()
{
    if(!IsRunning()) return;

    EndThread();
    EventThreadStop evt(this);
    SNotifyCenter::getSingleton().FireEventSync(&evt);

    SNotifyCenter::getSingleton().removeEvent(EventThreadStart::EventID);
    SNotifyCenter::getSingleton().removeEvent(EventThreadStop::EventID);
    SNotifyCenter::getSingleton().removeEvent(EventThread::EventID);
}
```

3. 使需要处理通知中心分发的事件的对象从TAutoEventMapReg继承，实现事件的自动订阅（方便在事件映射表中统一处理事件)，这一步是可选的，你也可以直接使用SOUI提供的事件订阅机制向通知中心订阅特定事件。

4. 在事件映射表里处理事件（没有第3步时，则同样没有这一步）。

<font color=red>具体用法参见SOUI的demo。</font>

