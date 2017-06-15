原文链接：[《在SOUI中非半透明窗口如何实现圆角窗口？》](http://www.cnblogs.com/setoutsoft/p/5088169.html)

如果SOUI的宿主窗口没有包含子窗口，直接使用窗口的半透明属性：`translucent=1`就可以解决了，整个窗口形状完全由背景图决定，可以实现完美的圆角。

然后窗口半透明时，窗口中的子窗口(非`SWindow`)就不能正常显示，所以有时候不得不使用`translucent=0`，这时窗口就成了方形。

实际上这个问题已经和SOUI没有什么关系了，你的问题变成了窗口如何做圆角，还不是在SOUI中窗口如何做圆角。

网上一搜索一大堆，可惜经常有人要问。

给窗口做圆角或者异形好像只有一个办法：`SetWindowRgn`，自己创建一个`HRGN`，再调用这个API就可以了，关键问题是在窗口大小变化时注意重新设置。

好人做到底，这里帖一份专业做圆角的代码：

```c++
template <class T>
class CWHRoundRectFrameHelper
{
protected:

    SIZE m_sizeWnd;

    void OnSize(UINT nType, CSize size)
    {
        T *pT = static_cast<T*>(this);

        if (nType == SIZE_MINIMIZED)
            return;

        if (size == m_sizeWnd)
            return;

        CRect rcWindow, rcClient;
        CRgn rgnWindow, rgnMinus, rgnAdd;

        pT->CSimpleWnd::GetWindowRect(rcWindow);
        pT->CSimpleWnd::GetClientRect(rcClient);
        pT->CSimpleWnd::ClientToScreen(rcClient);

        rcClient.OffsetRect(- rcWindow.TopLeft());

        rgnWindow.CreateRectRgn(rcClient.left, rcClient.top + 2, rcClient.right, rcClient.bottom - 2);
        rgnAdd.CreateRectRgn(rcClient.left + 2, rcClient.top, rcClient.right - 2, rcClient.top + 1);
        rgnWindow.CombineRgn(rgnAdd, RGN_OR);
        rgnAdd.OffsetRgn(0, rcClient.Height() - 1);
        rgnWindow.CombineRgn(rgnAdd, RGN_OR);
        rgnAdd.SetRectRgn(rcClient.left + 1, rcClient.top + 1, rcClient.right - 1, rcClient.top + 2);
        rgnWindow.CombineRgn(rgnAdd, RGN_OR);
        rgnAdd.OffsetRgn(0, rcClient.Height() - 3);
        rgnWindow.CombineRgn(rgnAdd, RGN_OR);
        pT->CSimpleWnd::SetWindowRgn(rgnWindow);
        pT->SetMsgHandled(FALSE);
        m_sizeWnd = size;
    }

public:

    BOOL ProcessWindowMessage(HWND hWnd, UINT uMsg, WPARAM wParam, LPARAM lParam, LRESULT& lResult, DWORD dwMsgMapID = 0)
    {
        BOOL bHandled = TRUE;

        switch(dwMsgMapID)
        {
        case 0:
            if (uMsg == WM_SIZE)
            {
                OnSize((UINT)wParam, _WTYPES_NS::CSize(GET_X_LPARAM(lParam), GET_Y_LPARAM(lParam)));
                lResult = 0;
            }
            break;
        }
        return FALSE;
    }
};
```

这是一个模板类，下面给一个SOUI中使用的示例代码：

```c++
class CFuckDialog : public SHostDialog , public CWHRoundRectFrameHelper<CFuckDialog>
{
//xxxxx
    //HOST消息及响应函数映射表
    BEGIN_MSG_MAP_EX(CMainDlg)
        CHAIN_MSG_MAP(CWHRoundRectFrameHelper<CFuckDialog >)//重要
        CHAIN_MSG_MAP(SHostDialog)
        REFLECT_NOTIFICATIONS_EX()
    END_MSG_MAP()
//xxxx
};
```

这样你的`Dialog`就有圆角了。