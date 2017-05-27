原文链接：[《第四篇：SOUI资源文件组织》](http://www.cnblogs.com/setoutsoft/p/3923610.html)

### 什么是资源？
现代的软件只要有UI，基本上少不了资源。

资源是什么？资源就是在程序运行时提供固定的数据源的文件。

在MFC当道的时代，资源一般就是位图(Bitmap)，图标(Icon)，光标(Cursor)，对话框模板(Dialog)等资源。

在SOUI中，资源主要变成了XML布局和PNG图片文件。

### SOUI-DEMO的资源解析
首先看一下SOUI-DEMO中用到的资源索引XML(`uires.idx`)：

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

`uires.idx`文件是SOUI资源的入口，定义了程序中使用的各种资源，以"`resource`"为根节点。

在soui-demo的`uires.idx`中，我定义了`UIDEF，ICON，CURSOR，LAYOUT，IMGX，GIF，rtf，script，translator`这些资源类型。

其实你还可以定义任意的资源类型，只要类型长度不超过30个字符。

每一个资源类型下都是一些file元素，每个file包含两个属性：name 及path。

name是UI布局的XML文件引用该资源的标识，而path则是该资源真实的文件存储路径。

除了UIDEF中定义的xml_init及layout中定义的布局XML外，其它资源都比较简单，就是提供一种从name映射文件的方式。

### UIDEF资源（布局XML再单独开篇介绍）
其中uidef中定义的`init.xml`用来定义SOUI中使用的全局UI定义。

这个文件一般应该在main函数中被SApplication对象使用，如：

```c++
//...    
        //定义一个唯一的SApplication对象，SApplication管理整个应用程序的资源
        SApplication *theApp=new SApplication(pRenderFactory,hInstance);
        //... 
       //加载全局资源描述XML
        theApp->Init(_T("xml_init")); 
```

这里直接使用uidef中定义的name属性来初始化系统。`SApplication::Init`默认从uidef段中去查找xml_init资源，当然也可以定义在其它资源类型如xml中，但是需要在init中显示指定资源类型。

下面我们打开`xml\init.xml`看一下里面的内容：

```xml
<?xml version="1.0" encoding="utf-8"?>
<UIDEF>
  <font face="微软雅黑" size="18"/>
  
  <string>
    <ver value="1.0"/>
  </string>
  
  <skin>
    <imglist name="skin_page_icons"    src="imgx:png_page_icons" states="9"/>
    <imglist name="skin_small_icons" src="imgx:png_small_icons" states="12"/>
    <imglist name="skin_tree_icon"        src="imgx:png_treeicon" states="3"/>

    <imglist name="skin_menuicon" src="imgx:png_small_icons" states="12"/>
    <border name="skin_menuborder" src="imgx:png_menu_border" left="2" top="2" border="2,2,2,2" key="FF00FF" alpha="100"/>
    <imglist name="skin_webbtn_back" src="imgx:webbtn_back" states="4"/>
    <imglist name="skin_webbtn_forward" src="imgx:webbtn_forward" states="4"/>
    <imglist name="skin_webbtn_refresh" src="imgx:webbtn_refresh" states="4"/>
    <imglist name="skin_tab_left" src="imgx:png_tab_left" states="3"/>
    <imglist name="skin_tab_left_splitter" src="imgx:png_tab_left_splitter"/>
    <imglist name="skin_tab_main" src="imgx:png_tab_main" states="3"/>
    <imglist name="skin_btn_menu" src="imgx:btn_menu" states="3"/>
    <gif name="gif_horse" src="gif:gif_horse"/>
    <gif name="gif_penguin" src="gif:gif_penguin"/>

  </skin>

  <style>
    <class name="cls_dlg_frame" skin="_skin.sys.wnd.bkgnd" font="" colorText="#000000" margin-x="0"/>

    <class name="cls_btn_link"    cursor="hand" colorHover="#0A84D2" />
    <!--定义文字按钮的样式-->
    <class name="cls_btn_weblink" cursor="hand" colorText="#1e78d5" colorHover="#1e78d5" font="italic:1" fontHover="underline:1,italic:1" />

    <class name="cls_txt_red"     font="face:宋体,bold:1"  colorText="#FF0000" />
    <!--定义白色粗体宋体-->
    <class name="cls_txt_black"   font="face:宋体,bold:1"  colorText="#000000" />
    <!--定义黑色粗体宋体-->
    <class name="cls_txt_white"   font="face:宋体,bold:1"  colorText="#FFFFFF" />
    <!--定义白色粗体宋体-->

    <class name="normalbtn" font="" colorText="#385e8b" colorTextDisable="#91a7c0" textMode="25" cursor="hand" margin-x="0"/>

    <class name="toptext" textMode="20" />
    <class name="vcentertext" textMode="24"/>
    <class name="rightvcentertext" textMode="26"/>
    <class name="centertext" textMode="25"/>
    <class name="righttext" textMode="22"/>

    <class name="linkimage" cursor="hand"/>

    <class name="cls_edit" ncSkin="_skin.sys.border" margin-x="2" margin-y="2" />
  </style>

  <objattr>
    <button class="normalbtn"/>
    <imgbtn class="linkimage"/>
    <tabctrl colorText="000000" align="top" tabWidth="70" tabHeight="38" tabSpacing="0" tabPos="10" dotted="1"/>
    <edit transParent="1" margin-x="2" margin-y="2"/>
    <treectrl colorItemBkgnd="#FFFFFF" colorItemSelBkgnd="#000088" colorItemText="#000000" colorItemSelText="#FFFFFF" indent="17" itemMargin="4"/>
  </objattr>
</UIDEF>
```

**这个xml_init必须是以UIDEF为唯一根节点。**

在UIDEF下，可以定义`font，string，skins，style，objattr`五个子节点。

其中，font定义SOUI中使用的默认字体，只有face和size两个属性。

string是一个字符串表，定义一个"**name-字符串**"映射，在布局的XML文件中可以通过引用字符串的name来获得字符串。

skins定义SOUI中使用的全局窗口元素绘制对象，每一个对象都对应一个`SOUI::ISkinObj`的派生类。

### skins
SOUI系统默认实现了`SSkinImgList(imglist)`, `SSkinImgFrame(imgframe)`, `SSkinButton(button)`, `SSkinGradation(gradation)`, `SSkinScrollbar(scrollbar)`, `SSkinMenuBorder(border)`这几种绘图类型。SSkinImgList为SOUI中的C++类名，imglist为在skins节点中的元素类型名。

下面分别介绍这几种绘图类型：

#### 1.imglist

imglist是一个图片序列对象，可以包含一组小图片，常见的如按钮需要使用的4种状态图。

![](assets/002/004-1495814809000.png)

imglist包含4个属性：

```c++
SOUI_ATTRS_BEGIN()
        ATTR_CUSTOM(L"src", OnAttrImage)    //skinObj引用的图片文件定义在uires.idx中的name属性。
        ATTR_INT(L"tile", m_bTile, TRUE)    //绘制是否平铺,0--位伸（默认），其它--平铺
        ATTR_INT(L"vertical", m_bVertical, TRUE)//子图是否垂直排列，0--水平排列(默认), 其它--垂直排列
        ATTR_INT(L"states",m_nStates,TRUE)  //子图数量,默认为1
    SOUI_ATTRS_END()
```

假定上图的图片在`uires.idx`中的定义为：

```xml
<skins>
    <imglist name=“skin_btn_next" src="imgx:btn_next" states="4" tile="0" vertical="0"/>
</skins>
```

要在soui中引用这个图片，需要在`init.xml`的skins结节中做如下声明：

```xml
<skins>
    <imglist name=“skin_btn_next" src="imgx:btn_next" states="4" tile="0" vertical="0"/>
</skins>
```

在上面的skin定义中，

name属性告诉系统如何引用定义的imglist

src属性定义该skin需要使用哪一个图片资源，资源引用格式为`type:name`，如上面使用的`imgx:btn_next`，对于图片资源，通常情况下也可以不指定type,系统会自动在常用的图片类型下查找，但不建议这样使用。

states定义图中包含多少个子图。

title定义图片在放大显示时时平铺还是拉伸，默认为拉伸。

vertical属性定义图中的子图的排列方式。

在本例子中tile和vertical属性都可以不指定。

#### 2.imgframe

imgframe是一个提供九宫格显示的绘图对象，SSkinImgFrame派生自SSkinImgList，因此imgframe也拥有imglist的全部属性。

此外，imgframe提供了几个新的属性：

```c++
SOUI_ATTRS_BEGIN()
        ATTR_INT(L"left", m_rcMargin.left, TRUE)        //九宫格左边距
        ATTR_INT(L"top", m_rcMargin.top, TRUE)          //九宫格上边距
        ATTR_INT(L"right", m_rcMargin.right, TRUE)      //九宫格右边距
        ATTR_INT(L"bottom", m_rcMargin.bottom, TRUE)    //九宫格下边距
        ATTR_INT(L"margin-x", m_rcMargin.left=m_rcMargin.right, TRUE)//九宫格左右边距
        ATTR_INT(L"margin-y", m_rcMargin.top=m_rcMargin.bottom, TRUE)//九宫格上下边距
    SOUI_ATTRS_END()
```

![](assets/002/004-1495814948000.png)

imgframe的格式如上图，在imgframe中通过`left, top, right, bottom`来定义九宫格。

#### 3.button

button绘图对象是绘制按钮时使用的，它使用渐变实现绘制按钮的4种状态。

包含以下属性：

```c++
SOUI_ATTRS_BEGIN()
        ATTR_COLOR(L"colorBorder", m_crBorder, TRUE)                //边框颜色
        ATTR_COLOR(L"colorUp", m_crUp[ST_NORMAL], TRUE)             //正常状态渐变起始颜色
        ATTR_COLOR(L"colorDown", m_crDown[ST_NORMAL], TRUE)         //正常状态渐变终止颜色
        ATTR_COLOR(L"colorUpHover", m_crUp[ST_HOVER], TRUE)         //浮动状态渐变起始颜色
        ATTR_COLOR(L"colorDownHover", m_crDown[ST_HOVER], TRUE)     //浮动状态渐变终止颜色
        ATTR_COLOR(L"colorUpPush", m_crUp[ST_PUSHDOWN], TRUE)       //下压状态渐变起始颜色
        ATTR_COLOR(L"colorDownPush", m_crDown[ST_PUSHDOWN], TRUE)   //下压状态渐变终止颜色
        ATTR_COLOR(L"colorUpDisable", m_crUp[ST_DISABLE], TRUE)     //禁用状态渐变起始颜色
        ATTR_COLOR(L"colorDownDisable", m_crDown[ST_DISABLE], TRUE) //禁用状态渐变终止颜色
    SOUI_ATTRS_END()
```

#### 4.gradation

渐变绘图对象，提供3个属性：

```c++
 SOUI_ATTRS_BEGIN()
        ATTR_COLOR(L"colorFrom", m_crFrom, TRUE)    //渐变起始颜色
        ATTR_COLOR(L"colorTo", m_crTo, TRUE)        //渐变终止颜色
        ATTR_INT(L"vertical", m_bVert, TRUE)        //渐变方向,0--水平(默认), 1--垂直
    SOUI_ATTRS_END()
```

#### 5.scrollbar

滚动条皮肤，虽然它派生自imglist，实际上imglist中实现的属性在scrollbar中没有意义，只是为了省点代码。

```c++
SOUI_ATTRS_BEGIN()
        ATTR_INT(L"margin",m_nMargin,FALSE)             //边缘不拉伸大小
        ATTR_INT(L"hasGripper",m_bHasGripper,FALSE)     //滑块上是否有帮手(gripper)
        ATTR_INT(L"hasInactive",m_bHasInactive,FALSE)   //是否有禁用态
    SOUI_ATTRS_END()
```

一般的scrollbar皮肤资源如下：

![](assets/002/004-1495815060000.png)

如果没有帮手也没有禁用状态，图片应该是8*3的正方形网格。

有帮手则X增加一个网格，有禁用状态则Y增加一个网格。

#### 6.border

给menu用的，以后再介绍。

#### style

在style节点中，定义UI布局中SOUI窗口对象的属性集合，它们是SWindow对象的属性，所有SWindow对象都可以通过class属性来引用style节点中定义的属性集合。

#### objattr

 控件的默认属性。

SOUI可以为每一类UI控件通过objattr来提供一种默认属性集合，以减少在XML布局中的重复定义。