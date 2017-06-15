原文链接：[《SOUI与WTL》](http://www.cnblogs.com/setoutsoft/p/5100474.html)

如果你想使用SOUI最好有点WTL基础，一点点就行了。

SOUI不依赖于WTL，但是SOUI的编码风格基本和WTL一样的：SOUI抄袭了WTL的消息处理形式，SOUI的事件处理也是模仿了WTL的消息映射宏。

抄袭WTL的消息处理形式表现在两个层次：

### 1.在`SWindow`及其派生类中处理消息使用WTL基本一致的消息映射宏：

```c++
        SOUI_MSG_MAP_BEGIN()
            MSG_WM_PAINT_EX(OnPaint)
            MSG_WM_DESTROY(OnDestroy)
            MSG_WM_LBUTTONDOWN(OnLButtonDown)
            MSG_WM_MOUSEMOVE(OnMouseMove)
            MSG_WM_MOUSELEAVE(OnMouseLeave)
            MSG_WM_KEYDOWN(OnKeyDown)
        SOUI_MSG_MAP_END()
```

上面是一个SOUI控件中处理消息映射的映射表，除了`SOUI_MSG_MAP_BEGIN/END()`和WTL有点区别外，中间的宏基本是一样的。不同在于SOUI中`WM_PAINT`消息的参数不是`HDC`，而且`IRenderTarget`，所以使用SOUI扩展的映射宏：`MSG_WM_PAINT_EX`来处理。

### 2.在`CSimpleWnd`的派生类（包括`SHostWnd`, `SHostDialog`）中直接使用WTL的消息映射宏处理真窗口消息：

```c++
    //HOST消息及响应函数映射表
    BEGIN_MSG_MAP_EX(CMainDlg)
        MSG_WM_CREATE(OnCreate)
        MSG_WM_INITDIALOG(OnInitDialog)
        MSG_WM_DESTROY(OnDestory)
        MSG_WM_CLOSE(OnClose)
        MSG_WM_SIZE(OnSize)
        MSG_WM_COMMAND(OnCommand)
        MSG_WM_SHOWWINDOW(OnShowWindow)
        CHAIN_MSG_MAP(SHostWnd)
        REFLECT_NOTIFICATIONS_EX()
    END_MSG_MAP()
```

由于`SWindow`的消息来自`SHostWnd`的消息分发，如果在`SHostWnd`或者`SHostDialog`的派生类中处理了一个消息，如果没有将该消息交给SHostWnd继续处理，可能导致SOUI不能正常工作。

没有WTL使用经历的朋友可能想知道如何将一个消息交给`SHostWnd`继续处理。当用户在自己的消息映射表中增加一个消息处理函数，而且是插入在映射表的`CHAIN_MSG_MAP(SHostWnd)`前（也应该在此之前，否则很可能就收不到消息），默认情况下会自动标志该消息已经被处理了，如此一来就不会继续交给`SHostWnd`处理，解决办法很简单，在你的消息处理函数中增加一行：

```c++
SetMsgHandled(FALSE);
```

有了这一行，你就不用担心你的消息处理影响到`SHostWnd`的处理了。

很多朋友在使用SOUI时处理自己的Timer，结果导致SOUI中的动画不动了，就是因为这个原因：SOUI内部需要处理自己的定时器消息，但它被最外层的消息映射表截断了。

基本上会上面这样一点WTL相关的知识就可以玩转SOUI了。