原文链接：[《第二十二篇：在SOUI中使用代码向窗口中插入子窗口》](http://www.cnblogs.com/setoutsoft/p/4301928.html)

使用SOUI开发客户端UI程序，通常也推荐使用XML代码来创建窗口，这样创建的窗口使用方便，当窗口大小改变时，内部的子窗口也更容易协同变化。

但是最近不断有网友咨询如何使用代码来创建SOUI子窗口，特此在这里统一解答。

要回答这个问题，首先要了解SOUI窗口创建及布局的流程。

先从swnd.cpp里抄一段创建子窗口的代码：

```c++
BOOL SWindow::CreateChildren(pugi::xml_node xmlNode)
    {
        TestMainThread();
        for (pugi::xml_node xmlChild=xmlNode.first_child(); xmlChild; xmlChild=xmlChild.next_sibling())
        {
            if(xmlChild.type() != pugi::node_element) continue;

            if(_wcsicmp(xmlChild.name(),KLabelInclude)==0)
            {//在窗口布局中支持include标签
                SStringW strSrc = S_CW2T(xmlChild.attribute(L"src").value());
                pugi::xml_document xmlDoc;
                SStringTList strLst;

                if(2 == ParseResID(strSrc,strLst))
                {
                    LOADXML(xmlDoc,strLst[1],strLst[0]);
                }else
                {
                    LOADXML(xmlDoc,strLst[0],RT_LAYOUT);
                }
                if(xmlDoc)
                {
                    CreateChildren(xmlDoc.child(KLabelInclude));
                }else
                {
                    SASSERT(FALSE);
                }
            }else if(!xmlChild.get_userdata())//通过userdata来标记一个节点是否可以忽略
            {
                SWindow *pChild = SApplication::getSingleton().CreateWindowByName(xmlChild.name());
                if(pChild)
                {
                    InsertChild(pChild);
                    pChild->InitFromXml(xmlChild);
                }
            }
        }
        return TRUE;
    }
```

这个函数的功能是为this从XML中创建它的子窗口，主要注意代码中红色部分。

其中第30行是从SOUI的窗口类厂根据XML结点名new出一个窗口对象。

第33行把创建出来的窗口插入到this的子窗口链表里去。

而第34行的功能是从子窗口对应的XML结点初始化子窗口属性。

很多网友以为只要完成上面步骤就应该可以显示窗口了，但结果确大失所望。

SOUI一个重要特点就是能够自动布局，这个过程的秘密就在于SWindow::OnRelayout方法。

```c++
void SWindow::OnRelayout(const CRect &rcOld, const CRect & rcNew)
    {
        SWindow *pParent= GetParent();
        if(pParent)
        {
            pParent->InvalidateRect(rcOld);
            pParent->InvalidateRect(rcNew);
        }else
        {
            InvalidateRect(m_rcWindow);
        }

        SSendMessage(WM_NCCALCSIZE);//计算非客户区大小

        UpdateChildrenPosition();   //更新子窗口位置

        CRect rcClient;
        GetClientRect(&rcClient);
        SSendMessage(WM_SIZE,0,MAKELPARAM(rcClient.Width(),rcClient.Height()));
    }
```

当this窗口位置改变后都会进入OnRelayout方法。

注意函数第15行：UpdateChildrenPosition()；这个调用的功能就是将this的所有子控件按照控件中定义的布局属性来自动布局。

要实现窗口的布局，除了调用父窗口的UpdateChildrenPosition()方法外，当然也可以使用SWindow::Move方法直接修改窗口位置。

看到这里大家应该已经明白是什么原因了。

当然上述方法是SOUI中使用的窗口创建及布局方法，具体到应用程序中使用代码创建控件还有一个地方需要注意。

关键的问题是在SOUI系统中编译默认使用MT（静态链接）方式来链接CRT。

MT方式编译时使用CRT分配内存是内存是属性调用的模块（DLL）的，内存的释放也因此必须在该模块内执行。

如果用户直接使用new来分配一个窗口对象，并把它插入到SOUI窗口链表中，在窗口释放的时机是在SWindow::OnFinalRelease()中（实际是基类TObjRefImpl2<IObjRef,SWindow>的方法)。

SWindow的代码段是编译在soui.dll中，因此默认执行内存释放的代码是在soui.dll中，从而导致程序崩溃。

要解决这个问题有两种方法：

对于系统控件，用户应该使用SApplication::getSingleton().CreateWindowByName(xmlChild.name());来创建窗口对象。

而对于用户自己派生实现的扩展窗口类并没有向SOUI的窗口类厂注册时，只能使用new方法来创建窗口对象。注意SWindow的基类：TObjRefImpl2<IObjRef,SWindow>

```c++
template<class T>
class TObjRefImpl :  public T
{
public:
    TObjRefImpl():m_cRef(1)
    {
    }

    virtual ~TObjRefImpl(){
    }

    //!添加引用
    /*!
    */
    virtual long AddRef()
    {
        return InterlockedIncrement(&m_cRef);
    }

    //!释放引用
    /*!
    */
    virtual long Release()
    {
        long lRet = InterlockedDecrement(&m_cRef);
        if(lRet==0)
        {
            OnFinalRelease();
        }
        return lRet;
    }

    //!释放对象
    /*!
    */
    virtual void OnFinalRelease()
    {
        delete this;
    }
protected:
    volatile LONG m_cRef;
};

template<class T,class T2>
class TObjRefImpl2 :  public TObjRefImpl<T>
{
public:
    virtual void OnFinalRelease()
    {
        delete static_cast<T2*>(this);
    }
};
```

注意代码中的OnFinalRelease，它是一个虚方法。因此对于使用new创建的窗口对象，只需要在窗口类中抄一段代码如下即可：

```c++
class myctrl : public SWindow
{
SOUI_CLASS_NAME(myctrl,L"myctrl")
public:
//...
virtual void OnFinalRelease() {delete this;}
//...
};
```

感谢大家的支持！