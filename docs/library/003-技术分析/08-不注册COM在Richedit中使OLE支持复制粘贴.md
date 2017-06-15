原文链接：[《不注册COM在Richedit中使OLE支持复制粘贴》](http://www.cnblogs.com/setoutsoft/p/5240324.html)

正常情况下在`Richedit`中使用OLE，如果需要OLE支持复制粘贴，那么这个OLE对象必须是已经注册的COM对象。

注册COM很简单，关键问题在于注册时需要管理员权限，这样一来，如果希望APP做成绿色版本就不好使了。

为什么需要注册成COM？因为在粘贴时`Richedit`需要能够从COM对象的GUID实例化出你的OLE对象。

从一个COM的`GUID`创建一个COM对象，必然需要通过`CoCreateInstance(Ex)`这个系统API。那么我们是不是只要Hook到这个API就可以不需要注册了呢？

通过`Hook CoCreateInstance`，我们发现创建已经注册的COM确实会到`CoCreateInstance`这个API里来。然而在试图粘贴未注册的COM对象时，确并没有走到自己Hook的`CoCreateInstance`，粘贴并没有成功。

 

为什么呢？

既然是粘贴，我们可以先看一下剪贴板里有些什么东西，随便找一个剪贴板查看的工具，看一下里面的RTF格式里有些什么东西。当复制的是注册的COM时，RTF里有这个COM的字符串ID，而当复制没有注册的COM时，剪贴板里没有这些信息。

问题出在哪呢？

有Richedit的原代码就好了。

正好有一份Wince里的Richedit的源代码，通过分析`Richedit`的复制的代码，可以发现在复制OLE对象时，先要调用`ProgIDFromCLSID`来查询这个对象的ProgID。

如果一个OLE对象没有注册，那么ProgIDFromCLSID会返回失败，从而导致复制阶段就失败了。

知道了这个流程就好办了，继续HOOK，把`ProgIDFromCLSID`加到HOOK表就好了。

HOOK了这个函数后发现粘贴时能够执行`CoCreateInstance`了，我们也可以自己实例化这个等待粘贴的对象了，但事实是还是粘贴失败了，因为在执行这个对象的Load方法前还没有设置ClientSite对象，而我的这个表情对象需要从这个`ClintSite`来`QueryInterface`出一个自己定义的接口，没有这个接口对象就没有办法初始化。

为什么呢？为什么呢？

如果有Windows的原代码查一下就好了。

对了，可以看看Wine里`OleLoad`是怎么做的（通过调用栈可以知道`Richedit`里直接调用的是`OleLoad`)。Wine是一个在`Linux`上运行Windows程序的开源框架，里面有各种`Windows API`的实现，虽然和Windows还是不一样，但大体流程差不多了。

https://source.winehq.org/ 这个网站不错，想看哪个API的实现搜索一下就出来。

```c++
/******************************************************************************
*              OleLoad        [OLE32.@]
 */
HRESULT WINAPI OleLoad(
  LPSTORAGE       pStg,
  REFIID          riid,
  LPOLECLIENTSITE pClientSite,
  LPVOID*         ppvObj)
{
  IPersistStorage* persistStorage = NULL;
  IUnknown*        pUnk;
  IOleObject*      pOleObject      = NULL;
  STATSTG          storageInfo;
  HRESULT          hres;

  TRACE("(%p, %s, %p, %p)\n", pStg, debugstr_guid(riid), pClientSite, ppvObj);

  *ppvObj = NULL;

  /*
   * TODO, Conversion ... OleDoAutoConvert
   */

  /*
   * Get the class ID for the object.
   */
  hres = IStorage_Stat(pStg, &storageInfo, STATFLAG_NONAME);
  if (FAILED(hres))
    return hres;

  /*
   * Now, try and create the handler for the object
   */
  hres = CoCreateInstance(&storageInfo.clsid,
              NULL,
              CLSCTX_INPROC_HANDLER|CLSCTX_INPROC_SERVER,
              riid,
              (void**)&pUnk);

  /*
   * If that fails, as it will most times, load the default
   * OLE handler.
   */
  if (FAILED(hres))
  {
    hres = OleCreateDefaultHandler(&storageInfo.clsid,
                   NULL,
                   riid,
                   (void**)&pUnk);
  }

  /*
   * If we couldn't find a handler... this is bad. Abort the whole thing.
   */
  if (FAILED(hres))
    return hres;

  if (pClientSite)
  {
    hres = IUnknown_QueryInterface(pUnk, &IID_IOleObject, (void **)&pOleObject);
    if (SUCCEEDED(hres))
    {
        DWORD dwStatus;
        hres = IOleObject_GetMiscStatus(pOleObject, DVASPECT_CONTENT, &dwStatus);
    }
  }

  /*
   * Initialize the object with its IPersistStorage interface.
   */
  hres = IUnknown_QueryInterface(pUnk, &IID_IPersistStorage, (void**)&persistStorage);
  if (SUCCEEDED(hres))
  {
    hres = IPersistStorage_Load(persistStorage, pStg);

    IPersistStorage_Release(persistStorage);
    persistStorage = NULL;
  }

  if (SUCCEEDED(hres) && pClientSite)
    /*
     * Inform the new object of its client site.
     */
    hres = IOleObject_SetClientSite(pOleObject, pClientSite);

  /*
   * Cleanup interfaces used internally
   */
  if (pOleObject)
    IOleObject_Release(pOleObject);

  if (SUCCEEDED(hres))
  {
    IOleLink *pOleLink;
    HRESULT hres1;
    hres1 = IUnknown_QueryInterface(pUnk, &IID_IOleLink, (void **)&pOleLink);
    if (SUCCEEDED(hres1))
    {
      FIXME("handle OLE link\n");
      IOleLink_Release(pOleLink);
    }
  }

  if (FAILED(hres))
  {
    IUnknown_Release(pUnk);
    pUnk = NULL;
  }

  *ppvObj = pUnk;

  return hres;
}
```

上面是`Wine 1.9.4`里`OleLoad`的源代码。

可以看到在执行`IPersistStorage_Load`前会先调用`IOleObject_GetMiscStatus`这个方法，然而从代码来看它并没有什么用啊？

哦，我差点忘记了，这是`Wine`，并不是真正的Windows的代码。赶紧查一下`IOleObject_GetMiscStatus`应该返回什么。

不看不知道，一看吓一跳，原来这里有一个`OLEMISC_SETCLIENTSITEFIRST`这个标志，看名字就知道有这个标志时，会先调用`SetClientSite`再调用`Load`。

好了，改写OLE的这个方法，不去查注册表，直接返回这个标志就好了。

到这里，一个不需要注册的OLE对象就完成了。

写这么多，希望看到的人能够有所启发。