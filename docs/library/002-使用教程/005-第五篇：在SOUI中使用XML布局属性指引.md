原文链接：[《第五篇：在SOUI中使用XML布局属性指引(pos, offset, pos2type)》](http://www.cnblogs.com/setoutsoft/p/3925952.html)

### 窗口布局的概念
每一个UI都是由大量的界面元素构成的，在Windows编程，这些界面元素的最小单位通常称之为控件。

布局就是这些控件在主界面上的大小及相对位置。

传统的布局一般使用一个4个绝对坐标来定义一个控件在主窗口的位置。对于窗口是固定大小的界面来说，这种方式是最简单有效的。

然而问题在于在Windows系统上编程，基本上很少有程序的窗口是固定大小的，用户希望它的窗口能够随时调整大小。调整大小后界面里的控件还能够按照一定的规则进行重排。

我自己最讨厌的就是在WM_SIZE里重排控件位置。

随着使用XML来描述控件布局方式的出现，这种窗口布局过程中的窗口重排才得到了根本的解决。

### 两种XML布局类型
目前流行的UI库基本都是采用XML来描述窗口布局，如Android, WPF, QQ等。

使用XML布局整体上可以划分为两种类型：锚点布局和流式布局。

所谓流式布局就是一个控件只描述控件的大小，而不关心位置，它的最终显示位置由布局器计算出来，如Android及DuiLib里实现的VerticalLayout及HorizontalLayout等。

锚点布局和流式布局不同在于，它具体的定义一个控件的4个点的坐标位置，但这些位置通常不是绝对位置，而是一个相对于父窗口不同锚点的位置，当父窗口大小改变时，子窗口也会根据锚点的位置变化自动调整。布局的人使用锚点布局时很清楚一个控件最终会显示在哪，不需要很强的想象能力。

两种布局都有它们的优点，完善的UI布局器通常会同时提供两种布局能力。

考虑到锚点布局更直观，而且功能上完全可以解决布局需求，为了简化设计，在SOUI中只提供锚点布局。

### SOUI布局范例及解析
下面XML为soui-demo的主界面布局文件。

```xml
<SOUI name="dlg_main" title="SOUI-DEMO version:%ver%" bigIcon="LOGO:32" smallIcon="LOGO:16" width="600" height="400" appWnd="1" margin="5,5,5,5"  resizable="1" translucent="1" >
  <skin>
    <!--局部skin对象-->
    <gif name="gif_horse" src="gif:gif_horse"/>
    <gif name="gif_penguin" src="gif:gif_penguin"/>
  </skin>
  <style>
    <!--局部style对象-->
    <class name="cls_edit" ncSkin="_skin.sys.border" margin-x="2" margin-y="2" />
  </style>
  <root class="cls_dlg_frame" cache="1">
    <caption pos="0,0,-0,30" show="1" font="adding:8">
      <icon pos="10,8" src="LOGO:16"/>
      <text class="cls_txt_red">SOUI-DEMO version:%ver%</text>
      <imgbtn id="1" skin="_skin.sys.btn.close"    pos="-45,0" tip="close" animate="1"/>
      <imgbtn id="2" skin="_skin.sys.btn.maximize"  pos="-83,0" animate="1" />
      <imgbtn id="3" skin="_skin.sys.btn.restore"  pos="-83,0" show="0" animate="1" />
      <imgbtn id="5" skin="_skin.sys.btn.minimize" pos="-121,0" animate="1" />
      <imgbtn name="btn_menu" skin="skin_btn_menu" pos="-151,2" animate="1" />
    </caption>
    <tabctrl name="tab_main" pos="5,30,-5,-5" show="1" curSel="0" focusable="0" animateSteps="10" tabHeight="75" tabSkin="skin_tab_main" text-y="50" iconSkin="skin_page_icons" icon-x="10">
      <page title="listctrl123">
        <listctrl name="lc_test" pos="10,0,-10,-10" itemHeight="20" headerHeight="30" cache="1" cursor="CUR_TST">
          <header align="left" itemSwapEnable="1" fixWidth="0" sortHeader="1">
            <items>
              <item width="150">name</item>
              <item width="150">gender</item>
              <item width="150">age</item>
              <item width="150">score</item>
            </items>
          </header>
        </listctrl>
      </page>
      <page title="webkit">
        <include src="layout:page_webkit"/>
      </page>
      <page title="flash">
        <flash pos="0,0,-0,-0" name="ctrl_flash" url="http://swf.sc.chinaz.com//Files//DownLoad//flash2//201401//flash2524.swf" delay="1"/>
      </page>
      <page title="gif">
        <gifplayer pos="10,10" skin="gif_horse" name="giftest" cursor="ANI_ARROW"/>
        <button width="250" height="30" name="btnSelectGif">load gif file</button>
        <gifplayer pos="10,150" skin="gif_penguin"/>
        <icon pos="10,300" src="LOGO:64"/>
      </page>
      <page title="layout">
        <include src="layout:page_layout"/>
      </page>
      <page title="treebox">
        <include src="layout:page_treebox"/>
      </page>
      <page title ="misc.">
        <include src="layout:page_misc"/>
      </page>
      <page title="treectrl">
        <include src="layout:page_treectrl"/>
      </page>
      <page title="about">
        <include src="layout:page_about"/>
      </page>
    </tabctrl>
  </root>
</SOUI>
```

可以看到在这个XML中，有一个根节点：SOUI。在根节点中，定义了主界面的真窗口的各种属性（属性的含义见后续篇）。

在根节点下有3个节点，分别是skin, style及root。

skin, 和style和前一篇讲的init.xml的功能一样。不同在于在布局文件中定义的skin及style只在当前窗口的生命周期期间有效，类似于C++函数中的局部变量，窗口关闭后这些对象会自动析构。我称之为局部skin及局部style。

窗口中控件的布局信息定义在root节点中。

root节点本身也是一个SWindow窗口对象，但是在这里必须是"root"才能识别，在这个节点中可以有SWindow的各种属性，但是和布局位置相关的属性自动无效，因为该窗口总是充满整个宿主窗口。

在root节点下可以按照不同的布局层次采用锚点布局方式布局各种系统内置控件及用户自定义控件。

在demo中，我首先在最上面布局一个caption控件，caption控件里又有各种标题，按钮等子控件。

然后在下面布局一个tabctrl以及它的子控件。

在这个布局XML中有大量的控件属性定义。不同的控件有不同的属性，这里不详细展开，这里主要关注一下page节点下的include节点。

include只有一个属性：src，src定义如何去引用在另一个XML文件中定义的布局XML，如“layout:page_layout”代表这里要引用在layout资源类型中定义的name为page_layout的XML文件（关于资源的定义参考第四篇）。

下面是`layout:page_layout`指向的XML文件的内容：

```c++
<include>
  <text pos="100,10" pos2type="center">center align1</text>
  <text pos="100,30" pos2type="center">center align align</text>
  <text pos="250,50" pos2type="rightTop">align right top</text>
  <text pos="250,70" pos2type="rightTop">align right top 2</text>
  <check pos="250,90" pos2type="rightTop">check right top</check>
  <check pos="250,110" pos2type="rightTop" font="adding:-5">check right top1235</check>

  <text pos="250,130" class="cls_txt_red">text left top</text>
  <button pos="10,150,@150,@30">button 1 using @</button>
  <button pos="10,200" width="150" height="30">button 1 using width</button>

  <button name="btn_hidetst" pos="300,150,@100,@30" display="0" tip="click me to hide me and see how the next image will move">hide test</button>
  <img skin="skin_page_icons" iconIndex="1" pos="[5,150,-10,-10" />
</include>
```

可以看到在这个文件中，有一个以"include"的根节点，在include节点下才是布局XML。

这里的include代表该文件只能是被其它的有include元素的布局文件引用。

需要注意的是，在include的XML文件中不能定义局部skin及局部style。

### SOUI的锚点布局
SOUI布局全部采用相对坐标，由`pos,offset(pos2type), size, width,height` 这几个个窗口属性配合指定。

#### size, width, height属性
size, width, height比较简单，是用来指定窗口的大小的，只有在pos属性指定的值个数不为4时生效。

size是2014年底增加的布局属性，`size="width,height"`。

width, height可以有3种值：full,-1,非负整数。

为full时，代表高度或者宽度和父窗口的客户区大小相等。

-1代表根据窗口内容自动计算窗口大小。

非负整数直接指定窗口大小。

在图片控件中，控件是指定的皮肤默认大小。
在文本控件中，还可以指定一个maxWidth属性，控件是文本内容的大小，但宽度不超过maxWidth。

#### pos属性
pos属性可以指定4个值，也可以指定2个值。指定4个值时，分别代表控件的`left,top,right,bottom`,指定两个值时代表控件的x,y，具体位置还依赖于另外3个参数。

指定4个值时，pos目前支持7种标志：`|,%,[,],{,},@`

“**|**”代表参考父窗口的中心；如`|-10`代表在父窗口的中心向`左/上`偏移10象素。

“**%**”代表在父窗口的百分比，可以是小数，负数。如：%40代表在父窗口的40%位置，`%-40`则等价于`(1-40%)`。

“**[**”相对于前一兄弟窗口。用于X时，参考前一兄弟窗口的right，用于Y时参考前一兄弟窗口的bottom

“**]**”相对于后一兄弟窗口。用于X时，参考后一兄弟的left,用于Y时参考后一兄弟的top

“**{**”相对于前一兄弟窗口。用于X时，参考前一兄弟窗口的left，用于Y时参考前一兄弟窗口的top

“**}**”相对于后一兄弟窗口。用于X时，参考后一兄弟的right,用于Y时参考后一兄弟的bottom

“**@**”标志用来指定窗口的大小，只能出现在pos属性的第3，4个值中，用来标识窗口的宽度。当后面的值为负时，代表自动计算窗口的宽度或者高度（2015.3.3新增加解释）。

>注：`“|“, "[" ,"]", "{", "}" `中指定的值都可以为正或者负，正时向右或者下偏移，负则向左或者上偏移。
>>当没有上述标志时，负号代表参考父窗口的右边或者下边缩进绝对值位置。如：`pos="0,0,-0,-0"`代表占满父窗口。而`pos="10,10,-10,-10"`则代表在父窗口的基础上向内全部缩进10点。

**@**:指定窗口的size。只能用于x2,y2，用于x2时，指定窗口的width，用于y2时指定窗口的height。注：只能为正值，负号会自动忽略。

其中“{”和“}”是SOUI在DUIENGINE的基础上新增加的布局标志（SOUI是在DUIENGINE的基础上全面重构而来）。

注意!!!由于系统运行向前及向后引用，理论上有可能出来循环引用，导致界面布局失败，因此在使用`"["，"{"，“}” 和"]"`这几个标志时需要特别注意。

当pos只指定了x1,y1时，通常需要和offset(或者pos2type),size(或者width,height)配合使用。

offset及pos2type属性
offset属性包含两个值，用来代表窗口在通过其它布局属性完成后的偏移量：如`offset="-1,-1"`，该offset表明窗口向左方及上方各平衡一个窗口大小的单位。

offset及pos2type属性具体请参考：[《第十六篇：SWindow的布局属性pos2type及offset》](http://www.cnblogs.com/setoutsoft/p/4110950.html)

在SOUI的布局系统中，使用一个pos属性基本可以完整90％以上的布局功能，建议用户在demo中修改各种布局属性来观察控件位置的变化以加深对SOUI布局系统的理解。