原文链接：[《第十二篇：SOUI的utilities模块为什么要用DLL编译？》](http://www.cnblogs.com/setoutsoft/p/3961808.html)

SOUI相对于DuiEngine一个重要的变化就是很多模块变成了一个单独的DLL。

然后很多情况下用户可能希望整个产品就是一个EXE，原来DuiEngine提供了LIB编译模式，此时链接LIB模式的DuiEngine就行了。

但是SOUI默认至少Utilities那个模块是不提供LIB编译模式的。

utilities之所以默认只提供DLL编译是因为SString类是由utilities实现的。

字符串是编译中碰到的最最见的基本对象之一。在运行库（CRT）动态编译（MD，MDd）时这不是问题，因为所有模块的内存分配都是在一个相同的运行库（CRT）上，这时在不同模块之间传递对象相对简单。如果项目采用运行库静态编译（MT or MTd)，在不同模块之间传递字符串对象是非常困难的，因为一不小心就会发生在A模块中分配的字符串对象被B模块释放。

utilities采用DLL编译就是为了解决这个字符串对象的跨模块传递。

采用运行库动态编译的情况就不说了，这里主要介绍采用静态库编译的CRT的情况。

SOUI中使用的字符串对象采用了一点技巧：每一个String对象中只有一个指针成员变量：

```c++
template <class tchar, class tchar_traits>
    class TStringT
    {
    public:
        typedef tchar    _tchar;
        typedef const _tchar * pctstr;

    protected:
        tchar* m_pszData;   // pointer to ref counted string data
    };
```

虽然TStringT是一个模板类，在SOUI中采用类导出的方式将该模板的两个特化类导出：

```c++
#ifdef UTILITIES_EXPORTS
#    define EXPIMP_TEMPLATE
#else
#    define EXPIMP_TEMPLATE extern
#endif

     #pragma warning (disable : 4231)

    EXPIMP_TEMPLATE template class UTILITIES_API  TStringT<char, char_traits>;
    EXPIMP_TEMPLATE template class UTILITIES_API  TStringT<wchar_t, wchar_traits>;
```

通过将string类导出，保证string的所有运行代码都是在utilities这个模块内部，这也就保证了string对象的唯一成员变量：

```c+++
tchar* m_pszData；
```

的内存分配及释放固定在utilities这个模块里。

通过这样处理，无论用户定义string是在哪一个模块，真正的内存管理还是在utilities里，从而使得string对象可以方便的在不同模块之间传递。

比较一下std::string就可以发现，如果使用`std::string`在不同模块之间传递对象将是非常危险的，因为`std::string`是模板类，它的代码将会被编译到不同的模块中，也就是说在不同的模块中调用std::string的成员函数执行的代码是不一样的，这样在A模块中声明的string传递到B模块再被B模块释放程序就崩溃了。

这就是为什么utilities模块默认只提供DLL编译的原因。

知道了原因就好办了。

对于那些希望整个项目就是一个EXE的情况，直接修改utilities模块的编译类型为LIB就行了，因为这种情况下根本不存在跨模块对象传递的问题。