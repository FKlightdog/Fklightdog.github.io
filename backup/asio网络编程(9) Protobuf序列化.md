本文是学习[**恋恋风辰Zack**](https://space.bilibili.com/271469206)asio网络编程的学习记录
### Protobuf简介
Protocol Buffers（简称：ProtoBuf）是一种开源跨平台的序列化数据结构的协议。其对于存储资料或在网络上进行通信的程序是很有用的。这个方法包含一个接口描述语言，描述一些数据结构，并提供程序工具根据这些描述产生代码，这些代码将用来生成或解析代表这些数据结构的字节流。
我们可以把一些类和结构体转换为字符串，因为Tcp是面向字节流的，所以我们可以把这个转换后的字符串发给服务器。不过在实际前后端通信的时候，我们是使用JSON序列化的，因为Protobuf的字符串可读性太差了。

### Protobuf的版本
此处我们涉及的是Windows的版本，并没有涉及到Linux。我此处使用的Protobuf的版本是**protobuf--3.21.9**，可以点击此处的链接下载：https://wwwc.lanzoue.com/iAvPp2b3h90d
对于Protobuf的安装而言，3.21.9的安装是比较简单的，其以上版本安装会比较复杂一些。
如果我们要配置Protobuf的话，我们还要下载Cmake，Cmake的下载比较简单方便一些，这里就不提及了
### Cmake编译
我们解压后可以在文件目录里面创建一个新的文件夹，这个新的文件夹用来存储我们用Cmake编译的结果，我们可以用Cmake编译Debug版本和Relase版本。
我们需要的有这些：
![图片](https://cdn.llfc.club/1684118846421.jpg)
然后我们点击Config，然后再点击Generate就可以了
> [!NOTE]
> Config选择我们自己版本的编译器，我得是visualstudio17 2022。

结束后，我们进入刚才创立的文件夹，然后点击protobuf.sln，然后会出现如下的图片
![图片](https://github.com/FKlightdog/Images/blob/main/ProtobufList.png?raw=true)
然后我们点击ALL_BUILD，点击重新生成，这样我们整个库都会生成出来了，我们可以把Release版本和Debug版本都生成出来。
创建新的文件夹进行来存储我们刚刚生成的库和头文件，其中bin来储存库，include来储存头文件
![图片](https://cdn.llfc.club/1684121373828.jpg)
我们将刚才Debug目录下的所有内容都拷贝到bin目录，protobuf文件夹下src文件夹里的google文件夹及其内容拷贝到protoc的include文件夹
接下来我们就是要配置系统的环境变量，这一步就比较简单了。
先新建PROTOBUF_HOME的环境变量，变量值是我们刚才的bin目录
![图片](https://cdn.llfc.club/1684129895718.jpg)
然后再PATH变量添加PROTOBUF_HOME。
![图片](https://cdn.llfc.club/1684130311229.jpg)
为了检查我们可以用如下命令来进行检测
```powershell
protoc.exe
```
如果输出了help，那说明我们已经配置成功了
### visualstudio中使用Protobuf
我们新建一个控制台项目，在项目属性中，配置选择Debug，平台选择X64，选择VC++目录，
在包含目录中添加 选择我们刚刚创建的include
在库目录中添加 选择我们刚刚创建的bin
在链接器的输入选项中添加protobuf用到的lib库
```dotnetcli
libprotobufd.lib
libprotocd.lib
```
这样我们的visualstudio的protobuf就配置好了
### 使用protobuf
我们刚刚创建好了我们的项目，我们在项目目录那创建一个新的文件msg.proto
```protobuf
syntax = "proto3";
message Book
{
    string name = 1;
    int32 pages = 2;
    float price = 3;
}
```
创建完后，我们在其目录下面用如下的命令
```powershell
protoc --cpp_out=. ./msg.proto
```
protoc就是protoc命令 --cpp是生成C++的结果，_out=.是生成在当前的目录下面,./msg.proto就是运行这个文件。
这样我们就可以生成对应的"msg.pb.h和msg.pb.cc" 。
然后我们在项目里面载入这个头文件和cpp文件
然后我们用如下代码进行测试
```cpp
#include <iostream>
#include "msg.pb.h"
int main()
{
    Book book;
    book.set_name("CPP programing");
    book.set_pages(100);
    book.set_price(200);
    std::string bookstr;
    book.SerializeToString(&bookstr);
    std::cout << "serialize str is " << bookstr << std::endl;
    Book book2;
    book2.ParseFromString(bookstr);
    std::cout << "book2 name is " << book2.name() << " price is " 
        << book2.price() << " pages is " << book2.pages() << std::endl;
    getchar();
}
```
对于上面的我们大概率是不能直接使用的，大概率会遇见Link2001错误，对于这个错误我们需要在预处理器里面加上这个
```dotnetcli
PROTOBUF_USE_DLLS
```
如果还有问题的话，我们可以看一看平台，平台必须要是X64平台。
同时还要看一下我们C/C++的代码生成的运行库是不是多线程调试DLL。
对于代码：
book.SerializeToString(&bookstr);是将其序列化
book2.ParseFromString(bookstr);是将其反序列化
这样我们就可以正确的运行结果了
运行结果如下
```dotnetcli
serialize str is
CPP programingdHC
book2 name is CPP programing price is 200 pages is 100
```
