原文链接：[《第十五篇：在SOUI中消息通讯》](http://www.cnblogs.com/setoutsoft/p/4100753.html)

SOUI是一套基于Win32 SDK的窗口开发的一套DirectUI框架。在SOUI中除了有真窗口使用窗口消息通讯机制外，还有SOUI控件之间的通讯，及控件的事件处理等。

### 1.真窗口消息通讯
因此可以使用`::SendMessage`这个API来与宿主窗口通讯。在任意一个地方只要获取到了SOUI的宿主窗口句柄就可以向该窗口发消息。

发消息以后可以在主界面的真窗口的消息映射表中响应各种自定义消息（如下）：

```c++
#define WM_MYMSG (WM_USER+100)
    LRESULT OnMyMsg(UINT uMsg,WPARAM wp,LPARAM lp,BOOL & bHandled)
    {
        return 0;
    }
    //HOST消息及响应函数映射表
    BEGIN_MSG_MAP_EX(CMainDlg)
        MESSAGE_HANDLER(WM_MYMSG,OnMyMsg)
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

注意上面代码的红色部分。有WTL开发经验的朋友应该已经看出来了，SOUI处理真窗口消息的形式和WTL完全一样。

### 2.SOUI控件通讯
我们知道，在win32编译中，要与一个控件（窗口）通讯能用`SendMessage(PostMessage)`发送一个消息给目标窗口，目标窗口收到后进行处理。那么问题来了，如何向一个SOUI窗口发消息？

SOUI的窗口类和MFC的窗口类很像，和MFC使用`SendMessage(PostMessage）`发消息类似，在SOUI中也可以使用`SWindow::SSendMessage`来向目标窗口发送一个消息来通讯，但不支持`PostMessage`，目标窗口在SOUI窗口的消息映射表中响应发送过来的消息。下面是一个内置控件STabCtrl的消息映射表：

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

和真窗口的映射表使用WTL的映射宏不一样，SOUI窗口的映射宏使用`SOUI_MSG_MAP_BEGIN` 和`SOUI_MSG_MAP_END`来构造消息处理函数，但是映射表中的消息映射项基本和WTL的映射形式是一样的（注意个别消息是经过重定义的，典型的如WM_PAINT消息，在SOUI中需要使用`MSG_WM_PAINT_EX`来处理）。

### 3.SOUI的事件机制
此外SOUI中控件要发出事件交给应用层处理使用的是一套事件机制。

每一个事件有对应一个`EventArg`类，事件在控件中使用`FireEvent`启动事件路由，应用程序可以在事件响应映射表中对各种事件统一处理，也可以使用`subscribeEvent`来直接订阅特定SOUI窗口的一个事件，直接将事件与事件处理函数关联。这一部分请参考前面相关章节。