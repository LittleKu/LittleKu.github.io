### 前言
前几天跟技术群小伙伴略微激烈的讨论了一下Windows下如果做一个中间组件，要用哪种方式比较好，从而回忆起几年前做过的一件事情，把公司的工程从VS2005升级到VS2019，历时了大半年，现在就其中一个问题做一个总结。

### 发现
打开VS2022，并创建了一个 DLL 工程和一个 EXE 工程；DLL 通过一个导出函数导出了一个接口，大致如下：
```cpp
// 简单定义接口文件
class IComponent
{
public:
	virtual unsigned long AddRef() = 0;
	virtual unsigned long Release() = 0;
	virtual void Print(const char *string) = 0;
};

// 在源文件简单实现一下接口功能
class ComponentImpl : public IComponent
{
	unsigned long ref_count_;
public:
	ComponentImpl():ref_count_(0) {}
	~ComponentImpl() {};

	unsigned long AddRef() {
		return ref_count_++;
	}

	unsigned long Release() {
		if (--ref_count_ == 0) {
			delete this;
			return 0;
		}
		return ref_count_;
	}

	void Print(const char *string) {
		printf("the print string is :\"%s\"\n", string);
	}
};

// 再实现一个导致接口的导出函数
__declspec(dllexport) bool ExportInterface(IComponent ** ppvObject) {
  if (ppvObject) {
    *ppvObject = new ComponentImpl();
    (*ppvObject)->AddRef();
    return true;
  }
  return false;
}
```
EXE 工程大致代码：
```cpp
int main(int argc, char ** argv) {
  IComponent *obj = NULL;
  ExportInterface(&obj);
  obj->Print("test string");
  obj->Release();

  return EXIT_SUCCESS;
}
```
至此准备工作完成。以目前的代码实现，在我的认知里不管用哪种链接方式去链接运行时库，都不会有什么问题。当我把 EXE 代码的 ``` obj->Release()``` 改成 ``` delete obj ``` 时，
预期应该是在 ```delete obj;```这行崩溃的，但并没有发生，印象中不是这样的。立即用调试器验证一下
![img](https://img2023.cnblogs.com/blog/567356/202406/567356-20240627200706308-893687744.png)
从命中的断点及对应的数据得出结论：```不管是 DLL 还是 EXE 使用的堆都是同一个堆```
从代码上看直接调用的是 ```GetProcessHeap()```，跟进这个函数发现
![img](https://img2023.cnblogs.com/blog/567356/202406/567356-20240627202315161-1933134753.png)
是 ```TEB+0x30``` 从数据结构上看是进程环境块
![img](https://img2023.cnblogs.com/blog/567356/202406/567356-20240628085058425-350000682.png)
再基于进程环境块偏移 ```0x18``` 得到该进程的堆
![img](https://img2023.cnblogs.com/blog/567356/202406/567356-20240628085304029-144878098.png)
我用同样的思路在VS2013的环境尝试一次后，得到同样的结果，但跟我以前的认知不一致，于是，我用同样的代码，在VS2005环境测试，果不其然，在 ```delete obj``` 此出现崩溃，跟进其CRT实现代码
![img](https://img2023.cnblogs.com/blog/567356/202406/567356-20240628090937815-486478289.png) 得出如果在用2005开发环境采用MT模式情况下，各自模块内部都会创建各自的堆，所以A模块的堆内存不能在B模块的堆管理器释放，至此，问题基本得到解决。
###结论
早期版本的开发环境的堆管理与新版本实现不一样；在早期版本中不同的运行时链接方式，会产生不同的堆管理句柄，那不同堆产生的内存显然不能相互释放，否则会导致堆破坏等错误；而新版本(仅从VS2013往后版本，其它环境未验证，个人观点)在一般情况下都统一使用了进程的堆句柄，这样一来不管使用哪种链接方式，最终其堆管理器都是统一的，就不存在跨堆释放的问题