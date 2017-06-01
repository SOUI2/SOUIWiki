原文链接：[《第十六篇：SWindow的布局属性pos2type及offset》](http://www.cnblogs.com/setoutsoft/p/4110950.html)

当窗口大小需要根据内容来确定时，使用XML布局可能需要做一些特殊的处理。

例如：不管窗口多大，我需要将该窗口相对于父窗口居中在XML中应该怎么处理？

如果窗口大小是固定的（如, 100 *100)，这样pos属性可以定义为`"|-50,|-50,|-50,|-50"`即可。

当窗口大小不确定时，SOUI中提供了`pos2type`及`offset`来协同处理。

其中`pos2type`是offset的子集。

### 下面先重点介绍offset属性
offset属性是SOUI在通过pos属性完成坐标定位后再将坐标进行偏移的属性。和pos中一般使用象素为单位不同，offset是以控件最后的大小为单位进行平移。

我们可以在XML中或者代码中使用`offset = "-0.5,-0.5"`这样的形式来描述窗口的坐标平移属性。

属性中包含两个值，分别对应X，Y方向的平移相对于窗口大小的倍数，一般为`[-1,0]`的小数(float)，当然也可以超过这个范围。。

我们先看一下代码中如何实现：

```c++
class SOUI_EXP SwndLayout
    {
    public:
        //...  
        float fOffsetX,fOffsetY;  /**< 窗口坐标偏移量, x += fOffsetX *     
        //...
    };
```

```c++
int SwndLayout::CalcPosition(LPRECT lpRcContainer,CRect &rcWindow )
    {
        int nRet=0;
       //...
        if(nRet==0)
        {//没有坐标等待计算了
            rcWindow.NormalizeRect();
            //处理窗口的偏移(offset)属性
            CSize sz = rcWindow.Size();
            CPoint ptOffset;
            ptOffset.x = (LONG)(sz.cx * fOffsetX);
            ptOffset.y = (LONG)(sz.cy * fOffsetY);
            rcWindow.OffsetRect(ptOffset);
        }
        return nRet;
    }
```

`SwndLayout::CalcPosition`是SOUI用来通过pos及offset属性计算窗口坐标的关键函数，为了突出重点，具体的坐标计算省略了，只列出平移处理部分的代码。

可以看出，在平移处理前，首先获得窗口的Size,再将Size分别乘以`fOffsetX,fOffsetY`这两个平移系数获得在x,y两个方向上的平移量。

最后才是将矩形做平移处理。

### 下面我们再来看看pos2type属性：

`pos2type`可以定义9个参考点：`center, lefttop, leftmid, leftbottom,midtop,midbottom,righttop,rightmid,rightbottom`。

下表显示对应原`pos2type`属性的`offset`属性：

| pos2type   | offset  |
| :---:   | :---:  |
| center     | -0.5,-0.5      | 
| lefttop    | 0,0      |
| leftmid    |	0,-0.5      |
| leftbottom | 0,-1      |
| midtop     | -0.5,0      |
| midbottom  | -0.5,-1      |
| righttop   | -1,0      |
| rightmid   | -1,-0.5      |
| rightbottom| -1,-1      |

从上表可以看出，原来的`pos2type`属性只能是0.5的倍数，新的`offset`属性没有该限制。

使用`pos2type`可能更为直观，但是`offset`属性则更灵活。如果两个属性同时使用，只有最后一个属性有效。

**注意**：`offset`属性是`2014.11.20`才新增加的属性，`pos2type`属性的命名是为了兼容`2014.11.20`前的版本。