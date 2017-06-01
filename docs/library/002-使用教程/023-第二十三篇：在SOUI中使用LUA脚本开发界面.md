原文链接：[《第二十三篇：在SOUI中使用LUA脚本开发界面》](http://www.cnblogs.com/setoutsoft/p/4340851.html)

像写网页一样做客户端界面可能是很多客户端开发的理想。

做好一个可以实现和用户交互的动态网页应该包含两个部分：使用html做网页的布局，使用脚本如vbscript,javascript做用户交互的逻辑。当需求变化时，只需要在服务端把相关代码调整一下，用户即可看到新的内容（界面）。

传统的客户端程序开发流程和网页开发可能完全不同。

首先是界面的布局，在老式的界面布局过程中，程序员先在界面上放好各种控件，然后需要自己通过相应的代码来维护界面在不同状态下控件的显示状态及位置。当界面中元素很多时，单纯布局的代码可能就会非常的复杂。

界面做好了，程序员需要增加代码响应界面中UI元素对应的逻辑。如逻辑比较固定，一旦做好就不需要变化时，这样的实现的程序效率可能更高。然而做客户端的人可能都经历过修改界面的问题。简单的控件坐标调整还好办，如果程序的界面风格甚至整个程序的运行逻辑都要变化时，使用传统的开发方式实现的客户端程序更新起来会非常困难，有时甚至变得不可能实现。

近年来，在各种新兴的界面开发库中使用XML来描述布局已经得到了广泛的应用。使用XML描述布局的基本出发点应该和使用HTML描述网页布局类似，关键是解决界面元素的批量创建及元素间位置的自动布局及UI大小变化后的UI元素自适应。然而到目前为止在UI库中使用脚本来实现逻辑控制的还不多见，据我所知，在流行的UI库中只有qt, bolt这两个项目支持。无论是QT还是Bolt，这两个都是重量级的UI库，而且QT中使用脚本（Bolt不了解）的方式也和网页开发想去甚远。

尽管在DuiEngine时代的代码库中就包含了一个LUA脚本模块，但那个时候的脚本模块也仅限于简单的LUA脚本和C++控件代码之间简单的相互调用，还没有形成一个清晰的应用模式。

经过2015年春节期间对脚本模块的重构，终于在SOUI中实现了和网页开发基本一样的界面开发流程。

先看一个SOUI DEMO中100％使用LUA脚本实现的小游戏效果：

![](assets/002/023-1496320185000.gif)

下面我们再看看实现上述效果用到的代码：

在SOUI中，脚本模块和其它如渲染模块等一样是使用类似插件形式实现的，当然首

```c++
int WINAPI _tWinMain(HINSTANCE hInstance, HINSTANCE /*hPrevInstance*/, LPTSTR /*lpstrCmdLine*/, int /*nCmdShow*/)
{
    //必须要调用OleInitialize来初始化运行环境
    HRESULT hRes = OleInitialize(NULL);
    SASSERT(SUCCEEDED(hRes));
//     LoadLibrary(L"E:\\soui.taobao\\richedit\\Debug\\riched20.dll");
        
    int nRet = 0; 

    SComMgr *pComMgr = new SComMgr;
    {
    //...
        //定义一个唯一的SApplication对象，SApplication管理整个应用程序的资源
        SApplication *theApp=new SApplication(pRenderFactory,hInstance);

#ifdef DLL_CORE
        //加载LUA脚本模块，注意，脚本模块只有在SOUI内核是以DLL方式编译时才能使用。
        bLoaded=pComMgr->CreateScrpit_Lua((IObjRef**)&pScriptLua);
        SASSERT_FMT(bLoaded,_T("load interface [%s] failed!"),_T("scirpt_lua"));
        theApp->SetScriptFactory(pScriptLua);
#endif//DLL_CORE
                
        //加载全局资源描述XML
        theApp->Init(_T("xml_init")); 

        {
            //创建并显示使用SOUI布局应用程序窗口,为了保存窗口对象的析构先于其它对象，把它们缩进一层。
            CMainDlg dlgMain;  
            dlgMain.Create(GetActiveWindow(),0,0,800,600);
            dlgMain.GetNative()->SendMessage(WM_INITDIALOG);
            dlgMain.CenterWindow();
            dlgMain.ShowWindow(SW_SHOWNORMAL);
            nRet=theApp->Run(dlgMain.m_hWnd);
        }

        //应用程序退出
        delete theApp; 

    //...
    }
exit:
    delete pComMgr;
    OleUninitialize();

    return nRet;
}
```

其次我们需要在XML布局中的SOUI节点下增加一个script的子节点：

```xml
<SOUI trCtx="dlg_main" title="SOUI-DEMO version:%ver%" bigIcon="LOGO:32" smallIcon="LOGO:16" width="600" height="400" appWnd="1" margin="5,5,5,5"  resizable="1" translucent="1" alpha="255">
  <skin>
    <!--局部skin对象-->
    <gif name="gif_penguin" src="gif:gif_penguin"/>
    <apng name="apng_haha" src="apng:apng_haha"/>
  </skin>
  <style>
    <!--局部style对象-->
    <class name="cls_edit" ncSkin="_skin.sys.border" margin-x="2" margin-y="2" />
  </style>
  <script src="lua:lua_test">
    <!--当没有指定src属性时从cdata段中加载脚本-->
    <![CDATA[
    function on_init(args)
        SMessageBox(0,T "execute script function: on_init", T "msgbox", 1);
    end

    function on_exit(args)
        SMessageBox(0,T "execute script function: on_exit", T "msgbox", 1);
    end

    function onEvtTest2(args)
        SMessageBox(0,T "onEvtTest2", T "msgbox", 1);
        return 1;
    end

    function onEvtTstClick(args)
        local txt3=SStringW(L"append",-1);
        local sender=toSWindow(args.sender);
        sender:GetParent():CreateChildrenFromString(L"<button pos=\"0,0,150,30\" on_command=\"onEvtTest2\">lua btn 中文</button>");
        sender:SetVisible(0,1);
        return 1;
    end
    ]]>
  </script>
  <root class="cls_dlg_frame" cache="1" on_init="on_init" on_exit="on_exit">
    <caption pos="0,0,-0,30" show="1" font="adding:8">
      <icon pos="10,8" src="LOGO:16"/>
      <text class="cls_txt_red">SOUI-DEMO version:%ver%</text>
      <imgbtn id="1" skin="_skin.sys.btn.close"    pos="-45,0" tip="close" animate="1"/>
      <imgbtn id="2" skin="_skin.sys.btn.maximize"  pos="-83,0" animate="1" />
      <imgbtn id="3" skin="_skin.sys.btn.restore"  pos="-83,0" show="0" animate="1" />
      <imgbtn id="5" skin="_skin.sys.btn.minimize" pos="-121,0" animate="1" />
      <imgbtn name="btn_menu" skin="skin_btn_menu" pos="-151,2" animate="1" />
    </caption>
    <other/>
  </root>
</SOUI>
```

在script节点中，可以使用src属性来指定脚本所在的资源文件，也可以将脚本直接使用cdata写在script节点中，但是src中指定的脚本优先。

注意，上述XML中我们还在root结点中增加了两个新的属性：on_init, on_exit，这两个属性告诉程序在界面初始化完成及界面销毁前需要执行的两个脚本函数。

实现了上面两步，在SOUI中使用脚本的准备工作已经就绪。

为了实现上图的跑马机效果，首先我们需要在一个界面中使用XML实现基本的界面元素布局：

```xml
<?xml version="1.0" encoding="utf-8"?>
<include>
  <window size="full,full" name="game_wnd" on_size="on_canvas_size" id="300">
    <text pos="0,0,-0,@30" colorBkgnd="#cccccc" colorText="#ff0000" align="center">SOUI + LUA 跑马机</text>
    <window pos="0,[0,-0,-50">
      <window pos="0,0,-64,-0" name="game_canvas" clipClient="1" colorBkgnd="#ffffff">
        <!--比赛场地-->
        <gifplayer name="player_1" float="1" skin="gif_horse">
          <text pos="0,%1" colorText="rgb(255,0,0)" font="size:20">1</text>
        </gifplayer>
        <gifplayer name="player_2" float="1" skin="gif_horse">
          <text pos="0,0" colorText="rgb(255,0,0)" font="size:20">2</text>
        </gifplayer>
        <gifplayer name="player_3" float="1" skin="gif_horse">
          <text pos="0,0" colorText="rgb(255,0,0)" font="size:20">3</text>
        </gifplayer>
        <gifplayer name="player_4" float="1" skin="gif_horse">
          <text pos="0,0" colorText="rgb(255,0,0)" font="size:20">4</text>
        </gifplayer>
        <gifplayer name="flag_win" float="1" skin="gif_win" show="0" id="400"/>
        <text pos="|0,0">赔率:</text>
        <text pos="[0,0,@20,[0" colorText="#ff0000" name="txt_rate">4</text>
        <hr pos="-1,0,-0,-0" mode="vertical" colorLine="#ff0000"/>
      </window>
      <window pos="[0,0,-0,-0">
        <window pos="0,%12.5,@64,@64" offset="0,-0.5" skin="img_coin" id="1" tip="下注1号马" on_command="on_bet">0</window>
        <window pos="0,%37.5,@64,@64" offset="0,-0.5" skin="img_coin" id="2" tip="下注2号马" on_command="on_bet">0</window>
        <window pos="0,%62.5,@64,@64" offset="0,-0.5" skin="img_coin" id="3" tip="下注3号马" on_command="on_bet">0</window>
        <window pos="0,%87.5,@64,@64" offset="0,-0.5" skin="img_coin" id="4" tip="下注4号马" on_command="on_bet">0</window>
      </window>
    </window>
    <window pos="10,[5,-0,-0" name="game_toolbar">
      <button name="btn_run" pos="0,|0,@100,@30" offset="0,-0.5" tip="run the game" on_command="on_run">run</button>
      <text pos="]-5,0,@-1,-0" offset="-1,0">现有金币:</text>
      <text pos="-64,0,@64,-0" name="txt_coins" align="center">100</text>
    </window>

  </window>
</include>
```

在上述XML中，我们使用gifplayer控件定义了4匹马，但是我们并没有使用pos属性定义它的位置，因为它的位置是在变化的，我们需要使用脚本在控制。注意上述XML中为几个按钮控件指定的on_command属性。on_command属性是窗口的点击事件属性，指定后可以执行脚本中对应的函数。

事实上在SOUI中每一个事件都定义了一个在XML中可以映射到脚本脚本函数的属性，这里简单介绍一下这个属性在哪里实现的：

```c++
class SOUI_EXP EventCmd : public TplEventArgs<EventCmd>
    {
        SOUI_CLASS_NAME(EventCmd,L"on_command")
    public:
        EventCmd(SObject *pSender):TplEventArgs<EventCmd>(pSender){}
        enum{EventID=EVT_CMD};
    };
```

通过上面代码，可以发现，这个属性实际上就是这个事件类在SOUI中的字符串名字。

下面我们看看实现这个跑马机真正使用的LUA脚本代码：

```lua
win = nil;
tid = 0;
gamewnd = nil;
gamecanvas = nil;
players = {};
flag_win = nil;

coins_all = 100;    --现有资金
coins_bet = {0,0,0,0} --下注金额
bet_rate = 4;        --赔率
prog_max     = 200;    --最大步数
prog_all = {0,0,0,0} --马匹进度

function on_init(args)
    --初始化全局对象
    win = toHostWnd(args.sender);
    gamewnd = win:GetRoot():FindChildByNameA("game_wnd",-1);
    gamecanvas = gamewnd:FindChildByNameA("game_canvas",-1);
    flag_win = gamewnd:FindChildByNameA("flag_win",-1);
    players = {
                    gamecanvas:FindChildByNameA("player_1",-1),
                    gamecanvas:FindChildByNameA("player_2",-1),
                    gamecanvas:FindChildByNameA("player_3",-1),
                    gamecanvas:FindChildByNameA("player_4",-1)
                    };
    --布局
    on_canvas_size(nil);

    math.randomseed(os.time());
    --SMessageBox(0,T "execute script function: on_init", T "msgbox", 1);
end

function on_exit(args)
    --SMessageBox(0,T "execute script function: on_exit", T "msgbox", 1);
end

function on_timer(args)
    if(gamewnd ~= nil) then

        local rcCanvas = gamecanvas:GetWindowRect2();
        local heiCanvas = rcCanvas:Height();
        local widCanvas = rcCanvas:Width();

        local rcPlayer =  players[1]:GetWindowRect2();
        local wid = rcPlayer:Width();
        local hei = rcPlayer:Height();

        local win_id = 0;
        for i = 1,4 do
            local prog = prog_all[i];
            if(prog<prog_max) then
                prog = prog + math.random(0,5);
                prog_all[i] = prog;
                local rc = players[i]:GetWindowRect2();
                rc.left = rcCanvas.left + (widCanvas-wid)*prog/prog_max;
                players[i]:Move2(rc.left,rc.top,-1,-1);
            else
                win_id = i;

                local rc = players[i]:GetWindowRect2();
                rc.left = rcCanvas.left + (widCanvas-wid);
                players[i]:Move2(rc.left,rc.top,-1,-1);
            end
        end

        if win_id ~= 0 then
            gamewnd:FindChildByNameA("btn_run",-1):FireCommand();
            coins_all = coins_all + coins_bet[win_id] * 4;
            gamewnd:FindChildByNameA("txt_coins",-1):SetWindowText(T(coins_all));

            coins_bet = {0,0,0,0};

            local rcPlayer = players[win_id]:GetWindowRect2();
            local szFlag = flag_win:GetDesiredSize(rcPlayer);
            rcPlayer.right = rcPlayer.left + szFlag.cx;
            rcPlayer.bottom = rcPlayer.top + szFlag.cy;
            rcPlayer:OffsetRect(-szFlag.cx,-szFlag.cy/3);

            flag_win:Move(rcPlayer);
            flag_win:SetVisible(1,1);
            flag_win:SetUserData(win_id);

            for i= 1,4 do
                gamewnd:FindChildByID(i,-1):SetWindowText(T("0"));
            end
        end
    end
end

function on_bet(args)
    if tid ~= 0 then
        return 1;
    end

    local btn = toSWindow(args.sender);
    if coins_all >= 10 then
        id = btn:GetID();
        coins_bet[id] = coins_bet[id] + 10;
        coins_all = coins_all -10;
        btn:SetWindowText(T(coins_bet[id]));

        gamewnd:FindChildByNameA("txt_coins",-1):SetWindowText(T(coins_all));

    end
    return 1;
end

function on_canvas_size(args)
    if win == nil then
        return 0;
    end

    local rcCanvas =  gamecanvas:GetWindowRect2();
    local heiCanvas = rcCanvas:Height();
    local widCanvas = rcCanvas:Width();

    local szPlayer = players[1]:GetDesiredSize(rcCanvas);

    local wid = szPlayer.cx;
    local hei = szPlayer.cy;

    local rcPlayer = CRect(0,0,wid,hei);
    local interval = (heiCanvas - hei*4)/5;
    rcPlayer:OffsetRect(rcCanvas.left,rcCanvas.top+interval);
    for i = 1, 4 do
        local rc = rcPlayer;
        rc.left = rcCanvas.left + (widCanvas-wid)*prog_all[i]/prog_max;
        rc.right = rc.left+wid;
        players[i]:Move(rc);
        rcPlayer:OffsetRect(0,interval+hei);
    end

    local win_id = flag_win:GetUserData();
    if win_id ~= 0 then
        local rcPlayer = players[win_id]:GetWindowRect2();
        local szFlag = flag_win:GetDesiredSize(rcPlayer);
        flag_win:Move2(rcPlayer.left-szFlag.cx,rcPlayer.top-szFlag.cy/3,-1,-1);
    end

    return 1;

end

function on_run(args)
    local btn = toSWindow(args.sender);
    if tid == 0 then
        prog_all = {0,0,0,0};
        on_canvas_size(nil);
        tid = win:setInterval("on_timer",200);
        btn:SetWindowText(T"stop");
        flag_win:SetVisible(0,1);
    else
        win:clearTimer(tid);
        btn:SetWindowText(T"run");
        tid = 0;
    end
    return 1;
end

function on_btn_select_cbx(args)
    local btn = toSWindow(args.sender);
    local cbxwnd = btn:GetWindow(2);--get previous sibling
    local cbx = toComboboxBase(cbxwnd);
    cbx:SetCurSel(-1);
end
```

在on_init中，我们获得必须的几个UI控件，并保存到LUA的全局变量中。

在on_canvas_size中，我们根据UI大小，自动调整4匹马的位置。

然后在几个按钮的事件响应函数中响应用户操作。

至此一个简单的跑马机效果就完成了。

也许有人觉得这个示例太简单，但请想象一下，采用类似的方法，以后UI及逻辑都可以在服务器控件，客户端就像网页一样每天都可以从服务器更新界面会是什么场景，那时客户端可真就成了客户了。

后记：在SOUI中实现和网页开发类似的脚本功能算是SOUI 2015年非常重要一次更新，本来早就想写一篇这样的博客来介绍，无奈由于原来使用的lua_tinker 0.5c版本导出C++类到LUA还有这样那样的问题，一直没有找到合适的解决方案。

虽然也有比较成熟的方法如，luabind, tolua++等，但是总觉得太庞大，不适合SOUI这个非常轻量级的UI库。

这期间也尝试了如luawrapper, fflua，以及网上找到的别人修改的各版本lua_tinker，但都不理想。

使用其它导出方法就不说了，我也没有深入。使用lua_tinker主要的问题就是导出的子类不能正常调用基类的方法，这一点很头痛，虽然用变通的方法可以获得调用基类方法的能力，但是看起来有点背叛了OOP的思想。

直到今天才搞明白不能调用基类成员最主要的问题来自C++对象的多继承，明白了这个问题才能规避问题，才敢把这个LUA模块拿出来讲解，也顺便把SOUI中使用的LUA内核和5.1升级到了5.2.3。

 
