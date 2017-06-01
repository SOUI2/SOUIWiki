原文链接：[《第十四篇：在SOUI中使用定时器》](http://www.cnblogs.com/setoutsoft/p/4014714.html)

### 前言：
定时器是win32编程中常用的制作动画效果的手段。在Win32编程中，可以使用`::SetTimer`来创建定时器，定时器消息会被会发到调用SetTimer时指定的HWND。

在SOUI中一般来说只有一个宿主窗口有HWND，所有的SWindow都属于一个宿主窗口，如此一来直接使用`::SetTimer`创建的定时器就难以直接分发到SWindow对象了。

### 在SOUI的控件中使用定时器
为了能够方便的在SWindow中使用定时器，在SOUI系统中，我们通过将定时器ID（共32位）按位进行分解：
```c++
class SOUI_EXP STimerID
    {
    public:
        DWORD    Swnd:24;        //窗口句柄,如果窗口句柄超过24位范围，则不能使用这种方式设置定时器
        DWORD    uTimerID:7;        //定时器ID，一个窗口最多支持128个定时器。
        DWORD    bSwndTimer:1;    //区别通用定时器的标志，标志为1时，表示该定时器为SWND定时器

        STimerID(SWND hWnd,char id)
        {
            SASSERT(hWnd<0x00FFFFFF && id>=0);
            bSwndTimer=1;
            Swnd=hWnd;
            uTimerID=id;
        }
        STimerID(DWORD dwID)
        {
            memcpy(this,&dwID,sizeof(DWORD));
        }
        operator DWORD &() const
        {
            return *(DWORD*)this;
        }
    };
```

低24位用来存储SWindow的窗口ID(swnd)。swnd是一个SWindow创建序号，在一个应用中，通常很难产生超过`0xFFFFFF(16777215)`个窗口对象，因此使用低24位来存储窗口ID在大部分情况下都是够用的了。

高8位中保留最高位设置为1,用来区别直接使用::SetTimer创建的定时器（不可以把最高位置1）。

剩下7位用于SWindow中作为定时器ID。因此在SOUI中，一个SWindow最多可以创建`0-127`个定时器。

##### 创建定时器：
`SWindow::SetTimer（0~127);`
```c++
/**
        * SWindow::SetTimer
        * @brief    利用窗口定时器来设置一个ID为0-127的SWND定时器
        * @param    char id --  定时器ID
        * @param    UINT uElapse --  延时(MS)
        * @return   BOOL 
        *
        * Describe  参考::SetTimer
        */
        BOOL SWindow::SetTimer(char id,UINT uElapse);
```

##### 销毁定时器:
`SWindow::KillTimer;`
```c++
/**
        * KillTimer
        * @brief    删除一个SWND定时器
        * @param    char id --  定时器ID
        * @return   void 
        *
        * Describe  
        */
        void KillTimer(char id);
```

##### 响应定时器消息：
在消息映射表中使用`MSG_WM_TIMER_EX`。参见：`SGifPlayer.h`
```c++
        SOUI_MSG_MAP_BEGIN()    
            MSG_WM_TIMER_EX(OnTimer)    //定时器消息
            MSG_WM_PAINT_EX(OnPaint)    //窗口绘制消息
            MSG_WM_SHOWWINDOW(OnShowWindow)//窗口显示状态消息
        SOUI_MSG_MAP_END()    
```

如果在一个窗口中必须要创建使用32位的定时器ID，在SOUI中可以使用另一个接口来实现:

```c++
/**
        * SetTimer2
        * @brief    利用函数定时器来模拟一个兼容窗口定时器
        * @param    UINT_PTR id --  定时器ID
        * @param    UINT uElapse --  延时(MS)
        * @return   BOOL 
        *
        * Describe  由于SetTimer只支持0-127的定时器ID，SetTimer2提供设置其它timerid
        *           能够使用SetTimer时尽量不用SetTimer2，在Kill时效率会比较低
        */
        BOOL SetTimer2(UINT_PTR id,UINT uElapse);

        /**
        * KillTimer2
        * @brief    删除一个SetTimer2设置的定时器
        * @param    UINT_PTR id --  SetTimer2设置的定时器ID
        * @return   void 
        *
        * Describe  需要枚举定时器列表
        */
        void KillTimer2(UINT_PTR id);
```

##### 响应定时器:
使用`SWindow::SetTimer2`创建的定时器,最后会通过一个消息:`WM_TIMER2`来分发到`SWindow`。
```c++
#define WM_TIMER2    (WM_USER+5432)    //定义一个与HWND定时器兼容的SOUI定时器

#define MSG_WM_TIMER2(func) \
    if (uMsg == WM_TIMER2) \
{ \
    SetMsgHandled(TRUE); \
    func(wParam); \
    lResult = 0; \
    if(IsMsgHandled()) \
    return TRUE; \
}
```

### 在应用程序中使用定时器
前面两种定时器都是在控件开发的时候使用定时器的方法。而在应用层，还可以为宿主窗口直接使用`::SetTimer`或者宿主窗口的基类：`CSimpleWnd::SetTimer`来创建定时器**（注意最高位必须是0）**。

在响应这类定时器时，一样可以在宿主窗口的消息映射表中使用`MSG_WM_TIMER`来响应定时器消息。但是需要注意的是，这个映射宏会截获所有分发给宿主窗口的定时器，如果不是自己创建的定时器，则需要继续交给基类处理。

可以调用：`SetMsgHandled(FALSE);` 或者：`SHostWnd::OnTimer(UINT_PTR idEvent);`实现。