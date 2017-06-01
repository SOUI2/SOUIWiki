原文链接：[《第二十九篇：使用SOUI的SMCListView控件》](http://www.cnblogs.com/setoutsoft/p/4863555.html)

列表控件是客户端应用最常用的控件之一。列表控件通常只负责显示数据，最多通知一下APP列表行的选中状态变化。

现在的UI经常要求程序猿在列表控件里不光显示内容，还要能和用户交互，显示动画等等，传统的列表控件对于这样的需求基本是无能为力了。

Android开发中很多界面都直接采用`ListView`实现，`ListView`中每一个Item中都可以容纳其它控件，这样的设计使得在表项中的交互和在主面板上交互一样简单。

虽然在列表项中容纳其它控件并不是什么新的思想，考虑到列表中的数据量是不确定的，如果给每一个表项的分配一个容器窗口，系统的内存占用及效率都成问题。

还好Andriod开源，简单看一下Android里`ListView`控件的源代码就可以发现，Android实现的`ListView`关键在于容器窗口的重用。

借鉴`Andriod ListView`控件的思想，在SOUI中也实现了对应的`ListView`控件。但是`ListView`只有一列，显示显示复杂内容问题不大，但是不能调整列宽等，和一个`ListCtrl`的功能还是有些差距。

多列列表和单列表最大的区别在于多了一个表头，核心的东西并没有区别，经过近3天的编码调试，终于完成了这个革命性的控件（至少我认为是Windows UI上革命性的）。

先看下效果：

![](assets/002/029-1496325573000.gif)

### SMCListView控件的使用：
#### XML配置：

要使用这个列表控件，首先应该在XML中定义该控件的位置及属性，参考下面摘自demo的代码：

```XML
<mclistview name="mclv_test" colorBkgnd="#ffffff" pos="10,10,-10,-10" headerHeight="30">
            <header align="center"  sortSkin="skin_lcex_header_arrow" itemSkin="skin_lcex_header" itemSwapEnable="1" fixWidth="0" font="underline:0,adding:-3" sortHeader="1" colorBkgnd="#ffffff" >
              <items>
                <item width="480">软件名称</item>
                <item width="95">软件评分</item>
                <item width="100">大小</item>
                <item width="100">安装时间</item>
                <item width="100">使用频率</item>
                <item width="100">操作</item>
              </items>
            </header>
            <template itemHeight="80" colorHover="#cccccc" colorSelected="#0000ff" id="30000">
              <window name="col1">
                <img name="img_icon" skin="skin_icon6" pos="10,8,@64,@64"/>
                <text name="txt_name" pos="[5,16" font="bold:1,adding:-1">火狐浏览器</text>
                <text name="txt_desc" pos="{0,36,-10,-10" font="bold:1,adding:-4" dotted="1">速度最快的浏览器</text>
                <text name="txt_index" pos="|0,|0" offset="-0.5,-0.5" font="adding:10" colorText="#ff000088">10</text>
              </window>
              <window name="col2">
                <ratingbar name="rating_score" starSkin="skin_star1" starNum="5" value="3.5" pos="10,16"  />
                <text name="txt_score" pos="15,36,50,-16" font="adding:-5"  >8.5分</text>
                <link pos="[5,36,@30,-16" cursor="hand" colorText="#1e78d5" href="www.163.com" font="adding:-5" >投票</link>
              </window>
              <window name="col3">
                <text name="txt_size" pos="0,26,-0,-26" font="adding:-4" align="center" >85.92M</text>
              </window>
              <window  name="col4">
                <text name="txt_installtime" pos="0,26,-0,-26" font="adding:-4" align="center" >2015-01-09</text>
              </window>
              <window name="col5">
                <text name="txt_usetime" pos="0,26,-0,-26" font="adding:-4" align="center" >今天</text>
                <animateimg pos="|0,|0" offset="-0.5,-0.5" skin="skin_busy" name="ani_test" tip="animateimg is used here" msgTransparent="0" />
              </window>
              <window name="col6">
                <imgbtn animate="1"  pos="|-35,|-14" font="adding:-3" align="center" skin="skin_install" name="btn_uninstall">卸载</imgbtn>
              </window>            
            </template>
          </mclistview>
```

`mclistview`有一个属性`headerHeight`，该属性定义表头的显示高度。

节点下有一个header控件，用来定义表头控件的样式，都很简单，自己看XML。

最关键的在于下面的template（模板）节点，该XML节点用来定义如何显示列表项。

模板内样式的定义其实并没有特别的规定，因为最后如何解析这个模板是由APP决定的，但推荐使用上面的样式：template节点下为每一列定义一个window节点，只需要指定一个name属性（当然也可以指定其它的窗口属性，布局属性无效）。在该window节点下可以定义任意的其它控件。

#### 代码编写：
和`listview`控件一样，`mclistview`也需要XML和代码配合才能正确显示数据。

要使用`mclistview`，首先需要实现一个数据适配器（`IMcAdapter`，继承自`SListView`中实现的`IAdapter`)，还是先看demo中的实现：

```c++
class CTestMcAdapterFix : public SMcAdapterBase
{
public:
struct SOFTINFO
{
    wchar_t * pszSkinName;
    wchar_t * pszName;
    wchar_t * pszDesc;
    float     fScore;
    DWORD     dwSize;
    wchar_t * pszInstallTime;
    wchar_t * pszUseTime;
};

static SOFTINFO s_info[];

public:
    CTestMcAdapterFix()
    {

    }

    virtual int getCount()
    {
        return 12340;
    }   

    SStringT getSizeText(DWORD dwSize)
    {
        int num1=dwSize/(1<<20);
        dwSize -= num1 *(1<<20);
        int num2 = dwSize*100/(1<<20);
        return SStringT().Format(_T("%d.%02dM"),num1,num2);
    }
    
    virtual void getView(int position, SWindow * pItem,pugi::xml_node xmlTemplate)
    {
        if(pItem->GetChildrenCount()==0)
        {
            pItem->InitFromXml(xmlTemplate);
        }
        int dataSize = 7;
        SOFTINFO *psi = s_info+position%dataSize;
        pItem->FindChildByName(L"img_icon")->SetAttribute(L"skin",psi->pszSkinName);
        pItem->FindChildByName(L"txt_name")->SetWindowText(S_CW2T(psi->pszName));
        pItem->FindChildByName(L"txt_desc")->SetWindowText(S_CW2T(psi->pszDesc));
        pItem->FindChildByName(L"txt_score")->SetWindowText(SStringT().Format(_T("%1.2f 分"),psi->fScore));
        pItem->FindChildByName(L"txt_installtime")->SetWindowText(S_CW2T(psi->pszInstallTime));
        pItem->FindChildByName(L"txt_usetime")->SetWindowText(S_CW2T(psi->pszUseTime));
        pItem->FindChildByName(L"txt_size")->SetWindowText(getSizeText(psi->dwSize));
        pItem->FindChildByName2<SRatingBar>(L"rating_score")->SetValue(psi->fScore/2);
        pItem->FindChildByName(L"txt_index")->SetWindowText(SStringT().Format(_T("第%d行"),position));
        
        SButton *pBtnUninstall = pItem->FindChildByName2<SButton>(L"btn_uninstall");
        pBtnUninstall->SetUserData(position);
        pBtnUninstall->GetEventSet()->subscribeEvent(EVT_CMD,Subscriber(&CTestMcAdapterFix::OnButtonClick,this));
    }

    bool OnButtonClick(EventArgs *pEvt)
    {
        SButton *pBtn = sobj_cast<SButton>(pEvt->sender);
        int iItem = pBtn->GetUserData();
        SMessageBox(NULL,SStringT().Format(_T("button of %d item was clicked"),iItem),_T("uninstall"),MB_OK);
        return true;
    }

    SStringW GetColumnName(int iCol) const{
        return SStringW().Format(L"col%d",iCol+1);
    }
    
    struct SORTCTX
    {
        int iCol;
        SHDSORTFLAG stFlag;
    };
    
    bool OnSort(int iCol,SHDSORTFLAG * stFlags,int nCols)
    {
        if(iCol==5) //最后一列“操作”不支持排序
            return false;
        
        SHDSORTFLAG stFlag = stFlags[iCol];
        switch(stFlag)
        {
            case ST_NULL:stFlag = ST_UP;break;
            case ST_DOWN:stFlag = ST_UP;break;
            case ST_UP:stFlag = ST_DOWN;break;
        }
        for(int i=0;i<nCols;i++)
        {
            stFlags[i]=ST_NULL;
        }
        stFlags[iCol]=stFlag;
        
        SORTCTX ctx={iCol,stFlag};
        qsort_s(s_info,7,sizeof(SOFTINFO),SortCmp,&ctx);
        return true;
    }
    
    static int __cdecl SortCmp(void *context,const void * p1,const void * p2)
    {
        SORTCTX *pctx = (SORTCTX*)context;
        const SOFTINFO *pSI1=(const SOFTINFO*)p1;
        const SOFTINFO *pSI2=(const SOFTINFO*)p2;
        int nRet =0;
        switch(pctx->iCol)
        {
            case 0://name
                nRet = wcscmp(pSI1->pszName,pSI2->pszName);
                break;
            case 1://score
                {
                    float fCmp = (pSI1->fScore - pSI2->fScore);
                    if(fabs(fCmp)<0.0000001) nRet = 0;
                    else if(fCmp>0.0f) nRet = 1;
                    else nRet = -1;
                }
                break;
            case 2://size
                nRet = (int)(pSI1->dwSize - pSI2->dwSize);
                break;
            case 3://install time
                nRet = wcscmp(pSI1->pszInstallTime,pSI2->pszInstallTime);
                break;
            case 4://user time
                nRet = wcscmp(pSI1->pszUseTime,pSI2->pszUseTime);
                break;

        }
        if(pctx->stFlag == ST_UP)
            nRet = -nRet;
        return nRet;
    }
};

CTestMcAdapterFix::SOFTINFO CTestMcAdapterFix::s_info[] =
{
    {
        L"skin_icon1",
        L"鲁大师",
        L"鲁大师是一款专业的硬件检测，驱动安装工具",
        5.4f,
        15*(1<<20),
        L"2015-8-5",
        L"今天"
    },
    {
        L"skin_icon2",
        L"PhotoShop",
        L"强大的图片处理工具",
        9.0f,
        150*(1<<20),
        L"2015-8-5",
        L"今天"
    },
    {
        L"skin_icon3",
        L"QQ7.0",
        L"腾讯公司出品的即时聊天工具",
        8.0f,
        40*(1<<20),
        L"2015-8-5",
        L"今天"
    },
    {
        L"skin_icon4",
        L"Visual Studio 2008",
        L"Microsoft公司的程序开发套件",
        9.0f,
        40*(1<<20),
        L"2015-8-5",
        L"今天"
    },
    {
        L"skin_icon5",
        L"YY8",
        L"YY语音",
        9.0f,
        20*(1<<20),
        L"2015-8-5",
        L"今天"
    },
    {
        L"skin_icon6",
        L"火狐浏览器",
        L"速度最快的浏览器",
        8.5f,
        35*(1<<20),
        L"2015-8-5",
        L"今天"
    },
    {
        L"skin_icon7",
        L"迅雷",
        L"迅雷下载软件",
        7.3f,
        17*(1<<20),
        L"2015-8-5",
        L"今天"
    }
};
```

注意`CTestMcAdapterFix::getView`虚函数，上面提到的`template`会通过该函数的参数 `pugi::xml_node xmlTemplate` 传递过来。

在`getView`中，首先需要判断表项容器的子窗口是不是已经被初始化过，如果没有就执行`InitFromXml`如下：

```c++
        if(pItem->GetChildrenCount()==0)
        {
            pItem->InitFromXml(xmlTemplate);
        }
```

在子窗口初始化完成后，还需要从数据表中获取对应项的数据填充到控件中显示：

```c++
int dataSize = 7;
        SOFTINFO *psi = s_info+position%dataSize;
        pItem->FindChildByName(L"img_icon")->SetAttribute(L"skin",psi->pszSkinName);
        pItem->FindChildByName(L"txt_name")->SetWindowText(S_CW2T(psi->pszName));
        pItem->FindChildByName(L"txt_desc")->SetWindowText(S_CW2T(psi->pszDesc));
        pItem->FindChildByName(L"txt_score")->SetWindowText(SStringT().Format(_T("%1.2f 分"),psi->fScore));
        pItem->FindChildByName(L"txt_installtime")->SetWindowText(S_CW2T(psi->pszInstallTime));
        pItem->FindChildByName(L"txt_usetime")->SetWindowText(S_CW2T(psi->pszUseTime));
        pItem->FindChildByName(L"txt_size")->SetWindowText(getSizeText(psi->dwSize));
        pItem->FindChildByName2<SRatingBar>(L"rating_score")->SetValue(psi->fScore/2);
        pItem->FindChildByName(L"txt_index")->SetWindowText(SStringT().Format(_T("第%d行"),position));
```

和主面板上的控件响应不同，要响应表项中控件的事件，没有事件映射表可以使用，可能在`IMcAdapter`的实现中使用控件的`GetEventSet()->subscribeEvent`方法来响应：

```c++
        SButton *pBtnUninstall = pItem->FindChildByName2<SButton>(L"btn_uninstall");
        pBtnUninstall->SetUserData(position);
        pBtnUninstall->GetEventSet()->subscribeEvent(EVT_CMD,Subscriber(&CTestMcAdapterFix::OnButtonClick,this));
```

除了`getView`这个方法外，相对于`IAdapter，IMcAdapter`还需要实现另外两个非常重要的方法：

```c++
interface IMcAdapter : public IAdapter
    {
        //获取列名
        virtual SStringW GetColumnName(int iCol) const PURE;
        //排序接口
        // int iCol:排序列
        // SHDSORTFLAG * stFlags [in, out]:当前列排序标志
        // int nCols:总列数,stFlags数组长度
        virtual bool OnSort(int iCol,SHDSORTFLAG * stFlags,int nCols) PURE;
    };
```

实现`GetColumnName`方法来获取每一列对应的子窗口名称，`SMcListView`通过它来确实template中的子窗口哪一个应该显示在什么位置，返回在`template`中定义的子节点的name即可。

实现`OnSort`来处理表头点击事件，以确实如何对数据排序。

至此，这个超级列表控件的使用就完成了。
