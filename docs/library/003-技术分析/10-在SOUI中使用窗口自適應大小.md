原文链接：[《在SOUI中使用窗口自适应大小》](http://www.cnblogs.com/setoutsoft/p/6621272.html)

`SOUI 2.5.0.3`开始支持窗口大小自适应。

`2.5.0.3`以前，宿主窗口要适应显示內容大小比较麻烦，因为一般都是布局内容适应宿主。

`SOUI 2.5.+`开始支持线性布局，线性布局是借鉴了Android的线性布局，对于内容自适应的支持更加理想。

要想窗口大小自适应，只需要在布局的SOUI结点中指定`widt`或者`height`为**-1**即可（哪个为-1代表哪个方向上自适应），前提是Create这个窗口時没有在代码中指定窗口大小。

示例：

```xml
<?xml version="1.0"?>
<SOUI name="wrap_content" title="wrap_content" width="400" height="-1" margin="5,5,5,5" resizable="1" wndType="normal" translucent="1">
  <!--演示root大小自动计算,注意soui结点width,height的设置，哪一个值需要自动计算就设置为-1-->
    <root size="-2,-1" layout="vbox" padding="5,5,5,5" colorBkgnd="#ffffff">
    <caption size="-2,-1" colorBkgnd="#ffff00">
      <text size="-1,-1" font="bold:1" text="窗口大小自适应演示,支持左右拉伸"/>
    </caption>
    <window size="-2,100" layout="hbox">
      <window size="0,-2" text="水平平分窗口1" weight="1" colorBkgnd="#ff0000"/>
      <window size="0,-2" text="水平平分窗口2" weight="1" colorBkgnd="#00ff00"/>
      <window size="0,-2" text="水平平分窗口3" weight="1" colorBkgnd="#0000ff"/>
    </window>
    <text size="-1,-1" text="使用layout_gravity属性居中对齐" layout_gravity="center"/>
    <window size="-1,-1" layout_gravity="right" extend_top="5" layout="hbox">
      <!--IDCANCEL支持enter退出窗口-->
      <button size="-1,-1" padding="10,5,10,5" id="IDOK" text="确定"/>
      <!--IDCANCEL支持esc退出窗口-->
      <button size="-1,-1" padding="10,5,10,5" extend_left="10" id="IDCANCEL" text="取消"/>
    </window>
    </root>
</SOUI>
```

显示效果：

![](assets/003/10-1497542012000.png)

