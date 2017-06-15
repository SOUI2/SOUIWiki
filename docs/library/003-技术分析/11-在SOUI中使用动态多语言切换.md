原文链接：[《在SOUI中使用动态多语言切换》](http://www.cnblogs.com/setoutsoft/p/6743006.html)

动态语言切换是很多国际化产品的需求，SOUI之前的版本支持静态多语言翻译，通过在程序启动时设置好语言翻译模块，在程序中打开的UI都会自动调用该翻译模块进行文字翻译，但是不支持运行进语言切换。

最近几个网友都提到这个需求，还是决定在SOUI实现一套动态多语言切换机制。

先看看运行效果：

![](assets/003/11-1497542079000.gif)

多语言切换首先需要在语言翻译模块管理对象，SOUI中使用一个扩展接口`ITranslatorMgr`处理。

下面是新版本的语言翻译接口：

```c++
namespace SOUI
{
    /** 
     * @struct     ITranslator
     * @brief      语言翻译接口
     *
     * Describe
     */
    struct ITranslator : public IObjRef
    {
        /**
         * Load
         * @brief    从资源中加载语言翻译数据
         * @param    LPVOID pData --  资源指针，具体含义由接口的实现来解释
         * @param    UINT uType --  资源类型，具体含义由接口的实现来解释
         * @return   BOOL true-加载成功, false-加载失败
         *
         * Describe  
         */
        virtual BOOL Load(LPVOID pData,UINT uType)=0;
        /**
         * name
         * @brief    获取翻译资源的name
         * @return   SOUI::SStringW 翻译资源的name
         *
         * Describe  
         */
        virtual SStringW name()=0;
        /**
         * guid
         * @brief    获取翻译资源的ID
         * @return   GUID 翻译资源的ID
         *
         * Describe  
         */
        virtual GUID     guid()=0;
        /**
         * tr
         * @brief    执行翻译的接口
         * @param    const SStringW & strSrc --  原字符串
         * @param    const SStringW & strCtx --  翻译上下文
         * @param    SStringW & strRet --  翻译后的字符串
         * @return   BOOL true-翻译成功，false-翻译失败
         *
         * Describe  
         */
        virtual BOOL tr(const SStringW & strSrc,const SStringW & strCtx,SStringW & strRet)=0;
    };


/** 
     * @struct     ITranslatorMgr
     * @brief      语言翻译接口管理器
     *
     * Describe
     */
    struct ITranslatorMgr : public IObjRef
    {
        /**
        * SetLanguage
        * @brief    设置翻译模块当前接受的语言
        * @param [in] const SStringW & strLang --  翻译语言
        *
        * Describe 自动清除语言和目标语言不同的模块
        */
        virtual void SetLanguage(const SStringW & strLang) = 0;

        /**
        * GetLanguage
        * @brief    获取翻译模块当前接受的语言
        * @return SStringW  --  翻译语言
        *
        * Describe 
        */
        virtual SStringW GetLanguage() const = 0;

        /**
         * CreateTranslator
         * @brief    创建一个语言翻译对象
         * @param [out] ITranslator * * ppTranslator --  接收语言翻译对象的指针
         * @return   BOOL true-成功，false-失败
         *
         * Describe  
         */
        virtual BOOL CreateTranslator(ITranslator ** ppTranslator)=0;
        /**
         * InstallTranslator
         * @brief    向管理器中安装一个语言翻译对象
         * @param    ITranslator * ppTranslator -- 语言翻译对象
         * @return   BOOL true-成功，false-失败
         *
         * Describe  
         */

        virtual BOOL InstallTranslator(ITranslator * ppTranslator) =0;
        /**
         * UninstallTranslator
         * @brief    从管理器中卸载一个语言翻译对象
         * @param    REFGUID id --  语言翻译对象的ID
         * @return   BOOL true-成功，false-失败
         *
         * Describe  
         */
        virtual BOOL UninstallTranslator(REFGUID id) =0;
        
        /**
         * tr
         * @brief    翻译字符串
         * @param    const SStringW & strSrc --  原字符串
         * @param    const SStringW & strCtx --  翻译上下文
         * @return   SOUI::SStringW 翻译后的字符串
         *
         * Describe  调用ITranslator的tr接口执行具体翻译过程
         */
        virtual SStringW tr(const SStringW & strSrc,const SStringW & strCtx)=0;


    };

}
```

用户切换UI语言后，使用`SDispatchMessage`方法向所有`SWindow`发送`UM_SETLANGUAGE`消息。
`SWindow`收到该消息后对窗口中需要做语言翻译的对象重新翻译语言后更新显示。

要在SOUI中使用多语言切换，首先需要在`winmain`里设置翻译模块： 

```c++
int WINAPI _tWinMain(HINSTANCE hInstance, HINSTANCE /*hPrevInstance*/, LPTSTR lpstrCmdLine, int /*nCmdShow*/) 
{
  HRESULT hRes = OleInitialize(NULL);
    SASSERT(SUCCEEDED(hRes));

    int nRet = 0;
    
    SComMgr *pComMgr = new SComMgr;

    //将程序的运行路径修改到项目所在目录所在的目录
    TCHAR szCurrentDir[MAX_PATH] = { 0 };
    GetModuleFileName(NULL, szCurrentDir, sizeof(szCurrentDir));
    LPTSTR lpInsertPos = _tcsrchr(szCurrentDir, _T('\\'));
    _tcscpy(lpInsertPos + 1, _T("..\\SouiWizard1"));
    SetCurrentDirectory(szCurrentDir);
    {
        BOOL bLoaded=FALSE;
        CAutoRefPtr<SOUI::IImgDecoderFactory> pImgDecoderFactory;
        CAutoRefPtr<SOUI::IRenderFactory> pRenderFactory;
        CAutoRefPtr<ITranslatorMgr> trans;                  //多语言翻译模块，由translator.dll提供

        bLoaded = pComMgr->CreateRender_GDI((IObjRef**)&pRenderFactory);
        SASSERT_FMT(bLoaded,_T("load interface [render] failed!"));
        bLoaded=pComMgr->CreateImgDecoder((IObjRef**)&pImgDecoderFactory);
        SASSERT_FMT(bLoaded,_T("load interface [%s] failed!"),_T("imgdecoder"));
        bLoaded = pComMgr->CreateTranslator((IObjRef**)&trans);
        SASSERT_FMT(bLoaded, _T("load interface [%s] failed!"), _T("translator"));

        pRenderFactory->SetImgDecoderFactory(pImgDecoderFactory);
        SApplication *theApp = new SApplication(pRenderFactory, hInstance);
        //从DLL加载系统资源
        HMODULE hModSysResource = LoadLibrary(SYS_NAMED_RESOURCE);
        if (hModSysResource)
        {
            CAutoRefPtr<IResProvider> sysResProvider;
            CreateResProvider(RES_PE, (IObjRef**)&sysResProvider);
            sysResProvider->Init((WPARAM)hModSysResource, 0);
            theApp->LoadSystemNamedResource(sysResProvider);
            FreeLibrary(hModSysResource);
        }else
        {
            SASSERT(0);
        }

        CAutoRefPtr<IResProvider>   pResProvider;
#if (RES_TYPE == 0)
        CreateResProvider(RES_FILE, (IObjRef**)&pResProvider);
        if (!pResProvider->Init((LPARAM)_T("uires"), 0))
        {
            SASSERT(0);
            return 1;
        }
#else 
        CreateResProvider(RES_PE, (IObjRef**)&pResProvider);
        pResProvider->Init((WPARAM)hInstance, 0);
#endif

        theApp->InitXmlNamedID(namedXmlID,ARRAYSIZE(namedXmlID),TRUE);
        theApp->AddResProvider(pResProvider);

        if (trans)
        {//加载中文语言翻译包
            theApp->SetTranslator(trans);
            pugi::xml_document xmlLang;
            if (theApp->LoadXmlDocment(xmlLang, _T("cn"), _T("lang")))
            {
                CAutoRefPtr<ITranslator> langCN;
                trans->CreateTranslator(&langCN);
                langCN->Load(&xmlLang.child(L"language"), 1);//1=LD_XML
                trans->InstallTranslator(langCN);
            }
        }
        // BLOCK: Run application
        {
            CMainDlg dlgMain;
            dlgMain.Create(GetActiveWindow());
            dlgMain.SendMessage(WM_INITDIALOG);
            dlgMain.CenterWindow(dlgMain.m_hWnd);
            dlgMain.ShowWindow(SW_SHOWNORMAL);
            nRet = theApp->Run(dlgMain.m_hWnd);
        }

        delete theApp;
    }
    
    delete pComMgr;
    
    OleUninitialize();
    return nRet;
}
```

参见上面红色代码。

需要切换语言时，如下加载新的翻译模块即可：

```c++
void CMainDlg::OnLanguage(int nID)
{
    ITranslatorMgr *pTransMgr =  SApplication::getSingletonPtr()->GetTranslator();
    bool bCnLang = nID == R.id.lang_cn;

        pugi::xml_document xmlLang;
        if (SApplication::getSingletonPtr()->LoadXmlDocment(xmlLang, bCnLang?_T("cn"):_T("en"), _T("lang")))
        {
            CAutoRefPtr<ITranslator> lang;
            pTransMgr->CreateTranslator(&lang);
            lang->Load(&xmlLang.child(L"language"), 1);//1=LD_XML
            pTransMgr->SetLanguage(lang->name());
            pTransMgr->InstallTranslator(lang);
            SDispatchMessage(UM_SETLANGUAGE,0,0);    //soui2.6 新增加的方法。
        }

}
```

 注： 该功能只在SOUI 2.6+版本支持。


