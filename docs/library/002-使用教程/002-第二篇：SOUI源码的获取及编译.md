原文链接：[《第二篇：SOUI源码的获取及编译》](http://www.cnblogs.com/setoutsoft/p/3908686.html)

### 源代码的获取

SOUI的源码采用SVN管理。

SVN：http://code.taobao.org/svn/soui2

![](assets/002/002-1495813468000.png)

这里主要包含两个目录：trunk 及 third-part。

trunk目录保存SOUI项目的全部代码，third-part保存soui系统使用到的不方便放到trunk的第三方库，目前只有一个WKE(一个精简的webkit)的源代码。

一般情况下只获取trunk的代码就行。

### SOUI的编译

SOUI项目采用QT的qmake管理项目文件。qmake已经从QT中分离出来，不需要你的机器上安装QT。

如果你的机器上安装了VS2008，可以直接打开trunk的根目录下的`soui.08.sln`来编译，这个项目中各工程的编译依赖已经设置好，直接F7就可以全部完成编译。

如果你的机器安装的是其它版本（支持vs2005-~~vs2013~~**vs2017**)，可以采用trunk目录下的`make(*).bat`来生成对应版本的项目文件，项目文件生成成功后会在根目录生成一个`soui.sln`，打开该sln即可。`VS2010+`的版本需要先生成VS2010的项目文件，再用VS打开并升级。要生成`vs2005`，可以手动修改`make(*).bat`中的参数。

如果安装的是vs2008或者vs2010还可以使用`buildAll_x86.bat`来生成项目文件并使用命名行完成编译。

打开`make(dll-win32-vs08).bat`可以看到里面只有两行代码：

```dos
call "%VS90COMNTOOLS%..\..\VC\vcvarsall.bat" x86
tools\qmake -tp vc -r -spec .\tools\mkspecs\win32-msvc2008 "CONFIG += DLL_SOUI USING_MT CAN_DEBUG"
```

第一行通过VS的环境变量加载VS的PATH信息。

第二行调用qmake生成项目文件： **-spec** 后面的参数指定生成的项目文件VS版本`(03,05,08,10)`，`CONFIG += ***`用来控制如何生成项目文件。项目文件支持4个预定义参数：

**DLL_SOUI**:代表将SOUI模块编译生成一个DLL，没有该参数则生成LIB；

**USING_MT**:代表使用MT方式连接CRT，否则采用MD方式；

**CAN_DEBUG**:为release版本生成调试符号；

**USING_CLR**:项目提供“公共语言运行时”支持；

如果需要其它配置，可以手动修改`common.pri`。

下面是`common.pri`的代码，基本可以望文生义：

```code
CONFIG -= qt
CONFIG += exceptions_off stl_off

CharacterSet = 1
#DEFINES -= UNICODE


CONFIG(debug, debug|release) {
OBJECTS_DIR = $$dir/obj/debug/$$TARGET
DESTDIR = $$dir/bin
QMAKE_LIBDIR += $$DESTDIR
}
else {
OBJECTS_DIR = $$dir/obj/release/$$TARGET
DESTDIR = $$dir/bin
QMAKE_LIBDIR += $$DESTDIR
}

#<--下面这段代码为debug和release生成不同的文件名
SAVE_TEMPLATE = $$TEMPLATE
TEMPLATE = fakelib
TARGET = $$qtLibraryTarget($$TARGET)
TEMPLATE = $$SAVE_TEMPLATE
#-->

DEFINES += _CRT_SECURE_NO_WARNINGS

QMAKE_LFLAGS += /MACHINE:X86


!CONFIG(USING_CLR){
#关闭RTTI
QMAKE_CXXFLAGS_RTTI_ON += /GR-
}
else{
QMAKE_CXXFLAGS += /clr
}

QMAKE_CXXFLAGS += -Fd$(IntDir)


QMAKE_CXXFLAGS_RELEASE += /O1
QMAKE_CXXFLAGS_RELEASE += /Zi

CONFIG(CAN_DEBUG){
#Release版本允许生产调试符号
QMAKE_LFLAGS_RELEASE += /DEBUG
QMAKE_LFLAGS_RELEASE += /OPT:REF /OPT:ICF
}

CONFIG(USING_MT){
#使用MT链接CRT
QMAKE_CXXFLAGS_RELEASE += /MT
QMAKE_CXXFLAGS_DEBUG += /MTd
}

CONFIG(USING_CLR){
#使用MD链接CRT
QMAKE_CXXFLAGS_RELEASE -= /MT
QMAKE_CXXFLAGS_DEBUG -= /MTd

QMAKE_CXXFLAGS_RELEASE += /MD
QMAKE_CXXFLAGS_DEBUG += /MDd
}
#关闭异常
QMAKE_CXXFLAGS -= -EHsc

win32-msvc*{
QMAKE_CXXFLAGS += /wd4100 /wd4101 /wd4102 /wd4189 /wd4996
}
```