原文链接：[《第七篇：创建一个SOUI的Hello World》](http://www.cnblogs.com/setoutsoft/p/3928361.html)

### 从0开始一个SOUI项目

#### 1.环境配置
SOUI项目本质是一个基于Win32窗口的应用程序。因此首先我们可以从Win32窗口应用程序向导创建一个简单的Win32项目。

![](assets/002/007-1495895465000.png)

并在第3页选择“**Window应用程序**”

![](assets/002/007-1495895479000.png)

选择“完成”后生成一个Win32应用程序骨架。

项目的文件结构如下图：

![](assets/002/007-1495895495000.png)

要使用SOUI开发程序程序，首先当然是要找到从SVN获取的SOUI项目代码。假定SOUI项目保存在`%SOUIPATH%`这个环境变量指向的目录（安装了SOUI向导后会自动创建这个环境变量）。

我们需要在VS的include目录中增加两个目录：

**"$(SOUIPATH)/soui/include";"$(SOUIPATH)/utilities/include"**

如图：

![](assets/002/007-1495895510000.png)

**注：新版本还需要增加一个包含目录：$(SOUIPATH)/config**

假定你已经编译好了SOUI内核代码，默认情况下内核库会生成在`$(SOUIPATH)/bin`目录下，我们还需要在引用的库中指定两个库文件:**souid.lib,utilitiesd.lib**(以debug版本为例）。

![](assets/002/007-1495895544000.png)

设置好项目后，默认情况下还需要把编译选项中的“代码生成”从MDd修改为MTd，并把“`将wchar_t视为内置类型`”修改为“否”。（因为这是SOUI的默认编译配置）

到这里环境配置基本完成。

#### 2.准备资源
到这里准备工作还没有完，我们需要为SOUI准备一套程序资源，并把它放到项目目录下。

最基本的资源至少应该包括：

**uires.idx**：定义资源索引

**init.xml**：定义全局UI的属性，包含字体，字符串表，skin，style，objattr，参见前篇介绍。

**dlg_main.xml**:主窗口而已XML。

**init.xml**和**dlg_main.xml**的文件名不限，但是`uires.idx`的文件名是固定的。

**uires.idx**：

```xml
<resource>  
  <UIDEF>
    <file name="XML_INIT" path="xml\init.xml"  />
  </UIDEF>
  <LAYOUT>
    <file name="XML_MAINWND" path="xml\dlg_main.xml"  />
  </LAYOUT>
</resource>
```

**init.xml**:

```xml
<?xml version="1.0" encoding="utf-8"?>
<UIDEF>
  <font face="宋体" size="15"/>

  <string>
    <title value=""/>
    <ver value="1.0"/>
  </string>
  
  <skin>
  </skin>

  <style>
    <class name="normalbtn" font="" colorText="#385e8b" colorTextDisable="#91a7c0" textMode="25" cursor="hand" margin-x="0"/>
  </style>

  <objattr>
  </objattr>
</UIDEF>
```

**dlg_main.xml**:

```xml
<SOUI name="mainWindow" title="%title% ver:%ver%" width="600" height="400" appWnd="1" margin="20,5,5,5"  resizable="1" translucent="1" >
  <root skin="_skin.sys.wnd.bkgnd">
    <caption pos="0,0,-0,30">
      <text pos="11,9">%title% ver:%ver%</text>
      <imgbtn name="btn_close" skin="_skin.sys.btn.close"    pos="-45,0" tip="close" animate="1"/>
      <imgbtn name="btn_max" skin="_skin.sys.btn.maximize"  pos="-83,0" animate="1" />
      <imgbtn name="btn_restore" skin="_skin.sys.btn.restore"  pos="-83,0" show="0" animate="1" />
      <imgbtn name="btn_min" skin="_skin.sys.btn.minimize" pos="-121,0" animate="1" />
    </caption>
    <window pos="5,30,-5,-5">
      <text pos="|0,|0" pos2type="center" colorText="#ff0000">Hellow World! UI? Just so so!</text>
      <button class ="normalbtn" pos="|-50,[20,@100,@30" name="btn_msgbox">show msg box</button> 
    </window>
  </root>
</SOUI>
```

如果对UI布局不明白可以参考“第五篇”。

### 3.开始编码
要开始编写代码还需要修改一下**stdafx.h**如下：

```c++
// stdafx.h : 标准系统包含文件的包含文件，
// 或是经常使用但不常更改的
// 特定于项目的包含文件
//

#pragma once

#include "targetver.h"

#define  _CRT_SECURE_NO_WARNINGS
#define     DLL_SOUI   //SOUI是以DLL提供时需要定义这个宏

#include <souistd.h>
#include <core/SHostDialog.h>
#include <control/SMessageBox.h>
#include <control/souictrls.h>

using namespace SOUI;
```

功能就是引用几个SOUI头文件。

打开`helloworld.cpp`，删除文件中除空main函数以外的全部代码，剩下的代码如下：

```c++
// hellowworld.cpp : 定义应用程序的入口点。
//

#include "stdafx.h"
#include "hellowworld.h"


int APIENTRY _tWinMain(HINSTANCE hInstance,
                     HINSTANCE hPrevInstance,
                     LPTSTR    lpCmdLine,
                     int       nCmdShow)
{
    
    return 0;
}
```

这是一个可以编译的空项目。

### 4.开始填充Main程序框架
```c++
// helloworld.cpp : main source file
//

#include "stdafx.h"

#include <com-loader.hpp>


#ifdef _DEBUG
#define COM_IMGDECODER  _T("imgdecoder-wicd.dll")
#define COM_RENDER_GDI  _T("render-gdid.dll")
#define SYS_NAMED_RESOURCE _T("soui-sys-resourced.dll")
#else
#define COM_IMGDECODER  _T("imgdecoder-wic.dll")
#define COM_RENDER_GDI  _T("render-gdi.dll")
#define SYS_NAMED_RESOURCE _T("soui-sys-resource.dll")
#endif


int WINAPI _tWinMain(HINSTANCE hInstance, HINSTANCE /*hPrevInstance*/, LPTSTR /*lpstrCmdLine*/, int /*nCmdShow*/)
{
    HRESULT hRes = OleInitialize(NULL);
    SASSERT(SUCCEEDED(hRes));
    
    int nRet = 0; 

    SComLoader imgDecLoader;
    SComLoader renderLoader;
    SComLoader transLoader;
    //将程序的运行路径修改到项目所在目录所在的目录
    TCHAR szCurrentDir[MAX_PATH]={0};
    GetModuleFileName( NULL, szCurrentDir, sizeof(szCurrentDir) );
    LPTSTR lpInsertPos = _tcsrchr( szCurrentDir, _T('\\') );
    _tcscpy(lpInsertPos+1,_T(".."));
    SetCurrentDirectory(szCurrentDir);
        
    {

        CAutoRefPtr<SOUI::IImgDecoderFactory> pImgDecoderFactory;
        CAutoRefPtr<SOUI::IRenderFactory> pRenderFactory;
        imgDecLoader.CreateInstance(COM_IMGDECODER,(IObjRef**)&pImgDecoderFactory);
        renderLoader.CreateInstance(COM_RENDER_GDI,(IObjRef**)&pRenderFactory);

        pRenderFactory->SetImgDecoderFactory(pImgDecoderFactory);

        SApplication *theApp=new SApplication(pRenderFactory,hInstance);

        HMODULE hSysResource=LoadLibrary(SYS_NAMED_RESOURCE);
        if(hSysResource)
        {
            CAutoRefPtr<IResProvider> sysSesProvider;
            CreateResProvider(RES_PE,(IObjRef**)&sysSesProvider);
            sysSesProvider->Init((WPARAM)hSysResource,0);
            theApp->LoadSystemNamedResource(sysSesProvider);
        }
        

        CAutoRefPtr<IResProvider>   pResProvider;
        CreateResProvider(RES_FILE,(IObjRef**)&pResProvider);
        if(!pResProvider->Init((LPARAM)_T("uires"),0))
        {
            SASSERT(0);
            return 1;
        }
        theApp->AddResProvider(pResProvider);
        
　　　　 //2.x版本已经不需要下面这行。
        //theApp->Init(_T("XML_INIT")); 

        {//在这里加入主窗口运行代码
        }

        delete theApp;
    }

    OleUninitialize();
    return nRet;
}
```

将`helloworld.cpp`修改如上。

这里主要功能是配置几个SOUI需要的组件。

### 5.为主窗口布局生成一个C++类
和`MFC，WTL`等一样，SOUI可以有两种方式显示窗口：模态与非模态。

这里我们采用非模态窗口来演示。

因此我们需要从SHostWnd派生出一个C++对象：`CMainWnd`
```c++
// MainWnd.h : interface of the CMainWnd class
//
/////////////////////////////////////////////////////////////////////////////
#pragma once

class CMainWnd : public SHostWnd
{
public:
    CMainWnd() 
        : SHostWnd(_T("LAYOUT:XML_MAINWND"))//这里定义主界面需要使用的布局文件
    {
        m_bLayoutInited=FALSE;
    }

    void OnClose()
    {
        PostMessage(WM_QUIT);
    }
    void OnMaximize()
    {
        SendMessage(WM_SYSCOMMAND,SC_MAXIMIZE);
    }
    void OnRestore()
    {
        SendMessage(WM_SYSCOMMAND,SC_RESTORE);
    }
    void OnMinimize()
    {
        SendMessage(WM_SYSCOMMAND,SC_MINIMIZE);
    }

    void OnSize(UINT nType, CSize size)
    {
        SetMsgHandled(FALSE);
        if(!m_bLayoutInited) return;
        if(nType==SIZE_MAXIMIZED)
        {
            FindChildByName(L"btn_restore")->SetVisible(TRUE);
            FindChildByName(L"btn_max")->SetVisible(FALSE);
        }else if(nType==SIZE_RESTORED)
        {
            FindChildByName(L"btn_restore")->SetVisible(FALSE);
            FindChildByName(L"btn_max")->SetVisible(TRUE);
        }
    }
    void OnBtnMsgBox()
    {
        SMessageBox(NULL,_T("this is a message box"),_T("haha"),MB_OK|MB_ICONEXCLAMATION);
        SMessageBox(NULL,_T("this message box includes two buttons"),_T("haha"),MB_YESNO|MB_ICONQUESTION);
        SMessageBox(NULL,_T("this message box includes three buttons"),NULL,MB_ABORTRETRYIGNORE);
    }
    
    BOOL OnInitDialog( HWND hWnd, LPARAM lParam )
    {
        m_bLayoutInited=TRUE;

        return 0;
    }
protected:
    //按钮事件处理映射表
    EVENT_MAP_BEGIN()
        EVENT_NAME_COMMAND(L"btn_close",OnClose)
        EVENT_NAME_COMMAND(L"btn_min",OnMinimize)
        EVENT_NAME_COMMAND(L"btn_max",OnMaximize)
        EVENT_NAME_COMMAND(L"btn_restore",OnRestore)
        EVENT_NAME_COMMAND(L"btn_msgbox",OnBtnMsgBox)
    EVENT_MAP_END()    

    //窗口消息处理映射表
    BEGIN_MSG_MAP_EX(CMainWnd)
        MSG_WM_INITDIALOG(OnInitDialog)
        MSG_WM_CLOSE(OnClose)
        MSG_WM_SIZE(OnSize)
        CHAIN_MSG_MAP(SHostWnd)//注意将没有处理的消息交给基类处理
        REFLECT_NOTIFICATIONS_EX()
    END_MSG_MAP()
private:
    BOOL            m_bLayoutInited;
};
```

为了介绍方便，这个窗口类所有代码都在这个`mainwnd.h`文件里。

#### 6.再进一步填充main函数
```c++
{
            //在这里加入主窗口运行代码
            CMainWnd wndMain;  
            wndMain.Create(GetActiveWindow(),0,0,800,600);
            wndMain.SendMessage(WM_INITDIALOG);
            wndMain.CenterWindow(wndMain.m_hWnd);
            wndMain.ShowWindow(SW_SHOWNORMAL);
            nRet=theApp->Run(wndMain.m_hWnd);
            //程序结束
}
```

当然main前面还要有一行：`#include "mainwnd.h"`

大功造成!

#### 7.编译运行结果

![](assets/002/007-1495895981000.png)

### 认识SOUI应用程序向导

开发一个使用SOUI的应用程序，如果从Win32项目类型开始，即便是我对SOUI这样了如指掌，也至少要10分钟以上才能配置出一个使用SOUI的开发环境。

为了帮助程序员从简单的重复劳动解脱出来，也帮助初学都快速开始，我特别为SOUI制作了一个适用于VS各版本的应用程序向导。

使用向导可以通过简单的选择两个选项就自动完全项目配置，编译即可看到UI布局结果。

#### 1.SOUI向导的安装
在SOUI项目下有一个wizard目录，运行目录下的`wizard.setup.exe`会显示下面界面：

![](assets/002/007-1495896031000.png)

选择一个系统中安装的VS版本，点击安装就会向该版本中安装SOUI向导。

安装成功后显示：

![](assets/002/007-1495896060000.png)

注：原来DuiEngine的向导安装程序运行后设置的环境变量经常要重启系统才生效，SOUI中的版本已经经过优化，基本上不需要重启系统，但已经打开的VS肯定是没有新增加的环境变量的。

#### 2.使用向导创建SOUI应用程序
向导安装完成后，打开VS的新建项目窗口，会在“**Virtual C++**”下找到SOUI向导入口。

![](assets/002/007-1495896085000.png)

输入项目名称后点击确定打开SOUI向导页（只有一页）

![](assets/002/007-1495896102000.png)

点击完成即可开始享受最简单高效的界面编程了。

注：由于SOUI还在持续更新，上述手动创建helloWorld的代码有可能在新版本中失效。请以向导创建的代码为准！！！

下一篇介绍SOUI中控件事件响应。