原文链接：[《第三篇：用SOUI能做什么？》](http://www.cnblogs.com/setoutsoft/p/3913729.html)

### SOUI-DEMO界面预览
在回答SOUI能做什么之前，先看看SVN中demo工程的界面截图：

![](assets/002/003-1495814105000.png)

![](assets/002/003-1495814114000.png)

![](assets/002/003-1495814123000.png)

![](assets/002/003-1495814132000.png)

![](assets/002/003-1495814143000.png)

![](assets/002/003-1495814149000.png)

![](assets/002/003-1495814157000.png)

使用SOUI实现上面的界面主要的工作全在配置几个XML文件，基本不需要写C++代码。（如何配置XML布局将在后续文章中讲解）

### 从零开始生成一个使用SOUI的应用程序
以SOUI的demo为例，我们看在SOUI中如何一步一步实现一个应用程序。

首先使用Win32应用程序向导生成一个空项目。

新建一个如`demo.cpp`文件，定义一个`_tWinMain`函数。

```c++
int WINAPI _tWinMain(HINSTANCE hInstance, HINSTANCE /*hPrevInstance*/, LPTSTR /*lpstrCmdLine*/, int /*nCmdShow*/)
{
　　return 0;
}
```

在项目的包含目录中包含`$(SOUIPATH)\soui\include;$(SOUIPATH)\utilities\include；`这两个目录。同时在库依赖中增加`soui.lib utilities.lib`。

$(SOUIPATH)是从SVN签出的trunk的根目录，如果安装了soui下的应用程序向导会自动为系统增加这个环境变量。

做好上述准备工作后在工程目录下建立一个如uires的目录，用来存放程序中用到的资源文件，包括布局使用的图片及XML布局文件。

在该目录下应该至少有一个`uires.idx`文件。`uires.idx`是一个XML文件，它定义程序中用到的所有其它资源的类型及名称。

demo中使用的`uires.xml`如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<resource>
  <UIDEF>
    <file name="xml_init" path="xml\init.xml"  />
  </UIDEF>
  <ICON>
    <file name="LOGO"  path="image\img_logo.ico" />
  </ICON>
  <CURSOR>
    <file name="ANI_ARROW"  path="image\021.ani" />
    <file name="CUR_TST" path="image\camera_capture.cur"/>
  </CURSOR>
  <LAYOUT>
    <file name="maindlg" path="xml\dlg_main.xml"  />
    <file name="menu_test" path="xml\menu_test.xml"  />
    <file name="page_layout" path="xml\page_layout.xml"  />
    <file name="page_treebox" path="xml\page_treebox.xml"  />
    <file name="page_treectrl" path="xml\page_treectrl.xml"  />
    <file name="page_misc" path="xml\page_misc.xml"  />
    <file name="page_webkit" path="xml\page_webkit.xml"  />
    <file name="page_about" path="xml\page_about.xml"  />
  </LAYOUT>
  <IMGX>
    <file name="png_page_icons"   path="image\page_icons.png" />
    <file name="png_small_icons"  path="image\small_icons.png" />

    <file name="webbtn_back"  path="image\webbtn_back.png" />
    <file name="webbtn_forward"  path="image\webbtn_forward.png" />
    <file name="webbtn_refresh"  path="image\webbtn_refresh.png" />
    
    <file name="png_treeicon"             path="image\TreeIcon.png"/>
    <file name="png_menu_border"  path="image\menuborder.png" />

    <file name="png_vscroll"  path="image\vscrollbar.png" />
    
    <file name="png_tab_left"  path="image\tab_left.png" />
    <file name="png_tab_left_splitter"  path="image\tab_left_splitter.png" />
    <file name="png_tab_main"  path="image\tab_main.png" />
    <file name="btn_menu" path="image\btn_menu.png"  />
  </IMGX>
  <GIF>
    <file name="gif_horse"    path="image\horse.gif"/>
    <file name="gif_penguin"    path="image\penguin.gif"/>
  </GIF>
  <rtf>
    <file name="rtf_test"    path="rtf\RTF测试.rtf"/>
  </rtf>
  <script>
    <file name="lua_test" path="lua\test.lua"/>
  </script>
  <translator>
    <file name="lang_cn" path="translation files\lang_cn.xml"/>
  </translator>
</resource>
```

如上所示，该XML有一个resource的根节点，下面可以是任意定义的类型(ICON, BITMAP，CURSOR除外，它们是预定义的类型，不能修改类型名）。

每个类型下面定义有file元素，元素中有两个属性：name 及 path。

name即资源的名称，path即资源的路径。所有资源建议采用相对路径，即相对于`uires.idx`文件的路径。

在程序中通过type及name来引用资源。

下面开始填空：

```c++
int WINAPI _tWinMain(HINSTANCE hInstance, HINSTANCE /*hPrevInstance*/, LPTSTR /*lpstrCmdLine*/, int /*nCmdShow*/)
{
    //必须要调用OleInitialize来初始化运行环境
    HRESULT hRes = OleInitialize(NULL);
    SASSERT(SUCCEEDED(hRes));
    
    int nRet = 0; 

    //定义一组组件加载辅助对象
    //SComLoader实现从DLL的指定函数创建符号SOUI要求的类COM组件。
    SComLoader imgDecLoader;
    SComLoader renderLoader;
    SComLoader transLoader;
    SComLoader scriptLoader;
    SComLoader zipResLoader;
    
    //将程序的运行路径修改到demo所在的目录
    TCHAR szCurrentDir[MAX_PATH]={0};
    GetModuleFileName( NULL, szCurrentDir, sizeof(szCurrentDir) );
    LPTSTR lpInsertPos = _tcsrchr( szCurrentDir, _T('\\') );
    _tcscpy(lpInsertPos,_T("\\..\\demo"));
    SetCurrentDirectory(szCurrentDir);
    
    {
        //定义一组类SOUI系统中使用的类COM组件
        //CAutoRefPtr是一个SOUI系统中使用的智能指针类
        CAutoRefPtr<IImgDecoderFactory> pImgDecoderFactory; //图片解码器，由imagedecoder-wid.dll模块提供
        CAutoRefPtr<IRenderFactory> pRenderFactory;         //UI渲染模块，由render-gdi.dll或者render-skia.dll提供
        CAutoRefPtr<ITranslatorMgr> trans;                  //多语言翻译模块，由translator.dll提供
        CAutoRefPtr<IScriptModule> pScriptLua;              //lua脚本模块，由scriptmodule-lua.dll提供
        
        BOOL bLoaded=FALSE;
        int nType=MessageBox(GetActiveWindow(),_T("选择渲染类型：\n[yes]: Skia\n[no]:GDI\n[cancel]:Quit"),_T("select a render"),MB_ICONQUESTION|MB_YESNOCANCEL);
        if(nType == IDCANCEL) return -1;
        //从各组件中显示创建上述组件对象
        bLoaded=renderLoader.CreateInstance(nType==IDYES?COM_RENDER_SKIA:COM_RENDER_GDI,(IObjRef**)&pRenderFactory);
        SASSERT_FMT(bLoaded,_T("load module [%s] failed!"),nType==IDYES?COM_RENDER_SKIA:COM_RENDER_GDI);
        bLoaded=imgDecLoader.CreateInstance(COM_IMGDECODER,(IObjRef**)&pImgDecoderFactory);
        SASSERT_FMT(bLoaded,_T("load module [%s] failed!"),COM_IMGDECODER);
        bLoaded=transLoader.CreateInstance(COM_TRANSLATOR,(IObjRef**)&trans);
        SASSERT_FMT(bLoaded,_T("load module [%s] failed!"),COM_TRANSLATOR);
        //为渲染模块设置它需要引用的图片解码模块
        pRenderFactory->SetImgDecoderFactory(pImgDecoderFactory);
        //定义一个唯一的SApplication对象，SApplication管理整个应用程序的资源
        SApplication *theApp=new SApplication(pRenderFactory,hInstance);

        //定义一人个资源提供对象,SOUI系统中实现了3种资源加载方式，分别是从文件加载，从EXE的资源加载及从ZIP压缩包加载
        CAutoRefPtr<IResProvider>   pResProvider;
#if (RES_TYPE == 0)//从文件加载
        CreateResProvider(RES_FILE,(IObjRef**)&pResProvider);
        if(!pResProvider->Init((LPARAM)_T("uires"),0))
        {
            SASSERT(0);
            return 1;
        }
#elif (RES_TYPE==1)//从EXE资源加载
        CreateResProvider(RES_PE,(IObjRef**)&pResProvider);
        pResProvider->Init((WPARAM)hInstance,0);
#elif (RES_TYPE==2)//从ZIP包加载
        bLoaded=zipResLoader.CreateInstance(COM_ZIPRESPROVIDER,(IObjRef**)&pResProvider);
        SASSERT(bLoaded);
        ZIPRES_PARAM param;
        param.ZipFile(pRenderFactory, _T("uires.zip"));
        bLoaded = pResProvider->Init((WPARAM)&param,0);
        SASSERT(bLoaded);
#endif
        //将创建的IResProvider交给SApplication对象
        theApp->AddResProvider(pResProvider);

        if(trans)
        {//加载语言翻译包
            theApp->SetTranslator(trans);
            pugi::xml_document xmlLang;
            if(theApp->LoadXmlDocment(xmlLang,_T("lang_cn"),_T("translator")))
            {
                CAutoRefPtr<ITranslator> langCN;
                trans->CreateTranslator(&langCN);
                langCN->Load(&xmlLang.child(L"language"),1);//1=LD_XML
                trans->InstallTranslator(langCN);
            }
        }
#ifdef DLL_SOUI
        //加载LUA脚本模块，注意，脚本模块只有在SOUI内核是以DLL方式编译时才能使用。
        bLoaded=scriptLoader.CreateInstance(COM_SCRIPT_LUA,(IObjRef**)&pScriptLua);
        SASSERT_FMT(bLoaded,_T("load module [%s] failed!"),COM_SCRIPT_LUA);
        if(pScriptLua)
        {
            theApp->SetScriptModule(pScriptLua);
            size_t sz=pResProvider->GetRawBufferSize(_T("script"),_T("lua_test"));
            if(sz)
            {
                CMyBuffer<char> lua;
                lua.Allocate(sz);
                pResProvider->GetRawBuffer(_T("script"),_T("lua_test"),lua,sz);
                pScriptLua->executeScriptBuffer(lua,sz);
            }
        }
#endif//DLL_SOUI

        //向SApplication系统中注册由外部扩展的控件及SkinObj类
        SWkeLoader wkeLoader;
        if(wkeLoader.Init(_T("wke.dll")))        
        {
            theApp->RegisterWndFactory(TplSWindowFactory<SWkeWebkit>());//注册WKE浏览器
        }
        theApp->RegisterWndFactory(TplSWindowFactory<SGifPlayer>());//注册GIFPlayer
        theApp->RegisterSkinFactory(TplSkinFactory<SSkinGif>());//注册SkinGif
        SSkinGif::Gdiplus_Startup();
        
        //加载系统资源
        HMODULE hSysResource=LoadLibrary(SYS_NAMED_RESOURCE);
        if(hSysResource)
        {
            CAutoRefPtr<IResProvider> sysSesProvider;
            CreateResProvider(RES_PE,(IObjRef**)&sysSesProvider);
            sysSesProvider->Init((WPARAM)hSysResource,0);
            theApp->LoadSystemNamedResource(sysSesProvider);
        }

        //加载全局资源描述XML
        theApp->Init(_T("xml_init")); 

        {
            //创建并显示使用SOUI布局应用程序窗口,为了保存窗口对象的析构先于其它对象，把它们缩进一层。
            CMainDlg dlgMain;  
            dlgMain.Create(GetActiveWindow(),0,0,800,600);
            dlgMain.GetNative()->SendMessage(WM_INITDIALOG);
            dlgMain.CenterWindow();
            dlgMain.ShowWindow(SW_SHOWNORMAL);
            nRet=theApp->Run(dlgMain.m_hWnd);
        }
        
        //应用程序退出
        delete theApp; 
        SSkinGif::Gdiplus_Shutdown();

    }

    OleUninitialize();
    return nRet;
}
```

main中用到一个类CMainDlg，该类是demo的主窗口，前面提供的界面截图都是由该类渲染出来。

下面我们看一下CMainDlg的实现：

`maindlg.h`

```c++
/**
* Copyright (C) 2014-2050 
* All rights reserved.
* 
* @file       MainDlg.h
* @brief      
* @version    v1.0      
* @author     SOUI group   
* @date       2014/08/15
* 
* Describe    主窗口实现
*/

#pragma once

/**
* @class      CMainDlg
* @brief      主窗口实现
* 
* Describe    非模式窗口从SHostWnd派生，模式窗口从SHostDialog派生
*/
class CMainDlg : public SHostWnd
{
public:

    /**
     * CMainDlg
     * @brief    构造函数
     * Describe  使用uires.idx中定义的maindlg对应的xml布局创建UI
     */    
    CMainDlg() : SHostWnd(_T("maindlg")),m_bLayoutInited(FALSE)
    {
    } 

protected:
    //////////////////////////////////////////////////////////////////////////
    //  Window消息响应函数
    LRESULT OnInitDialog(HWND hWnd, LPARAM lParam);
    void OnDestory();

    void OnClose()
    {
        AnimateHostWindow(200,AW_CENTER|AW_HIDE);
        PostMessage(WM_QUIT);
    }
    void OnMaximize()
    {
        GetNative()->SendMessage(WM_SYSCOMMAND,SC_MAXIMIZE);
    }
    void OnRestore()
    {
        GetNative()->SendMessage(WM_SYSCOMMAND,SC_RESTORE);
    }
    void OnMinimize()
    {
        GetNative()->SendMessage(WM_SYSCOMMAND,SC_MINIMIZE);
    }

    void OnSize(UINT nType, CSize size)
    {
        SetMsgHandled(FALSE);   //这一行很重要，保证消息继续传递给SHostWnd处理，当然也可以用SHostWnd::OnSize(nType,size);代替，但是这里使用的方法更简单，通用
        if(!m_bLayoutInited) return;
        if(nType==SIZE_MAXIMIZED)
        {
            FindChildByID(3)->SetVisible(TRUE);
            FindChildByID(2)->SetVisible(FALSE);
        }else if(nType==SIZE_RESTORED)
        {
            FindChildByID(3)->SetVisible(FALSE);
            FindChildByID(2)->SetVisible(TRUE);
        }
    }
    
    int OnCreate(LPCREATESTRUCT lpCreateStruct);
    void OnShowWindow(BOOL bShow, UINT nStatus);

    //DUI菜单响应函数
    void OnCommand(UINT uNotifyCode, int nID, HWND wndCtl);
        
protected:
    //////////////////////////////////////////////////////////////////////////
    // SOUI事件处理函数
    //演示屏蔽指定edit控件的右键菜单
    BOOL OnEditMenu(CPoint pt)
    {
        return TRUE;
    }

    //按钮控件的响应
    void OnBtnSelectGIF();
    void OnBtnMenu();
    void OnBtnInsertGif2RE();
    void OnBtnHideTest();
    void OnBtnMsgBox();

    void OnBtnWebkitGo();
    void OnBtnWebkitBackward();
    void OnBtnWebkitForeward();
    void OnBtnWebkitRefresh();

    //演示如何使用subscribeEvent来不使用事件映射表实现事件响应
    bool OnListHeaderClick(EventArgs *pEvt);

    //UI控件的事件及响应函数映射表
    EVENT_MAP_BEGIN()
        EVENT_ID_COMMAND(1, OnClose)
        EVENT_ID_COMMAND(2, OnMaximize)
        EVENT_ID_COMMAND(3, OnRestore)
        EVENT_ID_COMMAND(5, OnMinimize)
        EVENT_NAME_CONTEXTMENU(L"edit_1140",OnEditMenu)
        EVENT_NAME_COMMAND(L"btn_msgbox",OnBtnMsgBox)
        EVENT_NAME_COMMAND(L"btnSelectGif",OnBtnSelectGIF)
        EVENT_NAME_COMMAND(L"btn_menu",OnBtnMenu)
        EVENT_NAME_COMMAND(L"btn_webkit_go",OnBtnWebkitGo)
        EVENT_NAME_COMMAND(L"btn_webkit_back",OnBtnWebkitBackward)
        EVENT_NAME_COMMAND(L"btn_webkit_fore",OnBtnWebkitForeward)
        EVENT_NAME_COMMAND(L"btn_webkit_refresh",OnBtnWebkitRefresh)
        EVENT_NAME_COMMAND(L"btn_hidetst",OnBtnHideTest)
        EVENT_NAME_COMMAND(L"btn_insert_gif",OnBtnInsertGif2RE)
    EVENT_MAP_END()    

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

protected:
    //////////////////////////////////////////////////////////////////////////
    //  辅助函数
    void InitListCtrl();
private:
    BOOL            m_bLayoutInited;/**<UI完成布局标志 */
};
```

`maindlg.cpp`

```c++
// MainDlg.cpp : implementation of the CMainDlg class
//
/////////////////////////////////////////////////////////////////////////////

#include "stdafx.h"
#include "MainDlg.h"
#include "helper/SMenu.h"
#include "../controls.extend/FileHelper.h"

#include <dwmapi.h>
#pragma comment(lib,"dwmapi.lib")



int CMainDlg::OnCreate( LPCREATESTRUCT lpCreateStruct )
{
//     MARGINS mar = {5,5,30,5};
//     DwmExtendFrameIntoClientArea ( m_hWnd, &mar );//打开这里可以启用Aero效果
    SetMsgHandled(FALSE);
    return 0;
}

void CMainDlg::OnShowWindow( BOOL bShow, UINT nStatus )
{
    if(bShow)
    {
         AnimateHostWindow(200,AW_CENTER);
    }
}

struct student{
    TCHAR szName[100];
    TCHAR szSex[10];
    int age;
    int score;
};

//init listctrl
void CMainDlg::InitListCtrl()
{
    SListCtrl *pList=FindChildByName2<SListCtrl>(L"lc_test");
    if(pList)
    {
        SWindow *pHeader=pList->GetWindow(GSW_FIRSTCHILD);
        pHeader->GetEventSet()->subscribeEvent(EVT_HEADER_CLICK,Subscriber(&CMainDlg::OnListHeaderClick,this));

        TCHAR szSex[][5]={_T("男"),_T("女"),_T("人妖")};
        for(int i=0;i<100;i++)
        {
            student *pst=new student;
            _stprintf(pst->szName,_T("学生_%d"),i+1);
            _tcscpy(pst->szSex,szSex[rand()%3]);
            pst->age=rand()%30;
            pst->score=rand()%60+40;

            int iItem=pList->InsertItem(i,pst->szName);
            pList->SetItemData(iItem,(DWORD)pst);
            pList->SetSubItemText(iItem,1,pst->szSex);
            TCHAR szBuf[10];
            _stprintf(szBuf,_T("%d"),pst->age);
            pList->SetSubItemText(iItem,2,szBuf);
            _stprintf(szBuf,_T("%d"),pst->score);
            pList->SetSubItemText(iItem,3,szBuf);
        }
    }
}

int funCmpare(void* pCtx,const void *p1,const void *p2)
{
    int iCol=*(int*)pCtx;

    const DXLVITEM *plv1=(const DXLVITEM*)p1;
    const DXLVITEM *plv2=(const DXLVITEM*)p2;

    const student *pst1=(const student *)plv1->dwData;
    const student *pst2=(const student *)plv2->dwData;

    switch(iCol)
    {
    case 0://name
        return _tcscmp(pst1->szName,pst2->szName);
    case 1://sex
        return _tcscmp(pst1->szSex,pst2->szSex);
    case 2://age
        return pst1->age-pst2->age;
    case 3://score
        return pst1->score-pst2->score;
    default:
        return 0;
    }
}

bool CMainDlg::OnListHeaderClick(EventArgs *pEvtBase)
{
    EventHeaderClick *pEvt =(EventHeaderClick*)pEvtBase;
    SHeaderCtrl *pHeader=(SHeaderCtrl*)pEvt->sender;
    SListCtrl *pList=FindChildByName2<SListCtrl>(L"lc_test");

    SHDITEM hditem;
    hditem.mask=SHDI_ORDER;
    pHeader->GetItem(pEvt->iItem,&hditem);
    pList->SortItems(funCmpare,&hditem.iOrder);
    return true;
}

void CMainDlg::OnDestory()
{
    SListCtrl *pList=FindChildByName2<SListCtrl>(L"lc_test");
    if(pList)
    {
        for(int i=0;i<pList->GetItemCount();i++)
        {
            student *pst=(student*) pList->GetItemData(i);
            delete pst;
        }
    }
    SetMsgHandled(FALSE); 
}


LRESULT CMainDlg::OnInitDialog( HWND hWnd, LPARAM lParam )
{
    m_bLayoutInited=TRUE;
    InitListCtrl();
    
#ifdef DLL_SOUI
    SWindow *pTst=FindChildByName(L"btn_tstevt");
    if(pTst) SApplication::getSingleton().GetScriptModule()->subscribeEvent(pTst,EVT_CMD,"onEvtTstClick");
#endif

    return 0;
}

void CMainDlg::OnBtnWebkitGo()
{
    SWkeWebkit *pWebkit= FindChildByName2<SWkeWebkit>(L"wke_test");
    if(pWebkit)
    {
        SEdit *pEdit=FindChildByName2<SEdit>(L"edit_url");
        SStringW strUrl=pEdit->GetWindowText();
        pWebkit->SetAttribute(L"url",strUrl,FALSE);
    }
}

void CMainDlg::OnBtnWebkitBackward()
{
    SWkeWebkit *pWebkit= FindChildByName2<SWkeWebkit>(L"wke_test");
    if(pWebkit)
    {
        pWebkit->GetWebView()->goBack();
    }
}

void CMainDlg::OnBtnWebkitForeward()
{
    SWkeWebkit *pWebkit= FindChildByName2<SWkeWebkit>(L"wke_test");
    if(pWebkit)
    {
        pWebkit->GetWebView()->goForward();
    }
}

void CMainDlg::OnBtnWebkitRefresh()
{
    SWkeWebkit *pWebkit= FindChildByName2<SWkeWebkit>(L"wke_test");
    if(pWebkit)
    {
        pWebkit->GetWebView()->reload();
    }
}

void CMainDlg::OnBtnSelectGIF()
{
    SGifPlayer *pGifPlayer = FindChildByName2<SGifPlayer>(L"giftest");
    if(pGifPlayer)
    {
        CFileDialogEx openDlg(TRUE,_T("gif"),0,6,_T("gif files(*.gif)\0*.gif\0All files (*.*)\0*.*\0\0"));
        if(openDlg.DoModal()==IDOK)
            pGifPlayer->PlayGifFile(openDlg.m_szFileName);
    }
}

void CMainDlg::OnBtnMenu()
{
    SMenu menu;
    menu.LoadMenu(_T("menu_test"),_T("LAYOUT"));
    POINT pt;
    GetCursorPos(&pt);
    menu.TrackPopupMenu(0,pt.x,pt.y,m_hWnd);
}

//演示如何响应菜单事件
void CMainDlg::OnCommand( UINT uNotifyCode, int nID, HWND wndCtl )
{
    if(uNotifyCode==0)
    {
        if(nID==6)
        {//nID==6对应menu_test定义的菜单的exit项。
            PostMessage(WM_CLOSE);
        }else if(nID==54)
        {//about SOUI
            STabCtrl *pTabCtrl = FindChildByName2<STabCtrl>(L"tab_main");
            if(pTabCtrl) pTabCtrl->SetCurSel(_T("about"));
        }
    }
}

void CMainDlg::OnBtnHideTest()
{
    SWindow * pBtn = FindChildByName(L"btn_hidetst");
    if(pBtn) pBtn->SetVisible(FALSE,TRUE);
}

#include "skinole\ImageOle.h"

void CMainDlg::OnBtnInsertGif2RE()
{
    SRichEdit *pEdit = FindChildByName2<SRichEdit>(L"re_gifhost");
    if(pEdit)
    {
        ISkinObj * pSkin = GETSKIN(L"gif_penguin");
        if(pSkin)
        {
            RichEdit_InsertSkin(pEdit,pSkin);
        }
    }
}

void CMainDlg::OnBtnMsgBox()
{
    SMessageBox(NULL,_T("this is a message box"),_T("haha"),MB_OK|MB_ICONEXCLAMATION);
    SMessageBox(NULL,_T("this message box includes two buttons"),_T("haha"),MB_YESNO|MB_ICONQUESTION);
    SMessageBox(NULL,_T("this message box includes three buttons"),NULL,MB_ABORTRETRYIGNORE);
}
```

maindlg.cpp的主要工作就是调用SHostWnd的`FindChildByName/FindChildByID`查找到SOUI的控件，然后调用控件提供的方法完成对控件的操作。

大家可能发现使用SOUI的这个main函数相对于其它程序可能要更加复杂，这是为了达到程序配置的灵活性需要付出的代价。

好在SOUI提供了应用程序向导，它会帮助你点两个按钮就生成一整套框架。

### SOUI与其它应用程序开发框架
SOUI是一个使用纯Win32 SDK开发的UI库，内核部分使用了pugixml这个第三方库作为XML解析的模块，除此之外，不再依赖其它第三方库，同时所有使用的模块都可以通过源代码编译。

SOUI提供了一整套完整的UI开发框架，不需要依赖其它的如MFC，WTL等开发框架。同时由于SOUI是纯win32的SDK开发的，它理论上也可以和任意的其它开发框架共存。（实际处理中由于SOUI中使用的一些类的命名可能和其它框架冲突，因此可能需要注意命名空间的使用。）