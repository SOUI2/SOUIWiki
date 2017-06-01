原文链接：[《第二十四篇：导出SOUI对象到LUA脚本》](http://www.cnblogs.com/setoutsoft/p/4361310.html)

LUA是一种体积小，速度快的脚本语言。脚本语言虽然性能上和C++这样的Naitive语言相比差一点，但是开发速度快，可以方便的更新代码等，近年来受到了越来越多开发者的重视。

在SOUI框架中，我把脚本模块参考CEGUI抽象出一个独立的脚本接口，方便实现各种脚本语言的对接。

下面简单介绍一下在SOUI中实现的LUA脚本模块的实现。

在客户端程序中使用脚本语言一个基本的需求就是C++代码和脚本代码的相互调用，即C++代码可以调用脚本代码，脚本代码也要能够方便的调用C++代码。

LUA脚本原生提供了访问C函数的方法，只需要简单的调用几行代码就可以方便的把C函数注册到LUA函数空间中，但是并没有原生提供访问C++对象的能力。

但是LUA中实现的`metatable`能够很好的模拟C++的OOP能力，这也为导出C++对象到LUA提供了可能。

目前已经有很多方法可以将C++对象导出到LUA，比如`luabind,tolua++`, `fflua`及本文中用到的`lua_tinker`。

`luabind`据说体积比较大，`tolua++`好像已经没人维护了，目前只支持`lua 5.1.4`，`fflua`是国内一个大神的作品，使用简单，只是我使用中碰到一点问题，最后还是选择了`lua_tinker`。

`lua_tinker`是一个韩国大神的作品，虽然作者本人没有维护了，但是代码相对比较简单易懂，国内有不少高手都对它进行了扩展。

在SOUI中使用的是官方的0.5c版本上结合网友修改的版本，实现对`lua 5.2.3`的支持。

言归正传，下面说说如何使用`lua_tinker`导出SOUI对外到LUA。

要使用LUA，首先当然要有一份LUA内核代码，这里用的`lua 5.2.3`。

为了在SOUI中使用LUA，我们还需要使用LUA内核实现一个`SOUI::IScriptModuler`接口：

```c++
namespace SOUI
{
    class SWindow;
/*!
\brief
    Abstract interface required for all scripting support modules to be used with
    the SOUI system.
*/
struct IScriptModule : public IObjRef
{
    /**
     * GetScriptEngine
     * @brief    获得脚本引擎的指针
     * @return   void * -- 脚本引擎的指针
     * Describe  
     */    
    virtual void * GetScriptEngine () = 0;

    /*************************************************************************
        Abstract interface
    *************************************************************************/
    /*!
    \brief
        Execute a script file.

    \param pszScriptFile
        String object holding the filename of the script file that is to be executed
        
    */
    virtual void    executeScriptFile(LPCSTR pszScriptFile)  = 0;

    /*!
    \brief
        Execute a script buffer.

    \param buff
        buffer of the script that is to be executed
        
    \param sz
        size of buffer
    */
    virtual    void    executeScriptBuffer(const char* buff, size_t sz)  = 0;
    /*!
    \brief
        Execute script code contained in the given String object.

    \param str
        String object holding the valid script code that should be executed.

    \return
        Nothing.
    */
    virtual void executeString(LPCSTR str) = 0;


    /*!
    \brief
        Execute a scripted global 'event handler' function.  The function should take some kind of EventArgs like parameter
        that the concrete implementation of this function can create from the passed EventArgs based object.  

    \param handler_name
        String object holding the name of the scripted handler function.

    \param EventArgs *pEvt
        SWindow based object that should be passed, by any appropriate means, to the scripted function.

    \return
        - true if the event was handled.
        - false if the event was not handled.
    */
    virtual    bool    executeScriptedEventHandler(LPCSTR handler_name, EventArgs *pEvt)=0;


    /*!
    \brief
        Return identification string for the ScriptModule.  If the internal id string has not been
        set by the ScriptModule creator, a generic string of "Unknown scripting module" will be returned.

    \return
        String object holding a string that identifies the ScriptModule in use.
    */
    virtual LPCSTR getIdentifierString() const = 0;

    /*!
    \brief
            Subscribes or unsubscribe the named Event to a scripted function

    \param target
            The target EventSet for the subscription.

    \param uEvent
            Event ID to subscribe to.

    \param subscriber_name
            String object containing the name of the script function that is to be subscribed to the Event.

    \return 
    */
    virtual bool subscribeEvent(SWindow* target, UINT uEvent, LPCSTR subscriber_name) = 0;

    /**
     * unsubscribeEvent
     * @brief    取消事件订阅
     * @param    SWindow * target --  目标窗口
     * @param    UINT uEvent --  目标事件
     * @param    LPCSTR subscriber_name --  脚本函数名
     * @return   bool -- true操作成功
     * Describe  
     */    
    virtual bool unsubscribeEvent(SWindow* target, UINT uEvent, LPCSTR subscriber_name ) = 0;

};

struct IScriptFactory : public IObjRef
{
    virtual HRESULT CreateScriptModule(IScriptModule ** ppScriptModule) = 0;
};

}
```

实现上述接口后，SOUI就可以用这个接口和脚本交互。

导出SOUI对象通常应该在`IScriptModule`的实现类的构造中执行。

使用`lua_tinker`导出C++对象非常简单，下面看一下`scriptmodule-lua`是如何导出SOUI中使用的几个C++对象的：

```c++
//导出基本结构体类型
UINT rgb(int r,int g,int b)
{
    return RGBA(r,g,b,255);
}

UINT rgba(int r,int g, int b, int a)
{
    return RGBA(r,g,b,a);
}

BOOL ExpLua_Basic(lua_State *L)
{
    try{
        lua_tinker::def(L,"RGB",rgb);
        lua_tinker::def(L,"RGBA",rgba);

        //POINT
        lua_tinker::class_add<POINT>(L,"POINT");
        lua_tinker::class_mem<POINT>(L, "x", &POINT::x);
        lua_tinker::class_mem<POINT>(L, "y", &POINT::y);
        //RECT
        lua_tinker::class_add<RECT>(L,"RECT");
        lua_tinker::class_mem<RECT>(L, "left", &RECT::left);
        lua_tinker::class_mem<RECT>(L, "top", &RECT::top);
        lua_tinker::class_mem<RECT>(L, "right", &RECT::right);
        lua_tinker::class_mem<RECT>(L, "bottom", &RECT::bottom);
        //SIZE
        lua_tinker::class_add<SIZE>(L,"SIZE");
        lua_tinker::class_mem<SIZE>(L, "cx", &SIZE::cx);
        lua_tinker::class_mem<SIZE>(L, "cy", &SIZE::cy);

        //CPoint
        lua_tinker::class_add<CPoint>(L,"CPoint");
        lua_tinker::class_inh<CPoint,POINT>(L);
        lua_tinker::class_con<CPoint>(L,lua_tinker::constructor<CPoint,LONG,LONG>);
        //CRect
        lua_tinker::class_add<CRect>(L,"CRect");
        lua_tinker::class_inh<CRect,RECT>(L);
        lua_tinker::class_con<CRect>(L,lua_tinker::constructor<CRect,LONG,LONG,LONG,LONG>);
        lua_tinker::class_def<CRect>(L,"Width",&CRect::Width);
        lua_tinker::class_def<CRect>(L,"Height",&CRect::Height);
        lua_tinker::class_def<CRect>(L,"Size",&CRect::Size);
        lua_tinker::class_def<CRect>(L,"IsRectEmpty",&CRect::IsRectEmpty);
        lua_tinker::class_def<CRect>(L,"IsRectNull",&CRect::IsRectNull);
        lua_tinker::class_def<CRect>(L,"PtInRect",&CRect::PtInRect);
        lua_tinker::class_def<CRect>(L,"SetRectEmpty",&CRect::SetRectEmpty);
        lua_tinker::class_def<CRect>(L,"OffsetRect",(void (CRect::*)(int,int))&CRect::OffsetRect);


        //CSize
        lua_tinker::class_add<CSize>(L,"CSize");
        lua_tinker::class_inh<CSize,SIZE>(L);
        lua_tinker::class_con<CSize>(L,lua_tinker::constructor<CSize,LONG,LONG>);

        return TRUE;
    }catch(...)
    {
        return FALSE;
    }

}
```

```c++
#include <core/swnd.h>

//定义一个从SObject转换成SWindow的方法
SWindow * toSWindow(SObject * pObj)
{
    return sobj_cast<SWindow>(pObj);
}

BOOL ExpLua_Window(lua_State *L)
{
    try{
        lua_tinker::def(L,"toSWindow",toSWindow);

        lua_tinker::class_add<SWindow>(L,"SWindow");
        lua_tinker::class_inh<SWindow,SObject>(L);
        lua_tinker::class_con<SWindow>(L,lua_tinker::constructor<SWindow>);
        lua_tinker::class_def<SWindow>(L,"GetContainer",&SWindow::GetContainer);
        lua_tinker::class_def<SWindow>(L,"GetRoot",&SWindow::GetRoot);
        lua_tinker::class_def<SWindow>(L,"GetTopLevelParent",&SWindow::GetTopLevelParent);
        lua_tinker::class_def<SWindow>(L,"GetParent",&SWindow::GetParent);
        lua_tinker::class_def<SWindow>(L,"DestroyChild",&SWindow::DestroyChild);
        lua_tinker::class_def<SWindow>(L,"GetChildrenCount",&SWindow::GetChildrenCount);
        lua_tinker::class_def<SWindow>(L,"FindChildByID",&SWindow::FindChildByID);
        lua_tinker::class_def<SWindow>(L,"FindChildByNameA",(SWindow* (SWindow::*)(LPCSTR,int))&SWindow::FindChildByName);
        lua_tinker::class_def<SWindow>(L,"FindChildByNameW",(SWindow* (SWindow::*)(LPCWSTR,int ))&SWindow::FindChildByName);
         lua_tinker::class_def<SWindow>(L,"CreateChildrenFromString",(SWindow* (SWindow::*)(LPCWSTR))&SWindow::CreateChildren);
        lua_tinker::class_def<SWindow>(L,"GetTextAlign",&SWindow::GetTextAlign);
        lua_tinker::class_def<SWindow>(L,"GetWindowRect",(void (SWindow::*)(LPRECT))&SWindow::GetWindowRect);
        lua_tinker::class_def<SWindow>(L,"GetWindowRect2",(CRect (SWindow::*)())&SWindow::GetWindowRect);
        lua_tinker::class_def<SWindow>(L,"GetClientRect",(void (SWindow::*)(LPRECT))&SWindow::GetClientRect);
        lua_tinker::class_def<SWindow>(L,"GetClientRect2",(CRect (SWindow::*)())&SWindow::GetClientRect);
        lua_tinker::class_def<SWindow>(L,"GetWindowText",&SWindow::GetWindowText);
        lua_tinker::class_def<SWindow>(L,"SetWindowText",&SWindow::SetWindowText);
        lua_tinker::class_def<SWindow>(L,"SendSwndMessage",&SWindow::SSendMessage);
        lua_tinker::class_def<SWindow>(L,"GetID",&SWindow::GetID);
        lua_tinker::class_def<SWindow>(L,"SetID",&SWindow::SetID);
        lua_tinker::class_def<SWindow>(L,"GetUserData",&SWindow::GetUserData);
        lua_tinker::class_def<SWindow>(L,"SetUserData",&SWindow::SetUserData);
        lua_tinker::class_def<SWindow>(L,"GetName",&SWindow::GetName);
        lua_tinker::class_def<SWindow>(L,"GetSwnd",&SWindow::GetSwnd);
        lua_tinker::class_def<SWindow>(L,"InsertChild",&SWindow::InsertChild);
        lua_tinker::class_def<SWindow>(L,"RemoveChild",&SWindow::RemoveChild);
        lua_tinker::class_def<SWindow>(L,"IsChecked",&SWindow::IsChecked);
        lua_tinker::class_def<SWindow>(L,"IsDisabled",&SWindow::IsDisabled);
        lua_tinker::class_def<SWindow>(L,"IsVisible",&SWindow::IsVisible);
        lua_tinker::class_def<SWindow>(L,"SetVisible",&SWindow::SetVisible);
        lua_tinker::class_def<SWindow>(L,"EnableWindow",&SWindow::EnableWindow);
        lua_tinker::class_def<SWindow>(L,"SetCheck",&SWindow::SetCheck);
        lua_tinker::class_def<SWindow>(L,"SetOwner",&SWindow::SetOwner);
        lua_tinker::class_def<SWindow>(L,"GetOwner",&SWindow::GetOwner);
        lua_tinker::class_def<SWindow>(L,"Invalidate",&SWindow::Invalidate);
        lua_tinker::class_def<SWindow>(L,"InvalidateRect",(void (SWindow::*)(LPCRECT))&SWindow::InvalidateRect);
        lua_tinker::class_def<SWindow>(L,"AnimateWindow",&SWindow::AnimateWindow);
        lua_tinker::class_def<SWindow>(L,"GetScriptModule",&SWindow::GetScriptModule);
        lua_tinker::class_def<SWindow>(L,"Move2",(void (SWindow::*)(int,int,int,int))&SWindow::Move);
        lua_tinker::class_def<SWindow>(L,"Move",(void (SWindow::*)(LPCRECT))&SWindow::Move);
        lua_tinker::class_def<SWindow>(L,"FireCommand",&SWindow::FireCommand);
        lua_tinker::class_def<SWindow>(L,"GetDesiredSize",&SWindow::GetDesiredSize);
        lua_tinker::class_def<SWindow>(L,"GetWindow",&SWindow::GetWindow);

        return TRUE;
    }catch(...)
    {
        return FALSE;
    }
}
```

还是很简单吧？！

这里有两点需要注意：

前面的代码里一般是导出全局函数，成员函数及成员变量，但是类的静态成员函数是不能用上面的方法导出的，下面看一下静态函数如何处理：

```c++
BOOL ExpLua_App(lua_State *L)
{
    try{
        lua_tinker::class_add<SApplication>(L,"SApplication");
        lua_tinker::class_def<SApplication>(L,"AddResProvider",&SApplication::AddResProvider);
        lua_tinker::class_def<SApplication>(L,"RemoveResProvider",&SApplication::RemoveResProvider);
        lua_tinker::class_def<SApplication>(L,"Init",&SApplication::Init);
        lua_tinker::class_def<SApplication>(L,"GetInstance",&SApplication::GetInstance);
        lua_tinker::class_def<SApplication>(L,"CreateScriptModule",&SApplication::CreateScriptModule);
        lua_tinker::class_def<SApplication>(L,"SetScriptModule",&SApplication::SetScriptFactory);
        lua_tinker::class_def<SApplication>(L,"GetTranslator",&SApplication::GetTranslator);
        lua_tinker::class_def<SApplication>(L,"SetTranslator",&SApplication::SetTranslator);
        lua_tinker::def(L,"theApp",&SApplication::getSingletonPtr);

        return TRUE;
    }catch(...)
    {
        return FALSE;
    }
}
```

注意上面导出`SApplication::getSingletonPtr`使用的方法，实际使用的是和导出全局函数一样的方法，因此在脚本中调用的时候也只能和全局函数一样调用，这一点和C++调用静态函数是不同的。

第二个需要注意的地方就是，使用`lua_tinker`导出的C++类如果是多继承的，那么只能导出一个基类，而且这个基类必须是第一个基类。

例如SWindow类，它从多个基类继承而来，但只能使用`lua_tinker::class_inh`来声明第一个基类SObject，如果把SWindow的继承顺序调整一下，在LUA脚本里获得SWindow对象后也访问不了SObject的方法，这一点需要特别注意。

<font color=red>注：上面这个问题是我搜索好长时间才发现的，但也没有完全解决问题，本来想在导出SHostWnd时声明继承自SWindow，尽管把SWindow放到了继承的第一位，但是在LUA脚本中用SHostWnd对象访问SWindow方法仍然失败，不知道什么原因，有兴趣的朋友可以研究一下。</font>

上面介绍了如何导出C++对象到LUA空间，下面介绍一下在LUA脚本中如何使用这些C++对象：

所有的C++对象导出到LUA后都将对应一个`metatable`，可以使用"."来访问table中的成员变量（映射了C++对象的成员变量），也可以使用“：”来访问table中的成员函数（映射了C++对象函数），全局函数则直接使用函数名调用。

例如上面导出的CRect对象，在LUA脚本中使用如下：

```lua
function test(arg)
     local rc = CRect(0,0,100,100);
     local wid = rc:Width(); --访问成员函数Width()
     local x1 = rc.left;--访问基类对象RECT的成员变量left
end
```

更多操作请参考SOUI的demo