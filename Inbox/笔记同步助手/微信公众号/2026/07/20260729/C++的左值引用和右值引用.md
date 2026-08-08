---
author: C加加幻想
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzk0MDc4NjcyMw==&mid=2247485907&idx=1&sn=a8a6e1278e66c3ed61a0649cb682e290&chksm=c30ae0f0da445d01cb5b896c92e97aff172733270d1243eb59bc8085cca0f58e3f69cf52d104&mpshare=1&scene=1&srcid=0729Kw0SETChUwfyBfa5bvqT&sharer_shareinfo=6716eef53cd1829498882260b095bcf3&sharer_shareinfo_first=6716eef53cd1829498882260b095bcf3#rd
saved: 2026-07-29 09:44:22
tags:
  - 笔记同步助手
id: 7e7aaafa-5df3-4973-aeaf-b89cd35987df
---

公众号名称：C++幻想

作者名称：C加加幻想

发布时间：2026-07-27 07:00

在C++中，一个作为引用的命名变量可以认为是一个已经存在对象或函数的别名。

一个& 声明的变量就是引用，但在C++11开始引入了右值引用概念，所以，一个&声明的变量通常就称为左值引用；两个&声明的变量称为右值引用。看下面的例子 demo1:

> ```
> #include <iostream>
> #include <type_traits>
> 
> #define PRINT_VARIABLE_INFO(v)  \
>     if (std::is_lvalue_reference<decltype(v)>::value)   \
>         std::cout << "变量 " << #v << " 是左值引用" << std::endl; \
>     else if (std::is_rvalue_reference<decltype(v)>::value)   \
>         std::cout << "变量 " << #v << " 是右值引用" << std::endl; \
>     else std::cout << "变量 " << #v << " 不是引用" << std::endl;
> 
> using left_ref = int&;
> using right_ref = int&&;
> 
> int main() {
>     int v0 = 8;
> 
>     // v1 v2 v3 都是左值引用
>     left_ref v1 = v0;
>     left_ref& v2 = v0;
>     left_ref&& v3 = v0;
> 
>     // v4 也是左值引用
>     right_ref& v4 = v0;
>     
>     // v5 v6 是右值引用
>     // v0 是左值，所以 v5和v6 都不能使用v0初始化
>     right_ref v5 = 0;
>     right_ref&& v6 = 0;
> 
>     PRINT_VARIABLE_INFO(v0);
>     PRINT_VARIABLE_INFO(v1);
>     PRINT_VARIABLE_INFO(v2);
>     PRINT_VARIABLE_INFO(v3);
>     PRINT_VARIABLE_INFO(v4);
>     PRINT_VARIABLE_INFO(v5);
>     PRINT_VARIABLE_INFO(v6);
> 
>     return0;
> }
> ```

demo1的运行结果：

> ```
> 变量 v0 不是引用
> 变量 v1 是左值引用
> 变量 v2 是左值引用
> 变量 v3 是左值引用
> 变量 v4 是左值引用
> 变量 v5 是右值引用
> 变量 v6 是右值引用
> ```

上面的例子中，重点关注v3和v4的类型。v3可以认为是左值引用后的右值引用、v4可以认为是右值引用后的左值引用。这个概念在后面的demo4和demo5中有用，这篇文章就不展开了。

另外，上面代码中如果要使用v0给v5或v6初始化，可以使用函数std::move把变量转为右值(附录一)。

右值引用可以延长匿名变量的生命周期，看下面的例子 demo2:

> ```
> #include <iostream>
> 
> class MyObject {
> public:
>     MyObject(int v): v_(v) {
>         std::cout << "Create MyObject -- v_=" << v_ << std::endl;
>     }
> 
>     MyObject(const MyObject& other): v_(other.v_) {
>         std::cout << "Create MyObject --  Copy v_=" << v_ << std::endl;
>     }
> 
>     MyObject(MyObject&& other): v_(other.v_) {
>         std::cout << "Create MyObject --  Move v_=" << v_ << std::endl;
>     }
> 
>     ～MyObject() {
>         std::cout << "Destroy MyObject -- v_=" << v_ << std::endl;
>     }
> 
>     // 这两个方法在例子中不需要
>     // MyObject& operator=(const MyObject& other);
>     // MyObject& operator=(MyObject&& other);
> 
>     friend MyObject operator+(const MyObject& other1, const MyObject& other2) {
>         return MyObject(other1.v_ + other2.v_);
>     }
> 
> private:
>     int v_;
> };
> 
> int main() {
>     MyObject o1(1);
>     MyObject o2(2);
> 
>     {
>         std::cout << "\n===== Step 1 -- begin =====" << std::endl;
> 
>         o1 + o2;
> 
>         std::cout << "===== Step 1 -- end   =====" << std::endl;
>     }
> 
>     {
>         std::cout << "\n===== Step 2 -- begin =====" << std::endl;
> 
>         MyObject MyObject = o1 + o2;
> 
>         std::cout << "===== Step 2 -- end   =====" << std::endl;
>     }
> 
>     {
>         std::cout << "\n===== Step 3 -- begin =====" << std::endl;
> 
>         MyObject&& MyObject = o1 + o2;
> 
>         std::cout << "===== Step 3 -- end   =====" << std::endl;
>     }
> 
>     {
>         std::cout << "\n===== Step 4 -- begin =====" << std::endl;
> 
>         // 下面，这句会编译失败
>         // MyObject& MyObject = o1 + o2;
> 
>         std::cout << "===== Step 4 -- end   =====" << std::endl;
>     }
> 
>     return0;
> }
> ```

demo2的运行结果：

> ```
> Create MyObject -- v_=1
> Create MyObject -- v_=2
> 
> ===== Step 1 -- begin =====
> Create MyObject -- v_=3
> Destroy MyObject -- v_=3
> ===== Step 1 -- end   =====
> 
> ===== Step 2 -- begin =====
> Create MyObject -- v_=3
> ===== Step 2 -- end   =====
> Destroy MyObject -- v_=3
> 
> ===== Step 3 -- begin =====
> Create MyObject -- v_=3
> ===== Step 3 -- end   =====
> Destroy MyObject -- v_=3
> 
> ===== Step 4 -- begin =====
> ===== Step 4 -- end   =====
> Destroy MyObject -- v_=2
> Destroy MyObject -- v_=1
> ```

从上面的输出可以看到，Step2和Step3的方式都延长了生命周期。Step3是右值引用的合理应用场景，Step2是因为编译优化导致(附录二)。

但很多时候，并不能指望编译优化，例如下面例子 demo3:

> ```
> #include <iostream>
> 
> class MyObject {
> public:
>     MyObject(int v): v_(v) {
>         std::cout << "Create MyObject -- v_=" << v_ << std::endl;
>     }
> 
>     MyObject(const MyObject& other): v_(other.v_) {
>         std::cout << "Create MyObject --  Copy v_=" << v_ << std::endl;
>     }
> 
>     MyObject(MyObject&& other): v_(other.v_) {
>         std::cout << "Create MyObject --  Move v_=" << v_ << std::endl;
>     }
> 
>     ～MyObject() {
>         std::cout << "Destroy MyObject -- v_=" << v_ << std::endl;
>     }
> 
>     friend MyObject operator+(const MyObject& other1, const MyObject& other2) {
>         return MyObject(other1.v_ + other2.v_);
>     }
> 
> public:
>     void print() {
>         std::cout << "MyObject::print v_:" << v_ << std::endl;
>     }
> 
> private:
>     int v_;
> };
> 
> void f1(MyObject obj) {
>     obj.print();
> }
> 
> void g1(MyObject obj) {
>     f1(obj);
> }
> 
> void f2(MyObject&& obj) {
>     obj.print();
> }
> 
> void g2(MyObject&& obj) {
>     f2(std::move(obj));
> }
> 
> void f3(MyObject& obj) {
>     obj.print();
> }
> 
> void g3(MyObject& obj) {
>     f3(obj);
> }
> 
> int main() {
>     MyObject o1(1);
>     MyObject o2(2);
> 
>     std::cout << "===== main   -- begin =====" << std::endl;
> 
>     {
>         std::cout << "\n===== Step 1 -- begin =====" << std::endl;
> 
>         g1(o1 + o2);
> 
>         std::cout << "===== Step 1 -- end   =====" << std::endl;
>     }
> 
>     {
>         std::cout << "\n===== Step 2 -- begin =====" << std::endl;
> 
>         g2(o1 + o2);
> 
>         std::cout << "===== Step 2 -- end   =====" << std::endl;
>     }
> 
>     {
>         std::cout << "\n===== Step 3 -- begin =====" << std::endl;
> 
>         // g3(o1 + o2); 会编译失败
>         // 只能这么用
>         MyObject o3 = o1 + o2;
>         g3(o3);
> 
>         std::cout << "===== Step 3 -- end   =====" << std::endl;
>     }
> 
>     std::cout << "\n===== main   -- end   =====" << std::endl;
> 
>     return0;
> }
> ```

demo3的运行结果:

> ```
> Create MyObject -- v_=1
> Create MyObject -- v_=2
> ===== main   -- begin =====
> 
> ===== Step 1 -- begin =====
> Create MyObject -- v_=3
> Create MyObject --  Copy v_=3
> MyObject::print v_:3
> Destroy MyObject -- v_=3
> Destroy MyObject -- v_=3
> ===== Step 1 -- end   =====
> 
> ===== Step 2 -- begin =====
> Create MyObject -- v_=3
> MyObject::print v_:3
> Destroy MyObject -- v_=3
> ===== Step 2 -- end   =====
> 
> ===== Step 3 -- begin =====
> Create MyObject -- v_=3
> MyObject::print v_:3
> ===== Step 3 -- end   =====
> Destroy MyObject -- v_=3
> 
> ===== main   -- end   =====
> Destroy MyObject -- v_=2
> Destroy MyObject -- v_=1
> ```

注意demo3的Step1，过程中调用了拷贝构造(创建了一个临时的中间对象)。

有兴趣的可以对比一下Step1、Step2和Step3的效果。

但思考一下demo3中的f1、f2和f3，为何不使用一个模版函数呢？当然，g1、g2和g3三个函数也一样。故，有了下面的例子demo4:

> ```
> #include <iostream>
> #include <string>
> 
> template<class T>
> void f(T&& text) {
>     if (std::is_lvalue_reference<decltype(text)>::value)
>         std::cout << "function f 参数text是左值引用" << std::endl;
>     elseif (std::is_rvalue_reference<decltype(text)>::value)
>         std::cout << "function f 参数text是右值引用" << std::endl;
> 
>     std::cout << "function f -- " << text << std::endl;
> }
> 
> template<class T>
> void g(T&& text) {
>     if (std::is_lvalue_reference<decltype(text)>::value)
>         std::cout << "function g 参数text是左值引用" << std::endl;
>     elseif (std::is_rvalue_reference<decltype(text)>::value)
>         std::cout << "function g 参数text是右值引用" << std::endl;
> 
>     f(std::move(text));
> }
> 
> int main() {
>     std::string text = "test";
> 
>     g(text);
> 
>     std::cout << "============" << std::endl;
> 
>     g(std::move(text));
> 
>     return0;
> }
> ```

从demo3的效果看，右值引用的参数效果最好，所以，在demo4中，函数f和g的参数都是右值引用(T&&)。为了检测这种方法是否有问题(打印这个信息，就说明这个地方有问题呗)，在函数f和g的实现中，都打印了参数的实际类型(都说了是右值引用，怎么有说看实际类型，这就是demo1的意义啦)。

下面看一下demo4的运行效果:

> ```
> function g 参数text是左值引用
> function f 参数text是右值引用
> function f -- test
> ============
> function g 参数text是右值引用
> function f 参数text是右值引用
> function f -- test
> ```

从这个输出结果看，是右问题的。当函数g接受的参数是右值引用时，函数f接受的参数也是右值引用，这符合预期。但是，当函数g接受的参数是左值引用时，函数f接受的参数还是右值引用，这不合理(但实际上是符合预期的，std::move函数就是转为右值引用呀，见附录一)。

所以，应该修改函数g的实现，使用完美转发，见下面demo5:

> ```
> #include <iostream>
> #include <string>
> 
> template<class T>
> void f(T&& text) {
>     if (std::is_lvalue_reference<decltype(text)>::value)
>         std::cout << "function f 参数text是左值引用" << std::endl;
>     elseif (std::is_rvalue_reference<decltype(text)>::value)
>         std::cout << "function f 参数text是右值引用" << std::endl;
> 
>     std::cout << "function f -- " << text << std::endl;
> }
> 
> template<class T>
> void g(T&& text) {
>     if (std::is_lvalue_reference<decltype(text)>::value)
>         std::cout << "function g 参数text是左值引用" << std::endl;
>     elseif (std::is_rvalue_reference<decltype(text)>::value)
>         std::cout << "function g 参数text是右值引用" << std::endl;
> 
>     // 使用完美转发
>     f(std::forward<T>(text));
> }
> 
> int main() {
>     std::string text = "test";
> 
>     g(text);
> 
>     std::cout << "============" << std::endl;
> 
>     g(std::move(text));
> 
>     return0;
> }
> ```

demo5的运行效果:

> ```
> function g 参数text是左值引用
> function f 参数text是左值引用
> function f -- test
> ============
> function g 参数text是右值引用
> function f 参数text是右值引用
> function f -- test
> ```

左值引用作为(函数)返回值的情况(非常常见)，但要注意被引用的对象的生命周期，这种使用方式，不会延长生命周期。而右值引用作为(函数)返回值的使用场景比较少见。例如：下面的函数：

> ```
> std::string&& f() {
>   std::string text;
>   // 一些对 text 内容的处理
>   return std::move(text);
> }
> ```

实际上是有风险的(看编译警告吧)，为何不写成下面这样：

> ```
> std::string f() {
>   std::string text;
>   // 一些对 text 内容的处理
>   return text;
> }
> ```

更简单直接，也合理。

当然，一个合理的原型是这样的：

> ```
> std::string f(std::string&& text) {
>   // 一些对 text 内容的处理
>   return std::move(text);
> }
> ```

它可以这么用：

> ```
> std::string&& result = f("hello");
> ```

可以认为 result延长了 以"hello"作为参数创建的std::string实例的生命周期。

附录：

附录一

关于std::move有两个原型。

一个是定义在头文件 utility 中，这个头文件一般不需要明确的包含(就是#include <utility>)，这个函数就是把对象转为右值引用。

另一个是定义在头文件 algorithm 中，这个函数通常是配合容器使用。

附录二

在demo2的Step2和Step3相比应该多一次拷贝构造，但因为编译优化，使step2的过程优化了。这个编译优化，当前的主流编译器都是默认开启的，具体可以参考[聊一聊C++的函数返回值](https://mp.weixin.qq.com/s?__biz=Mzk0MDc4NjcyMw==&mid=2247483751&idx=1&sn=b76fff1050242b6231cc01c1b7ea4ddf&scene=21#wechat_redirect)，可以按照这里的方法修改编译参数自己试一试。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/8bb63c91_1785289460885?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzk0MDc4NjcyMw%3D%3D%26mid%3D2247485907%26idx%3D1%26sn%3Da8a6e1278e66c3ed61a0649cb682e290%26chksm%3Dc30ae0f0da445d01cb5b896c92e97aff172733270d1243eb59bc8085cca0f58e3f69cf52d104%26mpshare%3D1%26scene%3D1%26srcid%3D0729Kw0SETChUwfyBfa5bvqT%26sharer_shareinfo%3D6716eef53cd1829498882260b095bcf3%26sharer_shareinfo_first%3D6716eef53cd1829498882260b095bcf3%23rd&s=obsidian)