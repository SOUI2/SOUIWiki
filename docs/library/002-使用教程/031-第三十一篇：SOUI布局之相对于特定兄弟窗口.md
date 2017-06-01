原文链接：[《第三十一篇：SOUI布局之相对于特定兄弟窗口》](http://www.cnblogs.com/setoutsoft/p/5164402.html)

SOUI中通过pos的标志如：`[, {, }, ]`，这4个标志可以相对于前一个及后一个兄弟窗口，但是有时候希望相对于不是前后窗口的兄弟窗口，比如一个通过一个中心窗口同时定义它的上下左右4个窗口，这个时候应该如何处理？

其实SOUI是支持相对于任意一个兄弟窗口的，但是定义方法有点复杂，所以在之前的博客文章中都没有介绍。

定义的方法是这样的：

首先被参考窗口（假定为窗口A）必须要指定窗口的ID属性，有了ID（假定id=100)，其它窗口才能引用它（这里指定name属性是不行的，系统只会通过ID去查询这个兄弟窗口）。

然后一个窗口（假定为窗口B）要相对于窗口A布局，只需要在pos中指定为如：`pos="sib.left@100:-20,sib.bottom@100:30,@100,@100"`，坐标定义中的`sib.left,sib.bottom`用来指定这两个坐标是相对于被引用窗口的`left,bottom`的值，坐标中的`100:20,100:30`刚代表相对于ID为100的兄弟窗口的left向左偏移20像素及bottom向下偏移30像素。这里的负数是代表偏移方向，和没有`sib.xxx`时的负值意义不同。

下面看下demo中的示例XML(`demo/uires/xml/page_layout.xml`)：

```xml
    <window skin="skin_page_icons" pos="[5,150,-10,-10" id="1236">
      <text pos="|0,|0" offset="-0.5,-0.5" font="adding:20" colorText="#ff000066">alpha test</text>
      <text pos="5,5" id="100" visible="0">ref text</text>
      <button pos="sib.left@100:10,sib.bottom@100:10,@100,@25" name="btn_hidetst" tip="click me to hide me and see how the next image will move">ref id:100</button>
    </window>
```

PS：这个定义方法有点山寨，将就着用吧，关键是能解决问题 :)
