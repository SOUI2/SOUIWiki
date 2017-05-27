原文链接：[《第十篇：扩展SOUI的控件及绘图对象(ISkinObj)》](http://www.cnblogs.com/setoutsoft/p/3931877.html)

尽管SOUI已经内置了大部分常用的控件，很显然内置控件很难满足各种应用的形式各异的需求。

因此只有提供足够的扩展性才能满足真实应用场景。

除了将系统尽可能的组件化外，SOUI在控件自绘(`SWindow`)及绘图对象(`ISkinObj`)两个方面提供用户扩展。

### 绘图对象(ISkinObj)的扩展
系统内置了如`SSkinImgList, SSkinImgFrame, SSkinScrollbar`等绘图对象，在窗口中通过引用这些绘图对象可以绘制出不同的预定义图形图象（如按钮，滚动条，九宫格等）。

实际上用户可以实现任意的绘图对象并把它们注册到系统里，以便在XML及代码中使用。

下面先看一下实现一个`ISkinObj`需要实现哪些接口：
```c++
/**
    * @struct     ISkinObj
    * @brief      Skin 对象
    * 
    * Describe
    */
    class SOUI_EXP ISkinObj : public SObject,public TObjRefImpl2<IObjRef,ISkinObj>
    {
    public:
        ISkinObj()
        {
        }
        virtual ~ISkinObj()
        {
        }

        /**
         * Draw
         * @brief    将this绘制到RenderTarget上去
         * @param    IRenderTarget * pRT --  绘制用的RenderTarget
         * @param    LPCRECT rcDraw --  绘制位置
         * @param    DWORD dwState --  绘制状态
         * @param    BYTE byAlpha --  透明度
         * @return   void
         * Describe  
         */    
        virtual void Draw(IRenderTarget *pRT, LPCRECT rcDraw, DWORD dwState,BYTE byAlpha=0xFF)=0;

        /**
         * GetSkinSize
         * @brief    获得Skin的默认大小
         * @return   SIZE -- Skin的默认大小
         * Describe  派生类应该根据skin的特点实现该接口
         */    
        virtual SIZE GetSkinSize()
        {
            SIZE ret = {0, 0};

            return ret;
        }

        /**
         * IgnoreState
         * @brief    查询skin是否有状态信息
         * @return   BOOL -- true有状态信息
         * Describe  
         */    
        virtual BOOL IgnoreState()
        {
            return TRUE;
        }

        /**
         * GetStates
         * @brief    获得skin对象包含的状态数量
         * @return   int -- 状态数量
         * Describe  默认为1
         */    
        virtual int GetStates()
        {
            return 1;
        }
    };
```

`ISkinObj`是一个派生自`SObject`及`TObjRefImpl2<IObjRef,ISkinObj>`的类，提供了几个状态查询相关的接口，也提供了一个Draw接口来在IRenderTarget上绘制该绘制对象。

注意的是，这些接口中只有Draw接口是纯虚接口。

在它的基类中，`SOjbect`使得`ISkinObj`可以方便的从XML配置文件中初始化，而`TObjRefImpl2<IObjRef,ISkinObj>`则提供引用计数的实现。

内置的`ISkinObj`不支持显示GIF图片，以显示GIF图象为例来分析如何扩展绘图对象来支持GIF图片显示。

对象定义（`trunk\controls.extend\gif\SSkinGif.h`）

```c++
namespace SOUI
{
    class SGifFrame
    {
    public:
        CAutoRefPtr<IBitmap> pBmp;
        int                  nDelay;
    };

    /**
    * @class     SSkinGif
    * @brief     GIF图片加载及显示对象
    * 
    * Describe
    */
    class SSkinGif : public ISkinObj
    {
        SOUI_CLASS_NAME(SSkinGif, L"gif")
    public:
        SSkinGif():m_nFrames(0),m_iFrame(0),m_pFrames(NULL)
        {

        }
        
        //初始化GDI+环境，由于这里需要使用GDI+来解码GIF文件格式
        static BOOL Gdiplus_Startup();
        //退出GDI+环境
        static void Gdiplus_Shutdown();

        virtual ~SSkinGif()
        {
            if(m_pFrames) delete [] m_pFrames;
        }

        /**
         * Draw
         * @brief    绘制指定帧的GIF图
         * @param    IRenderTarget * pRT --  绘制目标
         * @param    LPCRECT rcDraw --  绘制范围
         * @param    DWORD dwState --  绘制状态，这里被解释为帧号
         * @param    BYTE byAlpha --  透明度
         * @return   void
         * Describe  
         */    
        virtual void Draw(IRenderTarget *pRT, LPCRECT rcDraw, DWORD dwState,BYTE byAlpha=0xFF);

        /**
         * GetStates
         * @brief    获得GIF帧数
         * @return   int -- 帧数
         * Describe  
         */    
        virtual int GetStates(){return m_nFrames;}

        /**
         * GetSkinSize
         * @brief    获得图片大小
         * @return   SIZE -- 图片大小
         * Describe  
         */    
        virtual SIZE GetSkinSize()
        {
            SIZE sz={0};
            if(m_nFrames>0 && m_pFrames)
            {
                sz=m_pFrames[0].pBmp->Size();
            }
            return sz;
        }

        /**
         * GetFrameDelay
         * @brief    获得指定帧的显示时间
         * @param    int iFrame --  帧号,为-1时代表获得当前帧的延时
         * @return   long -- 延时时间(*10ms)
         * Describe  
         */    
        long GetFrameDelay(int iFrame=-1);

        /**
         * ActiveNextFrame
         * @brief    激活下一帧
         * @return   void 
         * Describe  
         */    
        void ActiveNextFrame();

        /**
         * SelectActiveFrame
         * @brief    激活指定帧
         * @param    int iFrame --  帧号
         * @return   void
         * Describe  
         */    
        void SelectActiveFrame(int iFrame);
        
        /**
         * LoadFromFile
         * @brief    从文件加载GIF
         * @param    LPCTSTR pszFileName --  文件名
         * @return   int -- GIF帧数，0-失败
         * Describe  
         */    
        int LoadFromFile(LPCTSTR pszFileName);

        /**
         * LoadFromMemory
         * @brief    从内存加载GIF
         * @param    LPVOID pBits --  内存地址
         * @param    size_t szData --  内存数据长度
         * @return   int -- GIF帧数，0-失败
         * Describe  
         */    
        int LoadFromMemory(LPVOID pBits,size_t szData);

        SOUI_ATTRS_BEGIN()
            ATTR_CUSTOM(L"src",OnAttrSrc)   //XML文件中指定的图片资源名,(type:name)
        SOUI_ATTRS_END()
    protected:
        LRESULT OnAttrSrc(const SStringW &strValue,BOOL bLoading);
        int LoadFromGdipImage(Gdiplus::Bitmap * pImg);
        int m_nFrames;
        int m_iFrame;

        SGifFrame * m_pFrames;
    };
}//end of name space SOUI
```

对象注册：

```c++
theApp->RegisterSkinFactory(TplSkinFactory<SSkinGif>());//注册SkinGif
```

对象的使用（`trunk\demo\uires\xml\dlg_main.xml`）：

```xml
  <skin>
    <!--局部skin对象-->
    <gif name="gif_horse" src="gif:gif_horse"/>
    <gif name="gif_penguin" src="gif:gif_penguin"/>
  </skin>
```

这里的gif标签与`SSkinGif`类中的宏`SOUI_CLASS_NAME(SSkinGif,L"gif")`中的“gif”是匹配的。

到此，在布局XML及程序中都可以获得这个`SSkinGif`对象的指针了。

### 控件的扩展
控件的扩展和绘图对象的扩展套路类似，也是先从系统提供的基础类派生，再注册到系统，最后再XML或者代码中使用。

和绘图对象不同在于，控件是UI，需要处理各种UI相关的消息以及向程序发出各种控件特有的事件。

同样我们还是以GIF显示的控件为例（`trunk\controls.extend\gif\SGifPlayer.h`）：

```c++
namespace SOUI
{

    /**
    * @class     SGifPlayer
    * @brief     GIF图片显示控件
    * 
    * Describe
    */
    class SGifPlayer : public SWindow
    {
        SOUI_CLASS_NAME(SGifPlayer, L"gifplayer")   //定义GIF控件在XM加的标签
    public:
        SGifPlayer();
        ~SGifPlayer();

        /**
         * PlayGifFile
         * @brief    在控件中播放一个GIF图片文件
         * @param    LPCTSTR pszFileName --  文件名
         * @return   BOOL -- true:成功
         * Describe  
         */    
        BOOL PlayGifFile(LPCTSTR pszFileName);

    protected://SWindow的虚函数
        virtual CSize GetDesiredSize(LPRECT pRcContainer);

    public://属性处理
        SOUI_ATTRS_BEGIN()        
            ATTR_CUSTOM(L"skin", OnAttrGif) //为控件提供一个skin属性，用来接收SSkinObj对象的name
        SOUI_ATTRS_END()
    protected:
        HRESULT OnAttrGif(const SStringW & strValue, BOOL bLoading);

    protected://消息处理，SOUI控件的消息处理和WTL，MFC很相似，采用相似的映射表，相同或者相似的消息映射宏
        
        /**
         * OnPaint
         * @brief    窗口绘制消息响应函数
         * @param    IRenderTarget * pRT --  绘制目标
         * @return   void
         * Describe  注意这里的参数是IRenderTarget *,而不是WTL中使用的HDC，同时消息映射宏也变为MSG_WM_PAINT_EX
         */    
        void OnPaint(IRenderTarget *pRT);

        /**
         * OnTimer
         * @brief    SOUI窗口的定时器处理函数
         * @param    char cTimerID --  定时器ID，范围从0-127。
         * @return   void
         * Describe  SOUI控件的定时器是Host窗口定时器ID的分解，以方便所有的控件都通过Host获得定时器的分发。
         *           注意使用MSG_WM_TIMER_EX来映射该消息。定时器使用SWindow::SetTimer及SWindow::KillTimer来创建及释放。
         *           如果该定时器ID范围不能满足要求，可以使用SWindow::SetTimer2来创建。
         */    
        void OnTimer(char cTimerID);

        /**
         * OnShowWindow
         * @brief    处理窗口显示消息
         * @param    BOOL bShow --  true:显示
         * @param    UINT nStatus --  显示原因
         * @return   void 
         * Describe  参考MSDN的WM_SHOWWINDOW消息
         */    
        void OnShowWindow(BOOL bShow, UINT nStatus);

        //SOUI控件消息映射表
        SOUI_MSG_MAP_BEGIN()    
            MSG_WM_TIMER_EX(OnTimer)    //定时器消息
            MSG_WM_PAINT_EX(OnPaint)    //窗口绘制消息
            MSG_WM_SHOWWINDOW(OnShowWindow)//窗口显示状态消息
        SOUI_MSG_MAP_END()    

    private:
        SSkinGif *m_pgif;
        int    m_iCurFrame;
    };

}
```

实现了该类以后，在WinMain中注册：

```c++
 theApp->RegisterWndFactory(TplSWindowFactory<SGifPlayer>());//注册GIFPlayer
```

我们可以在布局XML中创建GIF控件控件并显示了(`trunk\demo\uires\xml\dlg_main.xml`）：

```xml
        <gifplayer pos="10,10" skin="gif_horse" name="giftest" cursor="ANI_ARROW"/>
        <button width="250" height="30" name="btnSelectGif">load gif file</button>
        <gifplayer pos="10,150" skin="gif_penguin"/>
```

上面的`gifplayer`节点即表示从XML中创建两个`SGifPlayer`类对象。

而`gifplayer`对象的`skin`属性则引用前面定义的`SSkinGif`对象。

备注：

实际上更多扩展技巧可以参考系统内置的控件的实现。内置控件与扩展控件唯一的区别就在于由谁实现将控件向系统注册。

内置控件在SOUI内核初始化的时候自动注册，而扩展控件则需要手动增加一行注册代码。

### 效果预览
![](assets/002/010-1495897765000.png)