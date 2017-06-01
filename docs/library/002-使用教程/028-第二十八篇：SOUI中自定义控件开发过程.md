原文链接：[《第二十八篇：SOUI中自定义控件开发过程》](http://www.cnblogs.com/setoutsoft/p/4711308.html)

在SOUI中已经提供了大部分常用的控件，但是内置控件不可能满足用户的所有要求，因此一个真实的应用少不得还要做一些自定义控件。

学习一个新东西，最简单的办法就是依葫芦画瓢。事实上在SOUI系统中内置控件和自定义控件的开发流程是完全一样的，因此只需要打开SOUI的源代码，随便找一个控件看一下就大体差不多了。

下面我以`controls.extend`目录下的的`SRadioBox2`控件为例对控件开发过程需要注意的地方做一点说明。

要开发一个控件，首先要确定的是应该从哪个控件来继承。选择一个合适的基类是正确开发自定义控件的前提。

之所以要开发一个`SRadioBox2`控件，我需要解决的问题很简单：`SRadioBox`控件总是在左边显示一个圆圈，这个圆圈有时候不是我想要的。

因此我需要做的就是继承`SRadioBox`控件的行为，重写`WM_PAINT`的处理。

因此就有了下面的代码：

```c++
 class SRadioBox2 : public SRadioBox
    {
    public:
        SRadioBox2(void);
        ~SRadioBox2(void);
    }
```

需要注意的是，所有SOUI控件都是在`namespace SOUI`中，因此自定义控件也最好是放在SOUI这个`namespace`里。

有了上面的骨架，下面来逐步添加内容。

首先我们需要给自定义控件定义一个在XML中可以识别的标签。

只需要在类的最开始增加一行：

```c++
class SRadioBox2 : public SRadioBox
    {
    SOUI_CLASS_NAME(SRadioBox2,L"radio2")
    public:
        SRadioBox2(void);
        ~SRadioBox2(void);
    }
```

SOUI_CLASS_NAME告诉XML解析器，碰到radio2时自动创建`SRadioBox2`对象。

实际上这一行更重要的作用是用来做对象类型运行时识别（`RTTI`）,有了这个机制，在编译器关闭C++的RTTI时仍然可以安全的进行类型转换。

然后我们需要处理控件的`WM_PAINT`消息。

为了处理这个消息，我们需要加入消息映射表及消息处理函数：

```c++
class SRadioBox2 : public SRadioBox
    {
    SOUI_CLASS_NAME(SRadioBox2,L"radio2")
    public:
        SRadioBox2(void);
        ~SRadioBox2(void);
        

    protected:       
        void OnPaint(IRenderTarget *pRT);

        SOUI_MSG_MAP_BEGIN()
            MSG_WM_PAINT_EX(OnPaint)
        SOUI_MSG_MAP_END()
    };
```

SOUI控件的消息处理机制是和WTL中抄过来的，和MFC也很相似。只是对于部分消息，由于对于消息的参数的解释不一样，消息映射的宏会有一点变化。

如这里的`WM_PAINT`消息，在SOUI里wparam传递的是一个`IRenderTarget`指针，而传统的Win32传递的是一个HDC。因此我们需要使用`MSG_WM_PAINT_EX`代替WTL中使用的`MSG_WM_PAINT`。

大家可能会问有哪些消息映射宏SOUI和WLT不一样？

实际上对于一个有经验的程序员，他应该可以找到`MSG_WM_PAINT_EX`的宏定义，并且在定义附近就可以找到所有的SOUI和WLT不同的映射宏。

下面是到目前为止SOUI中所有和WTL不同的宏：

```c++
// BOOL OnEraseBkgnd(IRenderTarget * pRT)
#define MSG_WM_ERASEBKGND_EX(func) \
    if (uMsg == WM_ERASEBKGND) \
    { \
    SetMsgHandled(TRUE); \
    lResult = (LRESULT)func((IRenderTarget *)wParam); \
    if(IsMsgHandled()) \
    return TRUE; \
    }

// void OnPaint(IRenderTarget * pRT)
#define MSG_WM_PAINT_EX(func) \
    if (uMsg == WM_PAINT) \
    { \
    SetMsgHandled(TRUE); \
    func((IRenderTarget *)wParam); \
    lResult = 0; \
    if(IsMsgHandled()) \
    return TRUE; \
    }

// void OnNcPaint(IRenderTarget * pRT)
#define MSG_WM_NCPAINT_EX(func) \
    if (uMsg == WM_NCPAINT) \
{ \
    SetMsgHandled(TRUE); \
    func((IRenderTarget *)wParam); \
    lResult = 0; \
    if(IsMsgHandled()) \
    return TRUE; \
}

// void OnSetFont(IFont *pFont, BOOL bRedraw)
#define MSG_WM_SETFONT_EX(func) \
    if (uMsg == WM_SETFONT) \
    { \
    SetMsgHandled(TRUE); \
    func((IFont*)wParam, (BOOL)LOWORD(lParam)); \
    lResult = 0; \
    if(IsMsgHandled()) \
    return TRUE; \
    }

// void OnSetFocus()
#define MSG_WM_SETFOCUS_EX(func) \
    if (uMsg == WM_SETFOCUS) \
{ \
    SetMsgHandled(TRUE); \
    func(); \
    lResult = 0; \
    if(IsMsgHandled()) \
    return TRUE; \
}

// void OnKillFocus()
#define MSG_WM_KILLFOCUS_EX(func) \
    if (uMsg == WM_KILLFOCUS) \
{ \
    SetMsgHandled(TRUE); \
    func(); \
    lResult = 0; \
    if(IsMsgHandled()) \
    return TRUE; \
}

// void OnNcMouseHover(int nFlag,CPoint pt)
#define MSG_WM_NCMOUSEHOVER(func) \
    if(uMsg==WM_NCMOUSEHOVER)\
{\
    SetMsgHandled(TRUE); \
    func(wParam,CPoint(GET_X_LPARAM(lParam),GET_Y_LPARAM(lParam))); \
    lResult = 0; \
    if(IsMsgHandled()) \
    return TRUE; \
}

// void OnNcMouseLeave()
#define MSG_WM_NCMOUSELEAVE(func) \
    if(uMsg==WM_NCMOUSELEAVE)\
{\
    SetMsgHandled(TRUE); \
    func(); \
    lResult = 0; \
    if(IsMsgHandled()) \
    return TRUE; \
}


// void OnTimer(char cTimerID)
#define MSG_WM_TIMER_EX(func) \
    if (uMsg == WM_TIMER) \
{ \
    SetMsgHandled(TRUE); \
    func((char)wParam); \
    lResult = 0; \
    if(IsMsgHandled()) \
    return TRUE; \
}

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

通常情况下，自定义控件还需要处理一些自定义的属性，这时我需要还需要增加一个属性映射表（如`SGifPlayer`）：

完成上面几步，一个自定义控件基本上就完成了。

可能还需要实现几个修改基类行为的虚函数。