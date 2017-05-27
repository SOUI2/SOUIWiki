原文链接：[《第十一篇：SOUI系统资源管理》](http://www.cnblogs.com/setoutsoft/p/3936066.html)

### SOUI资源管理模块
从前篇已经讲到在SOUI中所有资源文件通过一个`uires.idx`文件进行索引。

这里将介绍在程序中如何引用这些资源文件。

在SOUI系统中，资源文件通过一个统一的接口对象读取：
```c++
namespace SOUI
{
    enum BUILTIN_RESTYPE
    {
        RES_PE=0,
        RES_FILE,
    };

    /**
    * @struct     IResProvider
    * @brief      ResProvider对象
    * 
    * Describe  实现各种资源的加载
    */
    struct IResProvider : public IObjRef
    {
        /**
         * Init
         * @brief    资源初始化函数
         * @param    WPARAM wParam --  param 1 
         * @param    LPARAM lParam --  param 2
         * @return   BOOL -- true:succeed
         *
         * Describe  every Resprovider must implement this interface.
         */
        virtual BOOL Init(WPARAM wParam,LPARAM lParam) =0;
        
        /**
         * HasResource
         * @brief    查询一个资源是否存在
         * @param    LPCTSTR strType --  资源类型
         * @param    LPCTSTR pszResName --  资源名称
         * @return   BOOL -- true存在，false不存在
         * Describe  
         */    
        virtual BOOL HasResource(LPCTSTR strType,LPCTSTR pszResName)=0;

        /**
         * LoadIcon
         * @brief    从资源中加载ICON
         * @param    LPCTSTR pszResName --  ICON名称
         * @param    int cx --  ICON宽度
         * @param    int cy --  ICON高度
         * @return   HICON -- 成功返回ICON的句柄，失败返回0
         * Describe  
         */    
        virtual HICON   LoadIcon(LPCTSTR pszResName,int cx=0,int cy=0)=0;

        /**
         * LoadBitmap
         * @brief    从资源中加载HBITMAP
         * @param    LPCTSTR pszResName --  BITMAP名称
         * @return   HBITMAP -- 成功返回BITMAP的句柄，失败返回0
         * Describe  
         */    
        virtual HBITMAP    LoadBitmap(LPCTSTR pszResName)=0;

        /**
         * LoadCursor
         * @brief    从资源中加载光标
         * @param    LPCTSTR pszResName --  光标名
         * @return   HCURSOR -- 成功返回光标的句柄，失败返回0
         * Describe  支持动画光标
         */    
        virtual HCURSOR LoadCursor(LPCTSTR pszResName)=0;

        /**
         * LoadImage
         * @brief    从资源加载一个IBitmap对象
         * @param    LPCTSTR strType --  图片类型
         * @param    LPCTSTR pszResName --  图片名
         * @return   IBitmap * -- 成功返回一个IBitmap对象，失败返回0
         * Describe  如果没有定义strType，则根据name使用FindImageType自动查找匹配的类型
         */    
        virtual IBitmap * LoadImage(LPCTSTR strType,LPCTSTR pszResName)=0;

        /**
         * LoadImgX
         * @brief    从资源中创建一个IImgX对象
         * @param    LPCTSTR strType --  图片类型
         * @param    LPCTSTR pszResName --  图片名
         * @return   IImgX   * -- 成功返回一个IImgX对象，失败返回0
         * Describe  
         */    
        virtual IImgX   * LoadImgX(LPCTSTR strType,LPCTSTR pszResName)=0;

        /**
         * GetRawBufferSize
         * @brief    获得资源数据大小
         * @param    LPCTSTR strType --  资源类型
         * @param    LPCTSTR pszResName --  资源名
         * @return   size_t -- 资源大小（byte)，失败返回0
         * Describe  
         */    
        virtual size_t GetRawBufferSize(LPCTSTR strType,LPCTSTR pszResName)=0;

        /**
         * GetRawBuffer
         * @brief    获得资源内存块
         * @param    LPCTSTR strType --  资源类型
         * @param    LPCTSTR pszResName --  资源名
         * @param    LPVOID pBuf --  输出内存块
         * @param    size_t size --  内存大小
         * @return   BOOL -- true成功
         * Describe  应该先用GetRawBufferSize查询资源大小再分配足够空间
         */    
        virtual BOOL GetRawBuffer(LPCTSTR strType,LPCTSTR pszResName,LPVOID pBuf,size_t size)=0;

        /**
         * FindImageType
         * @brief    查询与指定名称匹配的资源类型
         * @param    LPCTSTR pszImgName --  资源名称
         * @return   LPCTSTR -- 资源类型，失败返回NULL
         * Describe  没有指定图片类型时默认从这些类别中查找
         */    
        virtual LPCTSTR FindImageType(LPCTSTR pszImgName) =0;
    };

    /**
    * Helper_FindImageType
    * @brief    查询与指定名称匹配的资源类型
    * @param    IResProvider * pResProvider --  当前的ResProvider
    * @param    LPCTSTR pszImgName --  资源名称
    * @return   LPCTSTR -- 资源类型，失败返回NULL
    * Describe  提供一个公共的辅助函数
    */    

    
}//namespace SOUI
```

这个接口的实现类通过实现这些既定接口来完成图标(HICON)，光标(HCURSOR)，位图(HBITMAP)，一般图片(IBitmap)的解码，同时也提供原始数据(RawData)的读取。

在SOUI系统中内置了两种类型的资源加载(ResProvider）模块：SResProviderPE 和 SResProviderFiles，同时也通过外置组件的形式提供了从ZIP文件加载资源的功能。

这三种资源加载方式基本上涵盖了目前常见的资源加载方式。

为了能够从PE的资源数据段中加载资源，我们需要将`uires.idx`中索引的文件转换成PE资源可以识别的资源类型+资源名（不是资源ID）的形式。

为了达到这个目的，我们只需要在VS的资源文件中（.rc）将SOUI的资源中定义的文件按照`uires.idx`定义的类型和名称加进去即可。

手工添加资源文件很难保证不写错。为此，我提供了一个工具（`tools\uiresbuilder.exe`)，这个工具接收一组命令参数，用来将`uires.idx`转换成一个RC编译器可以识别的.rc2文件（命令行参见使用向导生成的工程）。要编译该.rc2文件，需要在.rc的资源包含中加上我们生成的.rc2文件。

![](assets/002/011-1495898542000.png)

如果程序中的资源不从PE资源加载，则不需要编译`soui_res.rc2`文件，以减少程序体积。

`SResProviderPE`, `SResProviderFiles` 和 `SResProviderZIP`分别从PE资源，文件夹及ZIP文件包中初始化资源：

```c++
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
        bLoaded=pComMgr->CreateResProvider_ZIP((IObjRef**)&pResProvider);
        SASSERT_FMT(bLoaded,_T("load interface [%s] failed!"),_T("resprovider_zip"));

        ZIPRES_PARAM param;
        param.ZipFile(pRenderFactory, _T("uires.zip"),"souizip");
        bLoaded = pResProvider->Init((WPARAM)&param,0);
        SASSERT(bLoaded);
#endif
```

资源加载成功后，调用`SApplication::AddResProvider(IResProvider *)`接口将创建的资源加载器交给SOUI系统管理。

`SApplication::AddResProvider`可以调用多次，便于加载不同的资源。

资源加载后不再需要了也可以使用`SApplication::RemoveResProvider(IResProvider *)`来删除。

程序中要使用一个资源时，首先调用`SApplication::HasResource`来查询一个资源是否存在，然后再根据资源类型选择不同的接口加载资源。

SApplication中管理着一个IResProvider列表，系统采用后进先查的算法处理资源重名，即最后调用AddResProvider加进来的资源加载器优先级最高。

### 应用程序中资源的组织
一个基于SOUI开发的应用程序通常资源分为两部分：

 -1.控件默认的系统资源，也可以理解为主题(theme)资源。

 -2.应用程序自定义的资源。

#### 系统资源又分为3部分：

##### 1.控件默认的皮肤(Skin)：
SOUI中很多控件都必须要定义一个皮肤(ISkinObj,称之为绘图对象更好解理)，例如绘制checkbox，radiobox，combobox等控件出出现的位图部分。如何在应用程序中使用了这些控件，则必须为它们定义这些皮肤资源。为了简化界面配置，我统一将这些资源进行命名，并打包到一起（参见`trunk\soui-sys-resource\theme_sys_res\sys_xml_skin.xml`）。

下面是系统中使用定义的命名ISkinObj:
```c++
const wchar_t * BUILDIN_SKIN_NAMES[]=
{
    L"_skin.sys.checkbox",
    L"_skin.sys.radio",
    L"_skin.sys.focuscheckbox",
    L"_skin.sys.focusradio",
    L"_skin.sys.btn.normal",
    L"_skin.sys.scrollbar",
    L"_skin.sys.border",
    L"_skin.sys.dropbtn",
    L"_skin.sys.tree.toggle",
    L"_skin.sys.tree.checkbox",
    L"_skin.sys.tab.page",
    L"_skin.sys.header",
    L"_skin.sys.split.vert",
    L"_skin.sys.split.horz",
    L"_skin.sys.prog.bkgnd",
    L"_skin.sys.prog.bar",
    L"_skin.sys.slider.thumb",
    L"_skin.sys.btn.close",
    L"_skin.sys.btn.minimize",
    L"_skin.sys.btn.maxmize",
    L"_skin.sys.btn.restore",
    L"_skin.sys.menu.check",
    L"_skin.sys.menu.sep",
    L"_skin.sys.menu.border",
    L"_skin.sys.menu.skin",
    L"_skin.sys.icons",
    L"_skin.sys.wnd.bkgnd"
};
```

如果程序中没有使用到特定控件，也可以不在系统资源中提供对应的ISkinObj。

##### 2.系统使用的MsgBox布局模板：

MsgBox是应用程序常用的功能。SOUI通过提供一个MSGBOX的XML布局模板来实现。如果需要修改MsgBox的样式，只需要修改这个XML。

```xml
<SOUI title="mesagebox" width="200" height="100" appwin="0" frameSize="40,30,10,80" minSize="300,100" resize="0" translucent="1" trCtx="messagebox">
  <style>
    <class name="normalbtn" skin="_skin.sys.btn.normal" font="" colorText="#385e8b" colorTextDisable="#91a7c0" textMode="25" cursor="hand" margin-x="0"/>
  </style>
  <root skin="_skin.sys.wnd.bkgnd">
    <caption id="101" pos="0,0,-0,29">
      <text pos="11,9" class="cls_txt_red" name="msgtitle" >title</text>
      <imgbtn id="2" skin="_skin.sys.btn.close"    pos="-45,0" tip="close"/>
    </caption>

    <window pos="5,30,-5,-50">
      <icon name="msgicon" pos="0,0,32,32" display="0"/>
      <text name="msgtext" pos="[0,0" colorText="#0000FF" multilines="1" maxWidth="300"/>
    </window>
    <tabctrl name="btnSwitch" pos="0,-50,-0,-0" tabHeight="0">
      <page>
        <button pos="|-50,10,|50,-10" name="button1st" class="normalbtn">button1</button>
      </page>
      <page>
        <button pos="|-100,10,|-10,-10" name="button1st" class="normalbtn">button1</button>
        <button pos="|10,10,|100,-10" name="button2nd" class="normalbtn">button2</button>
      </page>
      <page>
        <button pos="|-140,10,|-50,-10" name="button1st" class="normalbtn">button1</button>
        <button pos="|-45,10,|45,-10" name="button2nd" class="normalbtn">button2</button>
        <button pos="|50,10,|140,-10" name="button3rd" class="normalbtn">button3</button>
      </page>
    </tabctrl>
  </root>
</SOUI>
```

上面是SOUI默认提供的模板。

在这个模板中，只提供了必须的命名对象。如果要修改这个模板，这些命名对象不能缺少，不过布局位置可以任意调整。

##### 3.Edit控件使用的右键菜单定义XML：

edit控件的菜单相对固定，因此我们也采用系统资源的形式提供一个预定义的右键菜单定义。

```xml
<editmenu trCtx="editmenu" iconSkin="_skin.sys.icons" itemHeight="26" iconMargin="4" textMargin="8" >
  <item id="1" icon="3">cut</item>
  <item id="2" icon="4">copy</item>
  <item id="3" icon="5">paste</item>
  <item id="4" >delete</item>
  <sep/>
  <item id="5">select all</item>
</editmenu>
```

和前面两个不同，菜单资源每一个item必须包含一个id，取值从1-5,菜单项的位置可以任意。

#### 应用程序自定义资源

很显然，系统资源提供的样式使得应用中每一个同类控件长得都一样。但是很多时候我们会希望两个功能相似的控件有不一样的长相，这就需要使用用户自定义资源。

用户自定义资源和系统资源一样，只不过它可以包含更多类型（任意类型），资源也可以任意命名（只要不和系统资源冲突）。

下面为demo中使用的自定义资源(uires.idx)：

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

不管是系统资源还是用户资源都由资源管理模块管理。

实际上这两种资源可以合并到一个资源包中交给系统管理。有心人可能注意到了项目中使用的系统资源使用的是一个资源DLL，并且没有uires.idx文件。正是因为使用了资源DLL这一形式，才可以不提供uires.idx文件，因为PE资源本身已经有了分类命名（每一个资源都在一个类型下）。

#### DEMO中使用系统资源和用户资源
```c++
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
        bLoaded=pComMgr->CreateResProvider_ZIP((IObjRef**)&pResProvider);
        SASSERT_FMT(bLoaded,_T("load interface [%s] failed!"),_T("resprovider_zip"));

        ZIPRES_PARAM param;
        param.ZipFile(pRenderFactory, _T("uires.zip"),"souizip");
        bLoaded = pResProvider->Init((WPARAM)&param,0);
        SASSERT(bLoaded);
#endif
        //将创建的IResProvider交给SApplication对象
        theApp->AddResProvider(pResProvider);

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
```

可以看到这里根据预定义宏：RES_TYPE提供了3种参考资源加载形式来加载用户自定义资源。然后再采用SResProviderPE的资源加载器从系统资源DLL中加载系统资源。

加载系统资源一个关键步骤在于调用:

```c++
theApp->LoadSystemNamedResource(sysSesProvider);
```

这个函数从系统资源加载器中读取那些命名的系统资源。

所有资源加载完成后调用：

```c++
        //加载全局资源描述XML
        theApp->Init(_T("xml_init")); 
```

来初始化资源中定义的全局Skin,Style,ObjAttr对象。

大家可以想一想怎么样把这两种资源使用一个资源加载器来完成。