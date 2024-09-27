本文是学习[**恋恋风辰Zack**](https://space.bilibili.com/271469206)asio网络编程的学习记录
前文解决了粘包问题，但是我们对于数据的大小序，我们还没有进行动手解决，此处我们来解决大小序问题，同时对之前的单线程服务器进行一定的总结
### 单线程服务器的总结
![流程图](https://i0.hdslb.com/bfs/note/8367579a4c89df62c5aaac54eca2180a0ca251d8.png)

如图应用层我们有：async_read, io_context.run, 还有应用层的ReadHandler1
asio网络层我们有：io_context。
当有一个async_read、async_write的时候，我们的异步读写会向io_context注册事件，同时会把读写的回调函数也进行注册。
我们的Io_context模型在Windows和Linux是不同的，Windows是Iocp模型，Linux是epoll模型。
我们的IO_context会一直不停地run，可以理解为是一直在进行while(1)，当我们的异步函数注册的时候，我们将其回调函数放入事件队列中。
当有读写事件发生的时候，我们就把队头弹出，然后立马调用回调函数。
因为我们的服务器消息模型在回调函数还是Handle_read，因此箭头朝向了async_read，形成了一个闭环的操作。
### 字节序
对于高位字节存储在低位地址，而地位字节存储在高位地址，那我们就称为大端序
小端序则相反
网络编程主要采用大端序
我们可以用如下的程序来区分自己主机是大端序还是小端序
```cpp
#include <iostream>
using namespace std;
// 判断当前系统的字节序是大端序还是小端序
bool is_big_endian() {
    int num = 1;
    if (*(char*)&num == 1) {
        // 当前系统为小端序
        return false;
    } else {
        // 当前系统为大端序
        return true;
    }
}
int main() {
    int num = 0x12345678;
    char* p = (char*)&num;
    cout << "原始数据：" << hex << num << endl;
    if (is_big_endian()) {
        cout << "当前系统为大端序" << endl;
        cout << "字节序为：";
        for (int i = 0; i < sizeof(num); i++) {
            cout << hex << (int)*(p + i) << " ";
        }
        cout << endl;
    } else {
        cout << "当前系统为小端序" << endl;
        cout << "字节序为：";
        for (int i = sizeof(num) - 1; i >= 0; i--) {
            cout << hex << (int)*(p + i) << " ";
        }
        cout << endl;
    }
    return 0;
}
```
对于num而言，其二进制应该是0x00000001，如果是小端序，那其存储方式为

| 01 | 00 | 00 |00|
|-|-|-|-|
| 地址0x01| 地址0x02 | 地址0x03 | 地址0x04|

```c++
*(char*)&num == 1 
```
我们先对其取地址，强制转换为(char*)，这样对应的就是首地址，然后解引用。
boost::asio库有其对应的函数进行本地序转换为网络序，网络序转换为本地序
```cpp
boost::asio::detail::socket_ops::host_to_network_long()
boost::asio::detail::socket_ops::host_to_network_short()
boost::asio::detail::socket_ops::network_to_host_long()
boost::asio::detail::socket_ops::network_to_host_short()
```

### 修改代码
服务器的代码如下：
按照上面的理念我们进行代码的修改
```cpp
//在Handle_read函数中

				int data_len = 0;
				memcpy(&data_len, _recv_hand_node->_data, HEAD_LENGTH);
				data_len = boost::asio::detail::socket_ops::network_to_host_long(data_len);
				std::cout << "data_length is: " << data_len << "\n";
//
	MsgNode(char* msg, int max_len)//发送用
		:_total_len{ max_len + HEAD_LENGTH }, _cur_len{ 0 }
	{
		_data = new char[_total_len + 1]();
		std::cout << "The total_len is: " << _total_len << "\n";
		int max_len_host = boost::asio::detail::socket_ops::network_to_host_long(max_len);
		memcpy(_data, &max_len_host, HEAD_LENGTH);
		memcpy(_data + HEAD_LENGTH, msg, max_len);
		_data[_total_len] = '\0';
	}
```
客户端的代码如下：
```cpp
#include<boost/asio.hpp>
#include <iostream>
constexpr int MAX_LENGTH = 1024 * 2;
constexpr int HANDLE_LENGTH = 4;
using namespace boost::asio::ip;
int main()
{

	try
	{
		boost::asio::io_service ioc;
		tcp::socket socket(ioc);
		tcp::endpoint ep(address::from_string("127.0.0.1"), 10086);
		boost::system::error_code error = boost::asio::error::host_not_found;
		socket.connect(ep, error);
		if (error)
		{
			std::cout << "Connect failed, The code is" << error.value()
				<< "The message is" << error.message() << "\n";
			return 0;
		}
		std::cout << "Enter the message\n";
		char request[MAX_LENGTH];
		std::cin.getline(request, MAX_LENGTH);
		std::size_t request_length = std::strlen(request);
		char send_data[MAX_LENGTH] = { 0 };
		int request_host_length = boost::asio::detail::socket_ops::host_to_network_long(request_length);
		memcpy(send_data, &request_host_length, 4);//对于发送的数字进行大小序端
		memcpy(send_data + 4, request, request_length);//这个不需要
		boost::asio::write(socket, boost::asio::buffer(send_data, request_length + 4));

		char reply_head[HANDLE_LENGTH];
		std::size_t reply_length = boost::asio::read(socket, boost::asio::buffer(reply_head, HANDLE_LENGTH));
		int msglen = 0;
		memcpy(&msglen, reply_head, HANDLE_LENGTH);
		msglen = boost::asio::detail::socket_ops::network_to_host_long(msglen);
		char msg[MAX_LENGTH] = { 0 };
		std::size_t msg_length = boost::asio::read(socket, boost::asio::buffer(msg, msglen));
		std::cout << "msg_length" << msg_length << "\n";
		std::cout << "Reply is: ";
		std::cout.write(msg, msglen);
		std::cout << "\n";
		std::cout << "Reply length is: " << msglen << "\n";
		getchar();
	}
	catch (const std::exception& e)
	{
		std::cerr << "Expection:" << e.what() << "\n";
	}
	return 0;
}


```
