原文链接：[《第六篇：在SOUI中用九宫格拉伸方式显示一个图片资源》](http://www.cnblogs.com/setoutsoft/p/3925996.html)

SOUI的初学者刚开始可能难以搞清楚在SOUI中显示一个图片资源的流程，这里做一个简单的示范。

首先我们准备好一张图，以下图为例。

![](assets/002/006-1495895049000.png)

第一步，我们首先把这个图片文件复制到demo的uires目录下，新建一个目录jpg,下面只有这一个文件9.jpg

第二步，我们需要在uires.idx中引入该图片资源

```xml
 <jpg>
    <file name="girl" path="jpg\9.jpg"/>
  </jpg>
```

我们给这个资源命名为"girl"。

第三步，我们在全局或者窗口局部的skin结点中定义一个imgframe对象。这里定义在主窗口的局部skin中。

```xml
 <skin>
    <!--局部skin对象-->
    <gif name="gif_horse" src="gif:gif_horse"/>
    <gif name="gif_penguin" src="gif:gif_penguin"/>
    <imgframe name="skin_girl" src="jpg:girl" margin-x="150" margin-y="150"/>
  </skin>
```

注意上面代码中对girl的引用，我们保留x及y方向各150个点不拉伸。

第四步，在UI中定义一个img控件对象来显示该图片。

```xml
 <page title="jpg:girl">
     <img pos="0,0,-0,-0" skin="skin_girl"/>
 </page>
```

大功告成！

我们运行一下程序看看效果。

下面是缩小状态：

![](assets/002/006-1495895175000.png)

可以看到边缘的点和中间的点拉伸不一样。

再看看放大一点的状态：

![](assets/002/006-1495895193000.png)

这样效果看上去好些了。

全部工作就是修改XML文件，不需要涉及一行C++代码，即可完成一个图片的显示。

从文件中加载图片基本类似，可以参考demo中从文件中加载GIF动画的例子。

![](assets/002/006-1495895246000.png)


