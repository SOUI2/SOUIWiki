原文链接：[《为什么在soui中加载JPG文件失败》](http://www.cnblogs.com/setoutsoft/p/4594987.html)

在SOUI中解决解码器是一个独立的模块。

不同的解码器决定了程序中能够加载什么样的图片类型。

使用`SComMgr`来加载SOUI的模块时，debug模式下默认的图片解码器是`imgdecoder-png`。这个解码器只能解码PNG图片。至于为什么用这个解码器作为debug版本的默认解码器是为了演示在SOUI中使用`APNG`动画，只有这个解码器支持`APNG`解码。

要使用其它解码器只需要在实例化`SComMgr`时提供一个解码器参数就行：

```c++
class SComMgr
{
public:
    SComMgr(LPCTSTR pszImgDecoder = NULL)
    {
        if(pszImgDecoder) m_strImgDecoder = pszImgDecoder;
        else m_strImgDecoder = COM_IMGDECODER;
    }
 
    BOOL CreateImgDecoder(IObjRef ** ppObj)
    {
        if(m_strImgDecoder == _T("imgdecoder-wic"))
            return SOUI::IMGDECODOR_WIC::SCreateInstance(ppObj);
        else if(m_strImgDecoder == _T("imgdecoder-stb"))
            return SOUI::IMGDECODOR_STB::SCreateInstance(ppObj);
        else if(m_strImgDecoder == _T("imgdecoder-png"))
            return SOUI::IMGDECODOR_PNG::SCreateInstance(ppObj);
        else if(m_strImgDecoder == _T("imgdecoder-gdip"))
            return SOUI::IMGDECODOR_GDIP::SCreateInstance(ppObj);
        else
        {
            SASSERT(0);
            return FALSE;
        }
    }
    //...
}
```

可以看出SOUI实现了4种图片解码器，除了`imgdecoder-png`外，其它3个都是全图片格式支持的。

因此只需要使用

```c++
SComMgr *pComMgr = new SComMgr("imgdecoder-gdip");
```

代替

```c++
SComMgr *pComMgr = new SComMgr();
```

即可实现`JPG`的解码。