原文链接：[《实现一种快速查找Richedit中可见区域内OLE对象的方法》](http://www.cnblogs.com/setoutsoft/p/4227529.html)

`Richedit`是一个OLE容器，使用`Richedit`来显示IM聊天内容时，通常使用OLE对象来实现在`Richedit`中播放表情动画。

 - 触发表情的绘制有两种途径：
   - 来自`Richedit`的刷新消息。
   - 来自表情动画定时器的刷新消息。

要刷新表情的显示首先需要知道表情的显示位置。

第一种刷新过程中，绘制消息参数里已经给出绘制位置，直接在指定的位置绘制即可。

但是表情主动刷新时如何获取表情的显示位置确是一个问题。

网上有不少代码演示了如何获通过枚举`Richedit`中的OLE对象获取表情的代码。

这些代码估计很多都来自一个共同的祖先：

为了获得一个OLE对象的显示位置，都需要从当前`Richedit`中逐个枚举OLE对象，直接找到这个触发刷新的OLE对象为止。

再通过该OLE对象的字符索引计算出显示位置。

当`Richedit`中插入的OLE对象相对较少时，这种方法还可以满足需要，但是如果表情成千上万，这样的查询方式必然导致CPU占用飙升。

虽然要获得一个OLE对象的显示位置，目前看来只有先获得该对象在`Richedit`中的插入位置，但是有没有更简单的方法先判断一个OLE对象是否在可见范围内呢？

要解决这个问题，首先需要从`Richedit`中获取可见字符的范围。

首先通过`EM_GETFIRSTVISIBLELINE`+`EM_LINEINDEX`获得第一个可见行的第一个字符的索引号。

通过`EM_GETRECT`+`EM_CHARFROMPOS`可以获得最后一个可见字符的索引号。

如此我们就可以获得可见字符的范围。

表情动画触发时我们只知道这个OLE对象的指针，但并不知道这个OLE对象的字符索引。

要获得显示位置，首先需要判断这个OLE对象是不是在可见范围。

为此我可以采用两分法快速找出所有在可见范围内的OLE对象，再将当前的OLE对象和可见范围内的OLE对象逐个比较，进而判断该对象是否可见，并最终获得显示位置。

相比网上流传的方法，这种方法在查找一个OLE对象的显示位置时只需要和可见范围内的OLE对象逐个比较，比较次数通常是非常小的，因此完全不用担心CPU占用问题。

下面是用源代码，希望对那些被这个问题困扰的人有些帮助。

```c++
LONG GetOleCP(IRichEditOle *pOle, int iOle)
{
    REOBJECT reobj={0};
    reobj.cbStruct=sizeof(REOBJECT);
    pOle->GetObject(iOle,&reobj,REO_GETOBJ_NO_INTERFACES);
    return reobj.cp;
}

//find first Ole Object in char range of [cpMin,cpMax)
int FindFirstOleInrange(IRichEditOle *pOle, int iBegin,int iEnd,int cpMin,int cpMax)
{
    if(iBegin==iEnd) return -1;
    
    int iMid = (iBegin + iEnd)/2;
    
    LONG cp = GetOleCP(pOle,iMid);
    
    if(cp < cpMin)
    {
        return FindFirstOleInrange(pOle,iMid+1,iEnd,cpMin,cpMax);
    }else if(cp >= cpMax)
    {
        return FindFirstOleInrange(pOle,iBegin,iMid,cpMin,cpMax);
    }else
    {
        int iRet = iMid;
        while(iRet>iBegin)
        {
            cp = GetOleCP(pOle,iRet-1);
            if(cp<cpMin) break;
            iRet --;
        }
        return iRet;
    }
}

//find Last Ole Object in char range of [cpMin,cpMax)
int FindLastOleInrange(IRichEditOle *pOle, int iBegin,int iEnd,int cpMin,int cpMax)
{
    if(iBegin==iEnd) return -1;

    int iMid = (iBegin + iEnd)/2;

    LONG cp = GetOleCP(pOle,iMid);

    if(cp < cpMin)
    {
        return FindLastOleInrange(pOle,iMid+1,iEnd,cpMin,cpMax);
    }else if(cp >= cpMax)
    {
        return FindLastOleInrange(pOle,iBegin,iMid,cpMin,cpMax);
    }else
    {
        int iRet = iMid;
        while(iRet<(iEnd-1))
        {
            cp = GetOleCP(pOle,iRet+1);
            if(cp>=cpMax) break;
            iRet ++;
        }
        return iRet;
    }
}

int CGifSmileyCtrl::GetObjectPos( HWND hWnd)
{
    if ( !hWnd ) return -1;
    IRichEditOle * ole=NULL;
    if (!::SendMessage(hWnd, EM_GETOLEINTERFACE, 0, (LPARAM)&ole)) return -1;

    int iRet = -1;

    //获得可见字符范围
    int iFirstLine = SendMessage(hWnd,EM_GETFIRSTVISIBLELINE,0,0);
    RECT rcView;
    SendMessage(hWnd,EM_GETRECT,0,(LPARAM)&rcView);
    POINT pt={rcView.right+1,rcView.bottom-2};
    
    LONG cpFirst = SendMessage(hWnd,EM_LINEINDEX,iFirstLine,0);
    LONG cpLast  = SendMessage(hWnd,EM_CHARFROMPOS,0,(LPARAM)&pt);
    
    //采用两分法查找在可见范围中的OLE对象
    int nCount=ole->GetObjectCount();

    int iFirstVisibleOle = FindFirstOleInrange(ole,0,nCount,cpFirst,cpLast);
    if(iFirstVisibleOle!=-1)
    {
        int iLastVisibleOle = FindLastOleInrange(ole,iFirstVisibleOle,nCount,cpFirst,cpLast);
        ATLASSERT(iLastVisibleOle!=-1);

        for(int i=iFirstVisibleOle;i<=iLastVisibleOle;i++)
        {
            REOBJECT reobj={0};
            reobj.cbStruct=sizeof(REOBJECT);
            ole->GetObject(i,&reobj,REO_GETOBJ_NO_INTERFACES);

            if (reobj.clsid==__uuidof(CGifSmileyCtrl) && ((CGifSmileyCtrl*)reobj.dwUser)==this)
            {
                iRet = i;

                break;            
            }
            
        }
    }
    ole->Release();
    return iRet;
}
```

上面`CGifSmileyCtrl`代表一个表情OLE对象