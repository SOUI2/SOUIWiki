原文链接：[《第二十篇：在SOUI中使用分层窗口》](http://www.cnblogs.com/setoutsoft/p/4209566.html)

从Windows 2K开始，MS为UI开发引入了分层窗口这一窗口风格。使用分层窗口，应用程序的主窗口可以是半透明，也可以是逐点半透明（即每一个像素点的透明度可以不同）。

可以说，正是因为有了分层窗口，在Windows上开发的应用程序的UI才真正炫起来。

在UI的主窗口上加一个分层窗口的风格对于一个稍有点UI开发经验的程序员来说是非常简单的，本篇要说的是在SOUI的窗口系统中实现SOUI的分层窗口。

正如使用系统的窗口已经可以实现很漂亮的UI，我们还是会需要DirectUI这样的UI开发技术；有了系统的分层窗口，我们还是会需要在DirectUI Window之间的分层窗口技术。

### 一个DirectUI系统的分层窗口有什么用呢？
分层窗口和一般窗口的关键区别在在于：一个分层窗口是一个相对独立的渲染层，它能将它的子窗口渲染到这个窗口上，当所有分层窗口都渲染完成后，再按照分层窗口的zorder顺序通过不同的alpha混合技术进行混合；而一般的DirectUI系统只有一个渲染层，所有窗口按照zorder顺序依次渲染到这个渲染层上，缺少了不同渲染层的概念。

一个简单的例子：业务可能希望一个功能面板能够整体半透明，该功能面版可能包含很多子窗口。

如果没有分层窗口特性，在DirectUI中实现类似的需求很很复杂（如分别指定每一个子窗口的半透明）而且可能导致性能下降、内存消耗的提高等问题。

如果有了分层窗口，实现类似的需求就非常简单了：只需要为该功能面板指定一个透明度就行了。在渲染的时候，功能面板的子窗口会以正常的渲染方式渲染到分层窗口的缓存上，全部渲染完成后，再整体和上一个渲染层混合，从而得到期望的效果。

### 在SOUI里如何使用分层窗口：
SOUI采用XML定义UI，要定义分层窗口只需要一个`layeredWindow="1"`的属性即可。

XML定义：

```XML定义：
<window pos="0,0,-0,-0" clipClient="1" skin="skin_bkgnd" >
        <flywnd pos="-210,10,-0,-10" posEnd="-10,10,@210,-10" alpha="100" layeredWindow="1">
          <toggle pos="0,|-15,@10,@30" skin="_skin.sys.tree.toggle" name="switch" cursor="hand" tip="click me to show the animator that show or hide the pane"></toggle>
          <window pos="10,0,-0,-0" colorBkgnd="#ff0000">
            <treectrl pos="10,0,-10,-10"  name="mytree2" itemHeight="30" iconSkin="skin_tree_icon" checkBox="1" font="underline:1">
              <item text="组织机构" img="0" selImg="1"  expand="0">
                <item text="市场部" img="0" selImg="1">
                  <item text="南一区" img="2"/>
                  <item text="北二区" img="2"/>
                  <item text="西三区" img="2">
                    <item text="第一分队" img="0" selImg="1" expand="0">
                      <item text="张三组" img="2"/>
                      <item text="李四组" img="2"/>
                      <item text="王五组" img="2"/>
                    </item>
                  </item>
                </item>
              </item>
              <item text="宣传部" img="0" selImg="1" expand="0">
                <item text="南一区" img="2"/>
                <item text="北二区" img="2"/>
                <item text="西三区" img="2"/>
              </item>
            </treectrl>
            <text pos="10,-100,-0,@100" multilines="1">click the left grip to \nshow or hide the \npane</text>
          </window>
        </flywnd>
      </window>
```

显示效果：

![](assets/002/020-1496319441000.gif)
