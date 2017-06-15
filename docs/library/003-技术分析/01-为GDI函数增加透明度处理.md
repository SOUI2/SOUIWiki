原文链接：[《为GDI函数增加透明度处理》](http://www.cnblogs.com/setoutsoft/p/4086051.html)

用户对客户端的UI的要求越来越高，采用`alpha`通道对前景背景做混合是提高UI质量的重要手段。

UI开发离不开`GDI`，然后要用传统的`GDI`函数来处理`alpha`通道通常是一个恶梦：虽然有`AlphaBlend`这个`API`可以做`alpha`混合，但是前提必须是操作的`DC`中的位图有`alpha`通道的数据，问题的关键在于`GDI`函数在操作的地方会把原来的`alpha`通道清空。

- 使用`GDI`做`alpha`混合还要增加透明度关键要解决2个问题：
  - 需要把内容画到一个临时位图上，同时保护好`alpha`通道。
  - 在于把临时位图的数据和原位图做混合，而且不能改变镂空部分原位图的`alpha`通道的值。


在`SOUI`的`render-gdi`中我采用下面的类来实现`GDI`的半透明。

```c++
class DCBuffer
    {
    public:
        DCBuffer(HDC hdc,LPCRECT pRect,BYTE byAlpha,BOOL bCopyBits=TRUE)
            :m_hdc(hdc)
            ,m_byAlpha(byAlpha)
            ,m_pRc(pRect)
            ,m_bCopyBits(bCopyBits)
        {
            m_nWid = pRect->right-pRect->left;
            m_nHei = pRect->bottom-pRect->top;
            m_hBmp = SBitmap_GDI::CreateGDIBitmap(m_nWid,m_nHei,(void**)&m_pBits);
            m_hMemDC = ::CreateCompatibleDC(hdc);
            ::SetBkMode(m_hMemDC,TRANSPARENT);
            ::SelectObject(m_hMemDC,m_hBmp);
            ::SetViewportOrgEx(m_hMemDC,-pRect->left,-pRect->top,NULL);
            //从原DC中获得画笔，画刷，字体，颜色等
            m_hCurPen = ::SelectObject(hdc,GetStockObject(BLACK_PEN));
            m_hCurBrush = ::SelectObject(hdc,GetStockObject(BLACK_BRUSH));
            m_hCurFont = ::SelectObject(hdc,GetStockObject(DEFAULT_GUI_FONT));
            COLORREF crCur = ::GetTextColor(hdc);

            //将画笔，画刷，字体设置到memdc里
            ::SelectObject(m_hMemDC,m_hCurPen);
            ::SelectObject(m_hMemDC,m_hCurBrush);
            ::SelectObject(m_hMemDC,m_hCurFont);
            ::SetTextColor(m_hMemDC,crCur);

            if(m_bCopyBits) ::BitBlt(m_hMemDC,pRect->left,pRect->top,m_nWid,m_nHei,m_hdc,pRect->left,pRect->top,SRCCOPY);
            //将alpha全部强制修改为0xFF。
            BYTE * p= m_pBits+3;
            for(int i=0;i<m_nHei;i++)for(int j=0;j<m_nWid;j++,p+=4) *p=0xFF;
        }

        ~DCBuffer()
        {
            //将alpha为0xFF的改为0,为0的改为0xFF
            BYTE * p= m_pBits+3;
            for(int i=0;i<m_nHei;i++)for(int j=0;j<m_nWid;j++,p+=4) *p=~(*p);

            BLENDFUNCTION bf={AC_SRC_OVER,0,m_byAlpha,AC_SRC_ALPHA };
            BOOL bRet=::AlphaBlend(m_hdc,m_pRc->left,m_pRc->top,m_nWid,m_nHei,m_hMemDC,m_pRc->left,m_pRc->top,m_nWid,m_nHei,bf);
            ::DeleteDC(m_hMemDC);
            ::DeleteObject(m_hBmp);
            
            //恢复原DC的画笔，画刷，字体
            ::SelectObject(m_hdc,m_hCurPen);
            ::SelectObject(m_hdc,m_hCurBrush);
            ::SelectObject(m_hdc,m_hCurFont);
        }

        operator HDC()
        {
            return m_hMemDC;
        }

    protected:
        HDC m_hdc;
        HDC m_hMemDC;
        HBITMAP m_hBmp; 
        LPBYTE  m_pBits;
        BYTE    m_byAlpha;
        LPCRECT m_pRc;
        int     m_nWid,m_nHei;
        BOOL    m_bCopyBits;

        HGDIOBJ m_hCurPen;
        HGDIOBJ m_hCurBrush;
        HGDIOBJ m_hCurFont;
    };
```

下面以实现`DrawText`的半透明为例来分析如何实现`GDI`函数的半透明。

```c++
    HRESULT SRenderTarget_GDI::DrawText( LPCTSTR pszText,int cchLen,LPRECT pRc,UINT uFormat)
    {
        if(uFormat & DT_CALCRECT)
        {
            ::DrawText(m_hdc,pszText,cchLen,pRc,uFormat);
            return S_OK;
        }
        
        if(cchLen == 0) return S_OK;

        {
            DCBuffer dcBuf(m_hdc,pRc,m_curColor.a);
            ::DrawText(dcBuf,pszText,cchLen,pRc,uFormat);
        }

        return S_OK;
    }
```
首先来看如何解决`alpha`通道的保护问题。

为了在目标`HDC`上调用`DrawText`绘制文字，先声明一个`DCBuffer`对象：`dcBuf`。

`DCBuffer`的构造函数中，我们会创建一个临时的**32bit**位图。

再将原`DC`中的数据复制到临时位图中（注意，原位图也是**32bit**的）。

一个非常重要的工作在于在调用`GDI`的`DrawText`之前，`DCBuffer`会先把临时位图中`alpha`通道置为**255**。这样做的目的在于标识哪些像素被`DrawText`修改过。

在调用了`::DrawText`后，`SRenderTarget_GDI::DrawText`会进入`DCBuffer`的析构函数。

在析构函数中，首先对`alpha`通道中的值取反，经过这一步操作，被`::DrawText`清空的点的`alpha`通道值被修改成**255**，而那些需要透明的点的`alpha`值则变成了**0**。

到这里已经实现了对`alpha`通道的保护。

有了前面的基础，做第二步的`alphablend`就简单了，这里只需要直接调用API：`AlphaBlend`，注意`BLENDFUNCTION`中几个参数的设置。

下面解释一下为什么需要做上面的处理就可以实现`GDI`函数的半透明。

首先如`DrawText`这样的`GDI`函数通常会产生透明效果：`即矩形中的一部分点变色，而其它点不变色`。

`GDI`函数只会将那些变色的点的`alpha`通道**清0**。我们的目标则是将变色的点的RGB值与目标做alpha混合。

通过将临时位图中的`alpha`值做取反处理，被GDI函数修改过的点的`alpha`变为**255**，而需要镂空的点的`alpha`则变为了**0**。

此时再调用用`AlphaBlend`做混合，对于那些需要镂空的点，由于临时位图的`alpha`为**0**，混合后根据`AlphaBlend`的公式，即不会改变原来的RGB值，也不会改变原来的`alpha`值。

对于那些被`GDI`函数改变过的点，由于其`alpha`值都变成了**255**，其RGB部分，`AlphaBlend`会根据`BLENDFUNCTION`中指定的alpha值来和原值混合，而`alpha`部分则被修改为**255**。

最终达到半透明效果。

>注：`DCBuffer`中`CopyBits`这一步有时候不是必须的。不过很多函数如`DrawText`需要做反锯齿处理，反锯齿处理的关键也是和背景色做混合，因此从原位图复制出数据也是很有必要的。

如果用`GDI+`也可以达到相同的效果，但是`GDI+`出了名的效率低，不知道GDI函数经过如此处理后效率会不会比`GDI+`慢，从我目前简单的测试来看，效果还是很好的，效率也很高，有兴趣的朋友可以比较一下。