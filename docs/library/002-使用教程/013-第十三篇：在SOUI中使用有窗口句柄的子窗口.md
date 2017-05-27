原文链接：[《第十三篇：在SOUI中使用有窗口句柄的子窗口》](http://www.cnblogs.com/setoutsoft/p/4001039.html)

### 前言：
无论一个DirectUI系统提供的DUI控件多么丰富，总会有些情况下用户需要在DUI窗口上放置有窗口句柄的子窗口。

为了和无窗口句柄的子窗口相区别，这里将有窗口句柄的子窗口称之为真窗口。

每一个使用SOUI创建的界面都是从SHostWnd派生出来的。SHostWnd本身就是一个有窗口句柄的真窗口。

因此和一般的win32编程一样，用户可以简单的自己以`SHostWnd.m_hWnd`为父窗口创建各种真子窗口。然后和win32一样，响应resize等消息自己管理子窗口的位置及显示。

很显然，这样处理将不能有效的利用SOUI提供的强大的布局及子窗口管理功能。

为了能够更有效的管理真窗口，在SOUI系统中提供了一个控件：SRealWnd。

SRealWnd派生自SWindow，因此它能够实现和SWindow一样的布局功能，并被SOUI系统管理窗口的各种状态：如size,visible等。

要使用SReaWnd来管理子窗口，我们首先需要实现一个接口：IRealWndHandler

### IRealWndHandler的定义：
```c++
/** 
    * @struct     IRealWndHandler
    * @brief     
    *
    * Describe   
    */
    struct IRealWndHandler : public IObjRef
    {
        /**
        * SRealWnd::OnRealWndCreate
        * @brief    窗口创建
        * @param    SRealWnd *pRealWnd -- 窗口指针
        *
        * Describe  窗口创建
        */    
        virtual HWND OnRealWndCreate(SRealWnd *pRealWnd)=NULL;

        /**
        * SRealWnd::OnRealWndDestroy
        * @brief    销毁窗口
        * @param    SRealWnd *pRealWnd -- 窗口指针
        *
        * Describe  销毁窗口
        */
        virtual void OnRealWndDestroy(SRealWnd *pRealWnd)=NULL;

        /**
        * SRealWnd::OnRealWndInit
        * @brief    初始化窗口
        * @param    SRealWnd *pRealWnd -- 窗口指针
        * @return   BOOL -- FALSE:交由系统处理，TRUE:用户处理
        *
        * Describe  初始化窗口
        */
        virtual BOOL OnRealWndInit(SRealWnd *pRealWnd)=NULL;

        /**
        * SRealWnd::OnRealWndSize
        * @brief    调整窗口大小
        * @param    SRealWnd *pRealWnd -- 窗口指针
        * @return   BOOL -- FALSE：交由SOUI处理; TRUE:用户管理窗口的移动
        *
        * Describe  调整窗口大小
        */
        virtual BOOL OnRealWndSize(SRealWnd *pRealWnd)=NULL;
    };
```

可以看到这里一共有4个接口，其中OnRealWndInit是OnRealWndSize为真窗口初始化及位置调整的回调，一般可以不处理，其它2个接口则是管理真窗口的创建及销毁，因此必须有实现。

### 接口实现示例：
真窗口的具体使用方法可以参考SOUI代码中samples目录下的mfc.demo。

这里把代码实现帖出来：

**SouiRealWndHandler.h**

```c++
#pragma once

#include <unknown/obj-ref-impl.hpp>

namespace SOUI
{
    class CSouiRealWndHandler :public TObjRefImpl2<IRealWndHandler,CSouiRealWndHandler>
    {
    public:
        CSouiRealWndHandler(void);
        ~CSouiRealWndHandler(void);

        /**
         * SRealWnd::OnRealWndCreate
         * @brief    创建真窗口
         * @param    SRealWnd * pRealWnd --  窗口指针
         * @return   HWND -- 创建出来的真窗口句柄
         * Describe  
         */    
        virtual HWND OnRealWndCreate(SRealWnd *pRealWnd);

        /**
        * SRealWnd::OnRealWndDestroy
        * @brief    销毁窗口
        * @param    SRealWnd *pRealWnd -- 窗口指针
        *
        * Describe  销毁窗口
        */
        virtual void OnRealWndDestroy(SRealWnd *pRealWnd);

        /**
        * SRealWnd::OnRealWndInit
        * @brief    初始化窗口
        * @param    SRealWnd *pRealWnd -- 窗口指针
        *
        * Describe  初始化窗口
        */
        virtual BOOL OnRealWndInit(SRealWnd *pRealWnd);

        /**
        * SRealWnd::OnRealWndSize
        * @brief    调整窗口大小
        * @param    SRealWnd *pRealWnd -- 窗口指针
        * @return   BOOL -- TRUE:用户管理窗口的移动；FALSE：交由SOUI自己管理。
        * Describe  调整窗口大小, 从pRealWnd中获得窗口位置。
        */
        virtual BOOL OnRealWndSize(SRealWnd *pRealWnd);
    };

}
```

**SouiRealWndHandler.cpp**:

```c++
#include "StdAfx.h"
#include "SouiRealWndHandler.h"

namespace SOUI
{
    CSouiRealWndHandler::CSouiRealWndHandler(void)
    {
    }

    CSouiRealWndHandler::~CSouiRealWndHandler(void)
    {
    }

    HWND CSouiRealWndHandler::OnRealWndCreate( SRealWnd *pRealWnd )
    {
        const SRealWndParam &param=pRealWnd->GetRealWndParam();
        if(param.m_strClassName==_T("button"))
        {//只实现了button的创建
            //分配一个MFC CButton对象
            CButton *pbtn=new CButton;
            //创建CButton窗口,注意使用pRealWnd->GetContainer()->GetHostHwnd()作为CButton的父窗口
            //把pRealWnd->GetID()作为真窗口的ID
            pbtn->Create(param.m_strWindowName,WS_CHILD|WS_VISIBLE|BS_PUSHBUTTON,::CRect(0,0,0,0),CWnd::FromHandle(pRealWnd->GetContainer()->GetHostHwnd()),pRealWnd->GetID());
            //把pbtn的指针放到SRealWnd的Data中保存，以便在窗口destroy时释放pbtn对象。
            pRealWnd->SetData(pbtn);
            //返回成功创建后的窗口句柄
            return pbtn->m_hWnd;
        }else
        {
            return 0;
        }
    }

    void CSouiRealWndHandler::OnRealWndDestroy( SRealWnd *pRealWnd )
    {
        const SRealWndParam &param=pRealWnd->GetRealWndParam();
        if(param.m_strClassName==_T("button"))
        {//销毁真窗口，释放窗口占用的内存
            CButton *pbtn=(CButton*) pRealWnd->GetData();
            if(pbtn)
            {
                pbtn->DestroyWindow();
                delete pbtn;
            }
        }
    }
    
    //不处理，返回FALSE
    BOOL CSouiRealWndHandler::OnRealWndSize( SRealWnd *pRealWnd )
    {
        return FALSE;
    }

    //不处理，返回FALSE
    BOOL CSouiRealWndHandler::OnRealWndInit( SRealWnd *pRealWnd )
    {
        return FALSE;
    }
}
```

整体上代码很简单，配上注释，应该一看就懂。

### XML配置：

```xml
<SOUI title="DUI-DEMO" width="600" height="400" appwin="0" ncRect="5,5,5,5"  resize="1" translucent="0">
  <root skin="skin.bkframe" cache="1">
    <caption pos="0,0,-0,29">
      <text pos="11,9" >%title% ver:%ver%</text>
      <imgbtn id="1" name="btn_close" skin="skin.btnclose" pos="-45,0" tip="close" animate="0"/>
    </caption>
    <window pos="0,29,-0,-0">
      <realwnd pos="10,10,-10,-10" name="mfcbtn" wndclass="button" id="100" wndname="MFC Button"/>
    </window>
  </root>
</SOUI>
```

在XML中，我们使用了一个realwnd的标签，该标签有一个重要的属性：wndclass，IRealWndHandler通过该属性来判断应该创建一个什么样的真窗口。

### 运行效果：

![](assets/002/013-1495899309000.png)

上面红框中的按钮即为使用realwnd标签创建的MFC Button。

### 真窗口的消息响应：

由于真窗口是SOUI主窗口的子窗口，因此真窗口的消息可以在SOUI主窗口的消息映射表中处理（注意：这里不是SOUI控件的事件映射表）。

如：

```c++
#pragma once

class CRealWndDlg : public SOUI::SHostDialog
{
public:
    CRealWndDlg(void);
    ~CRealWndDlg(void);

    //响应MFC.button的按下消息, nID==100为在XML中指定的realwnd的id属性。
    void OnBtnClick( UINT uNotifyCode, int nID, HWND wndCtl )
    {
        if(uNotifyCode == BN_CLICKED && nID == 100)
        {
            SOUI::SMessageBox(m_hWnd,_T("the real mfc button is clicked!"),_T("mfc.demo"),MB_OK|MB_ICONEXCLAMATION);
        }
    }

    //消息映射表
    BEGIN_MSG_MAP_EX(CMainDlg)
        MSG_WM_COMMAND(OnBtnClick)
        CHAIN_MSG_MAP(SOUI::SHostDialog)
        REFLECT_NOTIFICATIONS_EX()
    END_MSG_MAP()
};
```

### 结束语：
很显然，通过这种方式，也可以非常方便的创建出各种类型的其它窗口。

窗口创建出来后，系统就会自动管理窗口状态。

<font color=red size=5><b>最后，要记住一条：有真窗口时，SOUI主窗口不能设置translucent="1"这一属性。因为任何子窗口在半透明窗口上都不能正常显示。这一条也适用于包含IE控件的窗口。</b></font>