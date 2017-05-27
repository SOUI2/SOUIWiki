原文链接：[《第八篇：SOUI中控件事件的响应》](http://www.cnblogs.com/setoutsoft/p/3930419.html)

SOUI中提供了大部分常用的win32标准控件的实现，如`pushbutton, checkbox, radiobox, edit, richedit, listbox, combobox, treectrl, listctrl (report), hotkeyctrl`等。

大部分控件在接收用户输入后，会发生状态的改变，并以事件的形式传递给UI的所有者。

在SOUI中提供了两种处理事件的方式：

#### 1.在SHostWnd的派生类中重载

```c++
virtual BOOL SHostWnd::_HandleEvent(SOUI::EventArgs *pEvt){return FALSE;}
```

为了更方便的处理事件，SOUI提供了一组宏来构造这个事件处理函数，从而提供一种类似消息映射的事件处理形式。

如demo的`CMainDlg`中的实现：

```c++
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
```

上面的`EVENT_MAP_BEGIN()`和`EVENT_MAP_END()`结合构造出一个`_HandleEvent`函数的实现，具体可以自己展开这两个宏查看代码。

同时SOUI也提供了一组解析`SOUI::EventArgs *pEvt`的宏，如上例中的`EVENT_NAME_COMMAND, EVENT_ID_COMMAND`等。

帮助用户直接从控件的name或者ID属性映射到消息响应函数。

这种事件响应方式最大的好处是能够集中处理事件的分发，方便阅读代码，同时也和传统的MFC，WTL的编程风格类似，降低用户的学习成本。

#### 2.采用事件订阅的方式响应控件事件
虽然事件映射表提供了一种简单有效的事件响应机制，由于事件映射表是一种编译期形成的静态的映射表，对于在运行期动态创建的控件的事件响应就无能为力了。

在MFC中，程序员通过要重载窗口类的`DefWindowProc`来处理运行期间动态创建的控件发来的消息。

这种方式灵活性够了，但是不够优雅，要在一个函数里做大量的swich分枝，导致这个处理函数很难维护。

设计模式里的观察者模式可以比较好的解决这个问题。

为些在SOUI中我提供了一种事件订阅的事件处理模式。

我们先看一下demo中怎样处理列表控件的表头点击来执行排序操作：

```c++
void CMainDlg::InitListCtrl()
{
    //找到列表控件
    SListCtrl *pList=FindChildByName2<SListCtrl>(L"lc_test");
    if(pList)
    {
        //列表控件的唯一子控件即为表头控件
        SWindow *pHeader=pList->GetWindow(GSW_FIRSTCHILD);
        //向表头控件订阅表明点击事件，并把它和OnListHeaderClick函数相连。
        pHeader->GetEventSet()->subscribeEvent(EVT_HEADER_CLICK,Subscriber(&CMainDlg::OnListHeaderClick,this));
        //省略列表初始化代码  
    }
}

//表头点击事件处理函数
bool CMainDlg::OnListHeaderClick(EventArgs *pEvtBase)
{
    //事件对象强制转换
    EventHeaderClick *pEvt =(EventHeaderClick*)pEvtBase;
    SHeaderCtrl *pHeader=(SHeaderCtrl*)pEvt->sender;
    //从表头控件获得列表控件对象
    SListCtrl *pList= (SListCtrl*)pHeader->GetParent();
    //列表数据排序
    SHDITEM hditem;
    hditem.mask=SHDI_ORDER;
    pHeader->GetItem(pEvt->iItem,&hditem);
    pList->SortItems(funCmpare,&hditem.iOrder);
    return true;
}
```

通过事件订阅可以在运行时方便的将一个控件的事件关联到一个处理函数上，当然也可以随时取消订阅。

同时事件订阅也是在脚本中响应控件事件的唯一方式（关于在SOUI中使用LUA脚本将在后续讲解）。