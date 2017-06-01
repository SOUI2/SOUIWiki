原文链接：[《第二十七篇：SOUI中控件属性查询方法》](http://www.cnblogs.com/setoutsoft/p/4700128.html)

SOUI项目的SVN根目录下有一个doc目录，下面有一份控件属性表。包含了大部分控件的大部分属性，不过也不一定完全准确。最保险的办法还是查源代码。

SOUI对象包含控件及`ISkinObj`等从SObject派生的对象都可以使用XML配置属性。要知道如何查SOUI对象属性，首先要看一下SOUI解释属性的流程。

```c++
BOOL SObject::InitFromXml( pugi::xml_node xmlNode )
    {
        if(!xmlNode) return FALSE;
#ifdef _DEBUG
        {
            pugi::xml_writer_buff writer;
            xmlNode.print(writer,L"\t",pugi::format_default,pugi::encoding_utf16);
            m_strXml=SStringW(writer.buffer(),writer.size());
        }
#endif

        //设置当前对象的属性

        //优先处理"class"属性
        pugi::xml_attribute attrClass=xmlNode.attribute(L"class");
        if(attrClass)
        {
            attrClass.set_userdata(1);      //预处理过的属性，给属性增加一个userdata
            SetAttribute(attrClass.name(), attrClass.value(), TRUE);
        }
        for (pugi::xml_attribute attr = xmlNode.first_attribute(); attr; attr = attr.next_attribute())
        {
            if(attr.get_userdata()) continue;   //忽略已经被预处理的属性
            SetAttribute(attr.name(), attr.value(), TRUE);
        }
        if(attrClass)
        {
            attrClass.set_userdata(0);
        }
        //调用初始化完成接口
        OnInitFinished(xmlNode);
        return TRUE;
    }
```

在`InitFromXml`方法中，首先获取`class`属性交给对象解释，然后再枚举其它属性。

获取到一个属性以后调用`SOjbect::SetAttribute`方法来处理属性。

```c++
class SObject{
public
        virtual HRESULT SetAttribute(const SStringW &  strAttribName, const SStringW &  strValue, BOOL bLoading)
        {
            return DefAttributeProc(strAttribName,strValue,bLoading);
        }
};
```

`SetAttribute`是一个虚函数，所有派生类都可以通过重载该方法来实现不同属性的处理。

在SOUI中提供了一组宏来处理SObject的属性，下面看一下SWindow类的实现：

```c++
SOUI_ATTRS_BEGIN()
            ATTR_CUSTOM(L"id",OnAttrID)
            ATTR_STRINGW(L"name",m_strName,FALSE)
            ATTR_CUSTOM(L"skin", OnAttrSkin)        //直接获得皮肤对象
            ATTR_SKIN(L"ncskin", m_pNcSkin, TRUE)   //直接获得皮肤对象
            ATTR_CUSTOM(L"class", OnAttrClass)      //获得style
            ATTR_INT(L"data", m_uData, 0 )
            ATTR_CUSTOM(L"enable", OnAttrEnable)
            ATTR_CUSTOM(L"visible", OnAttrVisible)
            ATTR_CUSTOM(L"show", OnAttrVisible)
            ATTR_CUSTOM(L"display", OnAttrDisplay)
            ATTR_CUSTOM(L"pos", OnAttrPos)
            ATTR_CUSTOM(L"offset", OnAttrOffset)
            ATTR_CUSTOM(L"pos2type", OnAttrPos2type)
            ATTR_CUSTOM(L"cache", OnAttrCache)
            ATTR_CUSTOM(L"alpha",OnAttrAlpha)
            ATTR_CUSTOM(L"layeredWindow",OnAttrLayeredWindow)
            ATTR_CUSTOM(L"trackMouseEvent",OnAttrTrackMouseEvent)
            ATTR_I18NSTRT(L"tip", m_strToolTipText, FALSE)  //使用语言包翻译
            ATTR_INT(L"msgTransparent", m_bMsgTransparent, FALSE)
            ATTR_INT(L"maxWidth",m_nMaxWidth,FALSE)
            ATTR_INT(L"clipClient",m_bClipClient,FALSE)
            ATTR_INT(L"focusable",m_bFocusable,FALSE)
            ATTR_INT(L"float",m_bFloat,FALSE)
            ATTR_CHAIN(m_style)                     //支持对style中的属性定制
        SOUI_ATTRS_END()
```

`SOUI_ATTRS_BEGIN + SOUI_ATTRS_END`宏展开构成一个空的`SetAttribute`函数。
ATTR_XXX则对不同类型的属性提供一种固定的处理方式，全部属性形成一个属性映射表。

因此要查对象属性，首先就是查对象代码中对应的属性映射表，同时也别忘记去查对象的基类的属性映射表。

有些对象（如Richedit)的属性可能在这个属性映射表里找不到，那又是为什么呢？
注意`SetAttribute`最后调用了`DefAttributeProc`，使用属性映射表构造的属性处理函数也同样会调用`DefAttributeProc`。
在`DefAttributeProc`中可以批量的按照自己的方式处理属性。下面看一下`Richedit::DefAttributeProc`的实现：

```c++
HRESULT SRichEdit::DefAttributeProc(const SStringW & strAttribName,const SStringW & strValue, BOOL bLoading)
{
    HRESULT hRet=S_FALSE;
    DWORD dwBit=0,dwMask=0;
    //hscrollbar
    if(strAttribName.CompareNoCase(L"hscrollBar")==0)
    {
        if(strValue==L"0")
            m_dwStyle&=~WS_HSCROLL;
        else
            m_dwStyle|=WS_HSCROLL;
        dwBit|=TXTBIT_SCROLLBARCHANGE;
        dwMask|=TXTBIT_SCROLLBARCHANGE;
    }
    //vscrollbar
    else if(strAttribName.CompareNoCase(L"vscrollBar")==0)
    {
        if(strValue==L"0")
            m_dwStyle&=~WS_VSCROLL;
        else
            m_dwStyle|=WS_VSCROLL;
        dwBit|=TXTBIT_SCROLLBARCHANGE;
        dwMask|=TXTBIT_SCROLLBARCHANGE;
    }
    //auto hscroll
    else if(strAttribName.CompareNoCase(L"autoHscroll")==0)
    {
        if(strValue==L"0")
            m_dwStyle&=~ES_AUTOHSCROLL;
        else
            m_dwStyle|=ES_AUTOHSCROLL;
        dwBit|=TXTBIT_SCROLLBARCHANGE;
        dwMask|=TXTBIT_SCROLLBARCHANGE;
    }
    //auto hscroll
    else if(strAttribName.CompareNoCase(L"autoVscroll")==0)
    {
        if(strValue==L"0")
            m_dwStyle&=~ES_AUTOVSCROLL;
        else
            m_dwStyle|=ES_AUTOVSCROLL;
        dwBit|=TXTBIT_SCROLLBARCHANGE;
        dwMask|=TXTBIT_SCROLLBARCHANGE;
    }
    //multilines
    else if(strAttribName.CompareNoCase(L"multiLines")==0)
    {
        if(strValue==L"0")
            m_dwStyle&=~ES_MULTILINE;
        else
            m_dwStyle|=ES_MULTILINE,dwBit|=TXTBIT_MULTILINE;
        dwMask|=TXTBIT_MULTILINE;
    }
    //readonly
    else if(strAttribName.CompareNoCase(L"readOnly")==0)
    {
        if(strValue==L"0")
            m_dwStyle&=~ES_READONLY;
        else
            m_dwStyle|=ES_READONLY,dwBit|=TXTBIT_READONLY;
        dwMask|=TXTBIT_READONLY;
        if(!bLoading)
        {//update dragdrop
            OnEnableDragDrop(!(m_dwStyle&ES_READONLY) && m_fEnableDragDrop);
        }
    }
    //want return
    else if(strAttribName.CompareNoCase(L"wantReturn")==0)
    {
        if(strValue==L"0")
            m_dwStyle&=~ES_WANTRETURN;
        else
            m_dwStyle|=ES_WANTRETURN;
    }
    //password
    else if(strAttribName.CompareNoCase(L"password")==0)
    {
        if(strValue==L"0")
            m_dwStyle&=~ES_PASSWORD;
        else
            m_dwStyle|=ES_PASSWORD,dwBit|=TXTBIT_USEPASSWORD;
        dwMask|=TXTBIT_USEPASSWORD;
    }
    //number
    else if(strAttribName.CompareNoCase(L"number")==0)
    {
        if(strValue==L"0")
            m_dwStyle&=~ES_NUMBER;
        else
            m_dwStyle|=ES_NUMBER;
    }
    //password char
    else if(strAttribName.CompareNoCase(L"passwordChar")==0)
    {
        SStringT strValueT=S_CW2T(strValue);
        m_chPasswordChar=strValueT[0];
    }
    //enabledragdrop
    else if(strAttribName.CompareNoCase(L"enableDragdrop")==0)
    {
        if(strValue==L"0")
        {
            m_fEnableDragDrop=FALSE;
        }else
        {
            m_fEnableDragDrop=TRUE;
        }
        if(!bLoading)
        {
            OnEnableDragDrop( !(m_dwStyle&ES_READONLY) & m_fEnableDragDrop);
        }
    }
    //auto Sel
    else if(strAttribName.CompareNoCase(L"autoSel")==0)
    {
        if(strValue==L"0")
        {
            m_fAutoSel=FALSE;
        }else
        {
            m_fAutoSel=TRUE;
        }
    }
    else
    {
        hRet=__super::DefAttributeProc(strAttribName,strValue,bLoading);
    }
    if(!bLoading)
    {
        m_pTxtHost->GetTextService()->OnTxPropertyBitsChange(dwMask,dwBit);
        hRet=TRUE;
    }
    return hRet;
}
```

可以看到在这个方法中对很多属性做了处理。

总结：

<font color=red>先查目标对象的属性映射表，再找基类的属性映射表，最后查对象的DefAttributeProc。</font>