原文链接：[《在SOUI中使用线性布局》](http://www.cnblogs.com/setoutsoft/p/6413442.html)

`SOUI 2.5.1.1`开始支持线性布局(`LinearLayout`).

要在SOUI布局中使用线性布局, 需要在布局容器窗口里指定布局类型为`vbox | hbox`, (`vbox`为垂直线性布局, `hbox`为水平线性布局).

在指定布局类型后还可以为容器窗口指定`gravity`属性, 用来指定子窗口的默认排列模式. `vbox`的`gravity`有:`left(默认), center, right`, `hbox`有: `top(默认), center, bottom`.

在线性布局中的子窗口pos属性没有意义, 一般直接指定`size="width,height"`, `width/height`值: `-1`代表`wrap_content`, `-2`代表`match_parent`

可以使用`layout_gravity`可以更改当前窗口的排列模式.

使用`extend="left,top,right,bottom", extend_left, extend_top, extend_right, extend_bottom`来指定间距. (相当于android的`margin`)

子窗口支持使用`weight`属性.

看下面demo中的例子:

```xml
<page title="linear layout">
      <!--这里演示在SOUI中使用线性布局,在window中指定layout="vbox,hbox,linearLayout"时窗口的子窗口布局变成自动布局模式-->
      <window layout="vbox" size="-1,-1" colorBkgnd="#cccccc" gravity="center">
        <!--线性布局的自适应子窗口大小-->
        <text>vbox + gravity + wrapContent</text>
        <window size="100,30" colorBkgnd="#ff0000"/>
        <window size="200,30" extend="10,5,10,5" colorBkgnd="#ff0000"/>
        <window size="120,30" layout_gravity="right" colorBkgnd="#ff0000"/>
      </window>

      <window pos="0,[5,@-1,@200" layout="vbox" colorBkgnd="#cccccc">
        <!--线性布局的weight属性-->
        <text extend_bottom="10">vbox + gravity + weight</text>
        <window size="100,30" colorBkgnd="#ff0000"/>
        <window size="200,30" extend="10,5,10,5" colorBkgnd="#ff0000" weight="1"/>
        <window size="120,30" layout_gravity="right" colorBkgnd="#ff0000" weight="1"/>
        <button size="100,30" extend_top="10">button test</button>
      </window>

      <window pos="0,[5" layout="vbox" colorBkgnd="#cccccc" id="10000">
        <text extend_bottom="10" layout_gravity="center">hbox demo</text>
        <window size="-1,-1" layout="hbox" colorBkgnd="#888888">
          <!--线性布局之hbox-->
          <button size="100,30">button1</button>
          <button size="100,30" extend_left="10">button2</button>
          <button size="100,30" extend_left="10">button3</button>
          <button size="100,30" extend_left="10">button4</button>
        </window>
      </window>
    </page>
```

