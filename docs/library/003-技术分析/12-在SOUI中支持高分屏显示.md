原文链接：[《在SOUI中支持高分屏显示》](http://www.cnblogs.com/setoutsoft/p/6807680.html)

和手机屏幕一样，高分屏在PC上使用越来越多。传统的桌面程序都是像素为单位进行UI布局，而且是适配传统的96dpi的显示器的。这就导致这些程序在高分屏上显示很小，用户用起来很难受。

虽然windows系统提供了桌面程序自动放大功能，但这个放大效果是以牺牲显示效果为代价的，一个在普屏上显示很好的软件到了高分屏上显示变得非常粗糙。

要解决这个问题非常麻烦，对于有些项目甚至变成不可能完成的任务：

如果使用同一套布局，布局系统需要支持放大时使用的字体的重建，还需要支持放大时使用的图片资源的重新获取，还需要支持放大时布局位置的自动调整，然而上述3条实现任意一条都不容易。

如果使用不同的布局，就是说为普屏配置一套布局，为高分屏配置另一套布局，尽管麻烦，理论上也是可行的，但还有新的问题：维护成本高，一个地方UI元素的调整可能需要修改两个不同的布局文件；不适应动态调整，对于多显示器有不同DPI的情况，很难做到DPI的动态切换。

Android系统对于DPI的支持比较理想（IOS没有研究）：

首先Android的布局语言使用的坐标一般都用dp,sp为单位，也就是说在布局的时候，UI的位置，字体的大小都是dpi无关的，无论在高分屏还是普屏下显示的大小都是一样的。

对于图片，Android系统提供适配多种DPI的图片资源包，如果开发者只提供一种资源包，则使用这个资源包适配目标屏幕的dpi进行缩放，同样达到类似dpi无关的显示效果。

当然Android系统适配手机通常只有一个屏幕，不需要做动态DPI切换，这一点是和PC不同的地方。

鉴于我对于Android还有些经验，SOUI的`dpiaware`工作主要是参考Android的设计。

如前所述，要在一套UI布局中支持`dpiaware`，SOUI需要支持DPI变化时的字体重建，布局坐标重排，图片资源重新获取3部分工作。

首先明确DPI是布局元素相关的，SOUI中最基本的布局元素就是`SWindow`对象，要在SOUI实现`DpiAware`, 我们在DPI变化时给每个SWindow发一个`UM_DIPCHANGED`消息，`SWindow`对象收到这个消息后处理自己持有的字体，图片的重建，完成后请求宿主重新执行布局操作。

1 字体重建：SOUI有一个UI中使用的字体Pool，在窗口中保存有使用的字体的描述，DPI变化后直接使用保存的字体描述重新创建新的字体。

2 图片重建：SOUI中使用ISkinObj来代表一个绘制对象，系统内的绘制对象保存在`SkinPool`中，每一个绘制对象有两个属性：`name, scale` （scale属性是新增加的），DPI变化时，一个窗口持有的`ISkinObj`对象到`SkinPool`中查询`name`相同但`scale`不同的`ISkinObj`，找到了就使用这个新对象绘制，如果没有找到则使用当前图片自动缩放出一个`Clone`对象，并把它加入到`SkinPool`里去。

3 重新布局：相对于2.6以前版本的SOUI的布局，2.6版本的坐标描述增加了坐标单位dp，在窗口的位置及大小描述中加上dp就代码这个值是dpi相关的坐标，窗口的DPI变化后自动使用新DPI重新布局。

上面是DPIAware的基本解决方案，当然细节还有很多，具体参见系统代码。

下面我们看一下`MultiLangs`这个demo如何处理`dpiaware`。

首先演示图片的自动dpi适配。

1 在资源中增加适配不同DPI的4个图片资源，如下图：

![](assets/003/12-1497542440000.png)

2 在`uires.idx`里引入这4个图片。

```xml
<img>
    <file name="pic_100" path="image\pic_100.png"/>
    <file name="pic_125" path="image\pic_125.png"/>
    <file name="pic_150" path="image\pic_150.png"/>
    <file name="pic_200" path="image\pic_200.png"/>
  </img>
```

3 在`skin.xml`中增加4个`skin`对象, 注意它们使用相同的`name`，不同的`scale`

```xml
<skin>
  <imglist name="scale_pic" scale="100"    src="img:pic_100"/>
  <imglist name="scale_pic" scale="125"    src="img:pic_125"/>
  <imglist name="scale_pic" scale="150"    src="img:pic_150"/>
  <imglist name="scale_pic" scale="200"    src="img:pic_200"/>
</skin>
```

4 最后就是在布局XML中引用该`skin`对象，直接使用`name`引用即可，不需要关注`scale`

```xml
<page title="page1">
          <img skin="scale_pic" pos="|0,|0" offset="-0.5,-0.5"/>
          <window pos="20,20,@100dp,@30dp" colorBkgnd="rgba(255,0,0,128)">100dp*30dp</window>
</page>
```

到此完成了图片的配置。

字体和布局的`DPIAware`很简单，只需要在那些需要dpi自动缩放的位置或者字体大小数字后加上dp即可，可参考上面的XML, `pos，size，width, height`等属性都可以通过加dp来实现`dpiaware`。

DpiAware至此完成！

启程软件  2017年5月4日