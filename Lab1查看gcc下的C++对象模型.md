# Lab1 查看gcc下的C++对象模型
```c++
class Test {
public:
  void TestNonStaticMethod() {}
  static void TestStaticMethod() {}
  virtual void TestVirtualMethod() {}

  static int static_data_;
private:
  int non_static_data_ = 1;
};

int Test::static_data_ = 1;
int main() {
  Test t;
  Test::static_data_ = 2;
  return 0;
}
```

编译运行后，使用gdb进行调试

```
$ g++ test.cc -o test -g
$ gdb test
```

在`return 0`这一语句位置打上断点并运行程序

```gdb
(gdb) b 18
(gdb) r
```

打印对象t的内存布局和t所占内存大小

```gdb
(gdb) p t
(gdb) p sizeof(t)
```

![](https://blog-1256131373.cos.ap-shanghai.myqcloud.com/Obsidian/20230304231620.png)

> - 可以看到对象t所占内存大小为16，且对象主要由一个虚函数表指针和非静态成员变量组成。所有函数和静态成员变量都不存在类对象实体中。
> - 虚函数表指针并非指向虚函数表最开头的位置
> 
> 注：这里使用的是64位机器，所以一个指针大小为8B，一个int大小为4B，所以还有4B是用于内存对齐的padding。
> 
> 图片中虽然打印了static static_data_ = 2，但它并不在对象所处结构中，可以通过下面打印对象地址附近的内存单元验证。

打印对象t的地址，并打印该地址附近16B数据

```gdb
(gdb) p &t
(gdb) x /2ag &t
```

![](https://blog-1256131373.cos.ap-shanghai.myqcloud.com/Obsidian/20230304231354.png)

>可以看到虚函数表指针存在对象开始的位置，非静态成员变量存在虚函数表指针后面

打印虚函数表，从上图知0x555555557da0指向虚函数表偏移16的位置，将该地址减16，并打印该地址附近单元

```gdb
(gdb) x /10ag 0x555555557d90
```

![](https://blog-1256131373.cos.ap-shanghai.myqcloud.com/Obsidian/20230304232847.png)

这样就打印出了虚函数表内容，但这样看不直观我们可以借用[Compiler Explorer (godbolt.org)](https://godbolt.org/)工具来查看

![](https://blog-1256131373.cos.ap-shanghai.myqcloud.com/Obsidian/20230304233035.png)

> 虚函数表第一项是top offset
> 
> 第二项是Test的typeinfo对象，用于RTTI（后续会讲到）
> 
> 第三项才是Test类中的虚函数的指针，而类对象中的虚函数表指针指向的也就是虚表中存放第一个虚函数指针的位置。