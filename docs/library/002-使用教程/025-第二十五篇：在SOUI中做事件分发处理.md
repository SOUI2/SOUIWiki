原文链接：[《第二十五篇：在SOUI中做事件分发处理》](http://www.cnblogs.com/setoutsoft/p/4399712.html)

不同的SOUI控件可以产生不同的事件。SOUI系统中提供了两种事件处理方式：事件订阅 + 事件处理映射表(参见[第八篇：SOUI中控件事件的响应](http://www.cnblogs.com/setoutsoft/p/3930419.html))

事件订阅由于直接将事件及事件处理函数连接，不存在事件分发的问题，这里主要介绍使用事件映射表时的事件分发。

在回答这个问题前，首先了解一下什么是事件分发。

在大型项目中，程序逻辑可能非常复杂，如果将所有UI中控件的事件处理集中在一个消息/事件映射表里，代码的可维护性会变得非常差。解决这个问题常见的方法就是将事件进行分类（如根据来源分类），不同类别的事件采用一个独立的事件处理对象来处理，这就是事件分发的核心。

目前流行的UI通常采用Tab控件来组织UI，不同的功能放到不同的Tab页中，不同的Tab页可能互不相干的功能模块，对于类似这样的情形很自然的会想到采用事件分发机制来实现模块之间逻辑的解耦（如下图中SoTool采用的UI）。

![](assets/002/025-1496323861000.png)

在上面的UI中，虽然整个UI被TAB分成了6个页面，但是6个页面都存在于同一个宿主窗口中。

一般情况下，如果UI相对比较简单，我们推荐直接在宿主窗口的事件处理映射表中统一处理控件事件。

但是当出现如上图这样复杂的界面时，最好是将不同功能页的事件处理在不同的对象中分别处理。

在MFC中，一个类要处理消息，这个类通常派生自`CCmdTarget`（可能记错了，太久不用MFC了），主窗口收到的消息会自动路由到这个消息处理对象中。

在WTL中，WTL提供了一组消息映射宏：`CHAIN_MSG_MAP`，`CHAIN_MSG_MAP_MEMBER`等以便将消息分发到同样实现了消息映射表的任意C++对象。

SOUI的事件分发采用了WTL消息分发类似的机制，同样采用事件映射宏的方式来构造事件映射表，下面是SOUI中几个主要的和事件分发相关的宏：

```c++
#define EVENT_MAP_BEGIN()                           \
protected:                                          \
    virtual BOOL _HandleEvent(SOUI::EventArgs *pEvt)\
    {                                               \
        UINT      uCode = pEvt->GetID();            \


#define EVENT_MAP_DECLEAR()                         \
protected:                                          \
    virtual BOOL _HandleEvent(SOUI::EventArgs *pEvt);\
    

#define EVENT_MAP_BEGIN2(classname)                 \
    BOOL classname::_HandleEvent(SOUI::EventArgs *pEvt)\
    {                                               \
        UINT      uCode = pEvt->GetID();            \
 

#define EVENT_MAP_END()                             \
        return __super::_HandleEvent(pEvt);         \
    }                                               \

#define EVENT_MAP_BREAK()                           \
    return FALSE;                                   \
    }                                               \
 
#define CHAIN_EVENT_MAP(ChainClass)                 \
        if(ChainClass::_HandleEvent(pEvt))          \
            return TRUE;                            \
 
#define CHAIN_EVENT_MAP_MEMBER(theChainMember)      \
    {                                               \
    if(theChainMember._HandleEvent(pEvt))           \
        return TRUE;                                \
    }

#define EVENT_CHECK_SENDER_ROOT(pRoot)              \
    {                                               \
    SWindow *pWnd = sobj_cast<SWindow>(pEvt->sender);\
    if(!pWnd->IsDescendant(pRoot))                  \
        return FALSE;                               \
    }

// void OnEvent(EventArgs *pEvt)
#define EVENT_HANDLER(cd, func)                     \
    if(cd == uCode)                                 \
    {                                               \
        func(pEvt); return TRUE;                    \
    }
```

下面是`SoTool`中的`MainDlg`中的事件处理：

```c++
//soui消息
    EVENT_MAP_BEGIN()
        EVENT_NAME_COMMAND(L"btn_close", OnClose)
        EVENT_NAME_COMMAND(L"btn_min", OnMinimize)
        EVENT_NAME_COMMAND(L"btn_max", OnMaximize)
        EVENT_NAME_COMMAND(L"btn_restore", OnRestore)
        CHAIN_EVENT_MAP_MEMBER(m_imgMergerHandler)
        CHAIN_EVENT_MAP_MEMBER(m_codeLineCounter)
        CHAIN_EVENT_MAP_MEMBER(m_2UnicodeHandler)
        CHAIN_EVENT_MAP_MEMBER(m_folderScanHandler)
        CHAIN_EVENT_MAP_MEMBER(m_calcMd5Handler)
    EVENT_MAP_END()
```

上面代码中，`EVENT_MAP_BEGIN()`和`EVENT_MAP_END()`这两个宏构造出一个空的事件处理函数，该函数自动将未处理的事件交给基类的事件处理函数处理。

如果基类中没有事件处理函数，显然这个事件映射表编译不能通过，此时SOUI提供了另一个`EVENT_MAP_BREAK()`来代替。

上面的事件分发表中，我使用`CHAIN_EVENT_MAP_MEMBER`宏将来自不同页面的控件事件传递到不同的事件处理对象中。

下面代码是`m_imgMergerHandler`对象头文件。

```c++
class CImageMergerHandler : public IFileDropHandler
{
friend class CMainDlg;
public:
    CImageMergerHandler(void);
    ~CImageMergerHandler(void);
    
    void OnInit(SWindow *pRoot);
    
    void AddFile(LPCWSTR pszFileName);
protected:
    virtual void OnFileDropdown(HDROP hDrop);

    void OnSave();
    void OnClear();
    void OnModeHorz();
    void OnModeVert();
    
    EVENT_MAP_BEGIN()
        EVENT_CHECK_SENDER_ROOT(m_pPageRoot)
        EVENT_NAME_COMMAND(L"btn_save", OnSave)
        EVENT_NAME_COMMAND(L"btn_clear", OnClear)
        EVENT_NAME_COMMAND(L"radio_horz", OnModeHorz)
        EVENT_NAME_COMMAND(L"radio_vert", OnModeVert)        
    EVENT_MAP_BREAK()
    
    SWindow *m_pPageRoot;
    SImgCanvas *m_pImgCanvas;
};
```

可以看到这里的事件映射表使用了`EVENT_MAP_BREAK`来结束。

在SOUI中推荐使用控件的name属性来标识一个控件（name属性是一个`wchar*`的字符串，使用name虽然在事件分发时采用字符串比较，较基于整数id属性的比较效率低一点，好处在于代码的可读性好），不同的页面中的控件如果出现相同的name该如何识别呢？

在SOUI中使用了一点小技巧：在事件处理对象中实现一个`oninit`函数，该函数在`maindlg`中处理`WM_INITDIALOG`时被调用，在`oninit`中保存了一个页面根节点的指针：

```c++
SWindow *m_pPageRoot;
```

在事件映射表的开始，我们采用`EVENT_CHECK_SENDER_ROOT(m_pPageRoot)`这个宏来识别那些来自本页面的事件。如果事件是来自其它页面则不处理。