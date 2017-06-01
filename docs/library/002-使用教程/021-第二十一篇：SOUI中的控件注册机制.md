原文链接：[《第二十一篇：SOUI中的控件注册机制》](http://www.cnblogs.com/setoutsoft/p/4280997.html)

Win32编程中，用户需要一个新控件时，需要向系统注册一个新的控件类型。注册以后，调用`::CreateWindow`时才能根据标识控件类型的字符串创建出一个新的控件窗口对象。

为了能够从XML描述的字符串中创建出需要的控件对象，和Win32类似，在SOUI中要创建一个新的控件也同样需要向SOUI系统注册新的控件类。

从`demo.cpp`的main中我们可以看到类似如下的控件注册控件的代码：

```c++
//向SApplication系统中注册由外部扩展的控件及SkinObj类
        SWkeLoader wkeLoader;
        if(wkeLoader.Init(_T("wke.dll")))        
        {
            theApp->RegisterWndFactory(TplSWindowFactory<SWkeWebkit>());//注册WKE浏览器
        }
        theApp->RegisterWndFactory(TplSWindowFactory<SGifPlayer>());//注册GIFPlayer
        theApp->RegisterSkinFactory(TplSkinFactory<SSkinGif>());//注册SkinGif
        theApp->RegisterSkinFactory(TplSkinFactory<SSkinAPNG>());//注册SSkinAPNG
        theApp->RegisterSkinFactory(TplSkinFactory<SSkinVScrollbar>());//注册纵向滚动条皮肤

        theApp->RegisterWndFactory(TplSWindowFactory<SIPAddressCtrl>());//注册IP控件
        theApp->RegisterWndFactory(TplSWindowFactory<SPropertyGrid>());//注册属性表控件
        theApp->RegisterWndFactory(TplSWindowFactory<SChromeTabCtrl>());//注册ChromeTabCtrl
        theApp->RegisterWndFactory(TplSWindowFactory<SIECtrl>());//注册IECtrl
        theApp->RegisterWndFactory(TplSWindowFactory<SChatEdit>());//注册ChatEdit
        theApp->RegisterWndFactory(TplSWindowFactory<SScrollText>());//注册SScrollText
        
        if(SUCCEEDED(CUiAnimation::Init()))
        {
            theApp->RegisterWndFactory(TplSWindowFactory<SUiAnimationWnd>());//注册动画控件
        }
        theApp->RegisterWndFactory(TplSWindowFactory<SFlyWnd>());//注册飞行动画控件
        theApp->RegisterWndFactory(TplSWindowFactory<SFadeFrame>());//注册渐显隐动画控件
        theApp->RegisterWndFactory(TplSWindowFactory<SRadioBox2>());//注册渐显隐动画控件
```

上面代码中即有新皮肤对象的注册也有窗口控件的注册，这种注册控件方式看起来有点怪异，但使用起来也还算是简单，只需要一行代码（实际上是从著名的游戏GUI CEGUI借鉴过来的）。

以注册窗口控件为例，下面我来解释一下为什么会有这样一种控件注册方式。

先看看`TplSWindowFactory`模板类的实现：

```c++
class SWindowFactory
    {
    public:
        virtual ~SWindowFactory() {}
        virtual SWindow* NewWindow() = 0;
        virtual LPCWSTR SWindowBaseName()=0;

        virtual const SStringW & getWindowType()=0;

        virtual SWindowFactory* Clone() const =0;
    };

    template <typename T>
    class TplSWindowFactory : public SWindowFactory
    {
    public:
        //! Default constructor.
        TplSWindowFactory():m_strTypeName(T::GetClassName())
        {
        }

        LPCWSTR SWindowBaseName(){return T::BaseClassName();}

        // Implement WindowFactory interface
        virtual SWindow* NewWindow()
        {
            return new T;
        }

        virtual const SStringW & getWindowType()
        {
            return m_strTypeName;
        }

        virtual SWindowFactory* Clone() const 
        {
            return new TplSWindowFactory();
        }
    protected:
        SStringW m_strTypeName;
    };
```

`TplSWindowFactory`从`SWindowFactory`派生，自动为模板参数中指定的类型生成一个SWindowFactory对象。
`SWindowFactory`有什么用呢？`SWindowFactory`最核心的用途就在于它的方法：`SWindow * SWindowFactory::NewWindow();`
该方法说来很简单，不过是根据不同的字符串从堆上分配一个SWindow对象，但是在C++中我们需要满足一条基本原则：内存在哪个模块中分配（从哪个模块的堆上分配）就必须被哪个模块释放。
在SOUI中，通常情况下控件都是由SOUI.DLL这个模块来分配内存的，也就是说`SWindow * SWindowFactory::NewWindow()`只会在SOUI模块中调用。

那么生成的对象在哪里释放呢？
这就需要看一下SWindow或者`SSkinObjBase`这两个类的实现了。

```c++
class SOUI_EXP SWindow : public SObject
        , public SMsgHandleState
        , public TObjRefImpl2<IObjRef,SWindow>
    {
        SOUI_CLASS_NAME(SWindow, L"window")
        //....
    };

    class SOUI_EXP SSkinObjBase : public TObjRefImpl<ISkinObj>
    {
       //......
    };
```

SWindow和`SSkinObjBase`实际上都派生自`TObjRefImpl`这个模板类。下面看一下这个类的实现:

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
    virtual void AddRef()
    {
        InterlockedIncrement(&m_cRef);
    }

    //!释放引用
    /*!
    */
    virtual void Release()
    {
        InterlockedDecrement(&m_cRef);
        if(m_cRef==0)
        {
            OnFinalRelease();
        }
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

`TObjRefImpl`一个重要的虚函数是`void OnFinalRelease(){delete this;}`

由于SWindow及`SSkinObjBase`是在SOUI模块中实现的，因此派生自这两个类的新的控件类及皮肤类最后都将在SOUI模块中被释放，从而保证了对象内存的分配、释放在一个模块。
这也就是说不管是SOUI模块内实现的控件还是在应用层扩展的控件一般情况下都应该向SOUI系统注册并经由SOUI内核来实现对象的分配与释放。

那么问题来了，是不是所有新控件都必须向系统注册呢？
当然不是的，注意`OnFinalRelease`是一个虚函数，只需要在新的控件类中增加一个

`void OnFinalRelease(){delete this;}`

就可以把控件内存的释放转移到应用层的模块了。如此你就可以在自己的模块中直接使用new来创建新控件了。(可以参考属性表控件中部分子控件的实现）

