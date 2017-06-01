原文链接：[《第三十三篇：使用uiresImporter生成uires.idx及skin.xml》](http://www.cnblogs.com/setoutsoft/p/5428158.html)

在SOUI中，使用uires.idx这个文件来记录程序中使用的所有资源文件。

此外绘制对象(ISkinObj)则一般放在skin.xml中描述。

要向一个界面中增加一个新的图片，在没有uiresImporter之前，首先我们需要把新的图片资源增复制到uires下的某个目录下，然后在uires.idx中加一条文件记录，然后在skin.xml中使用一个适当的skin类型(一般是imglist,imgframe)来描述图片的显示方式，再在UI中引用该skin来绘制。

由于SOUI目前没有提供UI编辑器，所有的XML都需要手写，图片很多的时间文件导入是一个很麻烦又容易出错的工作。

根据前段时间一个网友制作的内部使用的SOUI辅助工具的思想，我开发了uiresimporter这个工具。

`uiresimporter.exe`位于SOUI的tools目录下，对应源代码在`tools\src\uiresimporter`里。

`uiresimporter`的目标就是试图解决手动增加资源的麻烦。

和`uiresBuilder`一样，`uiresimporter`也是一个命令行工具，它支持5个参数，见下面示例代码（`demos\mclistview_demo\uiresimporter.bat`)：

```shell
rem 使用uiresImporter来自动导入资源到uires.idx及values\skin.xml.
rem -p中指定uires目录。
rem -s中指定需要在uires.idx中自动更新的文件夹。存在多个目录时应该使用"a|b|c"这样的形式分割，并使用引号。
rem -i参数中指定的图片支持自动生成skin，自动生成skin只支持imglist,imgframe两种，不支持的图片放到其它目录，如示例中的滚动条皮肤。
rem -b yes自动备份原有XML。no不备份。
rem -c yes 皮肤默认支持着色处理，no 默认禁止着色。
%SOUIPATH%\tools\uiresImporter.exe -p uires -s "layout|icon|imgx" -i image -b yes -c no
```

为了自动导入图片，我们需要为图片的文件名做点修改：`uiresimporter`通过文件名后的以[]包含的内容来识别图片显示格式。

可以有3种格式：

 1. 对于`imglist`，只需要在[]中指定一个子图数量即可，如`btn_login[3].png`，这样`uiresimporter`自动生成一个名字为`btn_login`的imglist对象，这个对象有3种状态。（当不指定[x]时，也生成一个imglist对象，状态数量为1。
 2. 对于imgframe，有一种完成的方式和一种缩略形式：  
  >2.1 完全形式：`bg_login[1{2,40,2,10}].png`。这代表图片只有一个状态，它的九宫切分为`left:2,top:40,right:2,bottom:10`。
  
  >2.2. 缩略形式：`bg_login[1{2,5}].png`。当九宫的上下及左右大小相同时，可以使用缩略形式来命名。
  
  >2016-5-2号版本新增加以下可选参数：

  >{ec=0/1} 是否支持皮肤着色(enableColorize)
  
  >{fit=0/1} 自适应绘图标志
  
  >{tile=0/1} 平铺标志
  
  >{filter=0/1/2/3} 插值滤镜类型, 0=null, 1=low, 2= midium, 3=high
  
  >{vert=0/1} 子图垂直排列标志。

  在imgframe中，上述新标志必须在margin标志之后，否则margin标志将不能识别。 

<font color=red>注：如果是需要将资源编译到EXE，导入文件后记得使用uiresbuilder来重新生成.rc2文件。</font>