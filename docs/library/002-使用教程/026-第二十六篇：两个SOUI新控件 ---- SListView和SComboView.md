原文链接：[《第二十六篇：两个SOUI新控件 ---- SListView和SComboView（借用Andorid的设计）》](http://www.cnblogs.com/setoutsoft/p/4691790.html)

SOUI原来实现的`SListBoxEx`的效率一直是我对SOUI不太满意的地方。包括后来网友实现的`SListCtrlEx`。

这类控件为每一个列表项创建一个SWindow来容纳数据，当数据量比较大（10000+)时，一方面内存消耗会很严重；另一方面列表数据初始化也需要大量的时间。

今年开始转型做Android开发。大家都知道Android开发APP和PC上开发APP相比要简单很多，其中我个人体会最深的就是Android的ListView控件。

在Android中，`ListView`中列表项的显示采用控件+适配器(`Adapter`)的模式，也就是所谓的MVC模式。一个表项在需要显示的时候才会把数据加载到View里去，当这个表项被隐藏起来以后，容纳该表项的容器(View)则自动被加入到`ListView`中保存的一个容器回收列表中。需要显示一个新表项时首先去回收站里查找是否存在指定类型的容器，存在则自动复用。

基本思想如上，当然实际实现还使用了很多技巧。通过上述机制，可以有效解决`ListView`显示大量数据的问题。

本来也一直想重写SOUI的`ListBox`, 这段时间正好项目需要，抽出时间模仿了一个，效果不错。

先看看效果：

![](assets/002/026-1496324134000.gif)

![](assets/002/026-1496324144000.gif)

第一张图是一个加载50000行的`SListView`控件，第二个图是演示使用`SComboView`来做用户登陆界面。

要在SOUI中使用`SListView`，我们首先需要自己实现一上数据填充的`Adapter`:

```c++
class CTestAdapter : public SAdapterBase
{
protected:
    SListView * m_pOwenr;
public:
    CTestAdapter(SListView *pOwner):m_pOwenr(pOwner)
    {

    }
    virtual int getCount()
    {
        return 50000;
    }   

    virtual void getView(int position, SWindow * pItem)
    {
        if(pItem->GetChildrenCount()==0)
        {
            pItem->InitFromXml(m_pOwenr->GetTemplate());
        }
        SAnimateImgWnd *pAni = pItem->FindChildByName2<SAnimateImgWnd>(L"ani_test");

        SButton *pBtn = pItem->FindChildByName2<SButton>(L"btn_test");
        pBtn->SetWindowText(SStringW().Format(L"button %d",position));
        pBtn->SetUserData(position);
        pBtn->GetEventSet()->subscribeEvent(EVT_CMD,Subscriber(&CTestAdapter::OnButtonClick,this));
    }

    bool OnButtonClick(EventArgs *pEvt)
    {
        SButton *pBtn = sobj_cast<SButton>(pEvt->sender);
        int iItem = pBtn->GetUserData();
        SMessageBox(NULL,SStringT().Format(_T("button of %d item was clicked"),iItem),_T("haha"),MB_OK);
        return true;
    }
};
```

这里最关键的就是实现`IAdapter`中的`getView`方法。

和Android的`ListView`不同，SOUI中使用条件`pItem->GetChildrenCount()==0`来判断一个容器是否是被复用。

当`pItem->GetChildrenCount()==0`时代表该容器还没有被初始化，需要自己从XML模板中初始化容器。XML模板可以自己自己定义的任意符合SOUI布局语法的数据。

容器初始化完成后就可以向里面填充数据，也可以向控件连接响应函数了（`subscribeEvent`）。

在UI创建完成后需要在代码中把这个`Adapter`交给`SListView`：

```c++
LRESULT CMainDlg::OnInitDialog( HWND hWnd, LPARAM lParam )
{
    //....
    
    SListView *pLstView = FindChildByName2<SListView>("lv_test");
    if(pLstView)
    {
        CTestAdapter *pAdapter = new CTestAdapter(pLstView);
        pLstView->SetAdapter(pAdapter);
        pAdapter->Release();
    }

    return 0;
}
```

第二个界面是演示`SComboView`的。`SComboView`的用户和`SListView`基本一样，具体看代码。