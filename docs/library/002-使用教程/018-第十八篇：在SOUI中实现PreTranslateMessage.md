原文链接：[《第十八篇：在SOUI中实现PreTranslateMessage》](http://www.cnblogs.com/setoutsoft/p/4127333.html)

在MFC中，通常可以通过重载CWnd::PreTranslateMessage这样一个虚函数来实现对一些窗口消息的预处理。多用于tooltip的显示控制。

在SOUI中也实现了类似的机制。

要在SOUI中实现PreTranslateMessage，我们首先需要实现一个接口：

```c++
struct IMessageFilter
{
    virtual BOOL PreTranslateMessage(MSG* pMsg) = 0;
};
```

可以看出，实现这个接口和在MFC中重载PreTranslateMessage是相同的道理。

和MFC中只需要重载这个接口不同，在SOUI中，除了需要实现IMessageFilter外，还需要向当前的MessageLoop注册该IMessageFilter。

```c++
class SOUI_EXP SMessageLoop
    {
    public:
        SArray<IMessageFilter*> m_aMsgFilter;
                
        // Message filter operations
        BOOL AddMessageFilter(IMessageFilter* pMessageFilter);

        BOOL RemoveMessageFilter(IMessageFilter* pMessageFilter);
        //...
    };
```

上面是SMessageLoop两个和IMessageFilter相关的方法。

SMessageLoop::AddMessageFilter向当前的message loop注册一个IMessageFilter；
SMessageLoop::RemoveMessageFilter则向当前的message loop注销一个IMessageFilter

剩下的问题就是如何获得当前的MessageLoop了。

在SHostWnd 或者SHostDialog中可以调用SHostWnd::GetMsgLoop()方法获得。

在SWindow中，则可以调用SWindow::GetContainer()->GetMsgLoop()获得。

使用示例可以参考SDropDownWnd的实现。

```c++
class SOUI_EXP SDropDownWnd : public SHostWnd, public IMessageFilter
{
//...
};
```