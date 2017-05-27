原文链接：[《第九篇：在SOUI中使用多语言翻译》](http://www.cnblogs.com/setoutsoft/p/3931429.html)

为UI在不同地区显示不同的语言是产品国际化的一个重要要求。

在SOUI中实现了一套类似QT的多语言翻译机制：布局XML不需要调整，程序代码也不需要调整，只需要为不同地区的用户提供不同的语言翻译文件即可。

在SOUI中，我们实现了一个使用明文XML的语言翻译模块：`translator.dll`

为了使用多语言翻译，首先需要准备一个语言翻译的XML文件。demo中使用的翻译文件如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<language name="ch" guid="{0DAEDE3C-6B94-4a81-9A55-C304FDD69D98}">
  <context>
    <!--没有上下文的翻译-->
  </context>
  <context name="editmenu">
    <message>
      <source>copy</source>
      <translation>复制</translation>
    </message>
    <message>
      <source>cut</source>
      <translation>剪切</translation>
    </message>
    <message>
      <source>paste</source>
      <translation>粘贴</translation>
    </message>
  </context>
  <context name="messagebox">
    <message>
      <source>ok</source>
      <translation>确定</translation>
    </message>
    <message>
      <source>cancel</source>
      <translation>取消</translation>
    </message>
    <message>
      <source>retry</source>
      <translation>重试</translation>
    </message>
  </context>
  <context name="dlg_main">
    <message>
      <source>close</source>
      <translation>关闭窗口</translation>
    </message>
  </context>
</language>
```

可以看到该XML中有一个`language`的根节点，该节点有两个属性：`name`和`guid`，这两个属性都是用来标识该翻译文件的。

在`language`节点下，有多个`context`节点，每个`context`节点有一个`name`属性（可以为空），对应一个翻译上下文。

每一个`context`下有不同数量的`message`结点，每个`message`又有两个子节点：`source` 和 `translation`。

`source`对应需要翻译的文字，而`translation`则对应翻译后的文字。

要使用这个语言翻译文件，首先需要从`translator.dll`中创建一个`SOUI::ITranslatorMgr`对象，并将该对象交给`SOUI::SApplication`管理。

再从`ItranslatorMgr`对象创建`SOUI::ITranslator`对象，并将`Itranslator`对象添加到`ItranslatorMgr`管理的翻译列表中。

最后还要为`ITranslator`对象加载翻译数据源（也就是前面提供的XML文件）。

下面是demo中使用和语言翻译相关的代码（见`_tWinMain`函数）

```c++
SApplication *theApp=new SApplication(pRenderFactory,hInstance);//SOUI APP
        CAutoRefPtr<ITranslatorMgr> transMgr; //多语言翻译模块，由translator.dll提供
        transLoader.CreateInstance("translator.dll",(IObjRef**)&transMgr);//
        if(trans)
        {//加载语言翻译包
            theApp->SetTranslator(transMgr);
            pugi::xml_document xmlLang;
            if(theApp->LoadXmlDocment(xmlLang,_T("lang_cn"),_T("translator")))
            {
                CAutoRefPtr<ITranslator> langCN;
                transMgr->CreateTranslator(&langCN);
                langCN->Load(&xmlLang.child(L"language"),1);//1=LD_XML
                transMgr->InstallTranslator(langCN);
            }
        }
```

我们先看一下`editmemu`的XML：

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

在这个XML中，根节点有一个属性trCtx，代表翻译上下文，对应语言翻译文件中的context中的name属性。

在这个menu定义中，所有的菜单项的文字全是英文。其中前面3项：`cut, copy and paste`在语言翻译文件中有对应的翻译项。

下图为程序运行时edit的右键菜单显示结果：

![](assets/002/009-1495897020000.png)

上面演示的是菜单资源的语言翻译，布局XML中的文字的翻译基本一样，只需要为布局的根结点定义一个翻译上下文(trCtx)（没有定义时则从没有指定name属性的context里查找翻译结果）。

可以参见demo的"close”按照的tooltip的翻译。

不管是菜单XML还是布局XML，它们都是静态的，经过设计需要使用代码往UI添加新的文字，同样也需要翻译，这个时候如何处理？

其实看一下静态资源翻译的代码就知道，要实现语言翻译，需要为每一句待翻译的文字调用一个函数：

```c++
SApplication::getSingleton().GetTranslator()->tr(const SStringW & strSrc,const SStringW & strCtx)
```

第一个参数是等翻译字符串，第二个参数是翻译上下文。

考虑到语句太长，系统提供了一个宏：

```c++
#define TR(p1,p2)       SApplication::getSingleton().GetTranslator()->tr(p1,p2)
```

这样用TR就可以实现文字翻译了。