本文是学习[**恋恋风辰Zack**](https://space.bilibili.com/271469206)asio网络编程的学习记录
这节我们对粘包简易化，之前的粘包流程代码比较长，而且费逻辑，我们用asio的方法来处理粘包问题。
之前我们的异步服务器API中，有使用过async_read()函数，这个等到读取完指定的字节数后才调用回调函数，因此我们可以先用async_read()读取我们的消息长度，然后再接着读取完消息的内容，这样我们就可以简易化粘包的流程了
### 修改服务器代码
我们先修改Start函数的代码
```cpp
void CSession::Start() {
	_recv_head_node->Clear();
	//std::cout << _data << "\n";
	boost::asio::async_read(_socket, boost::asio::buffer(_recv_head_node->_data, HEAD_LENGTH), std::bind(&CSession::HandleReadHead, this,
		std::placeholders::_1, std::placeholders::_2, SharedSelf()));
}
```
等到我们读取完消息头结点的信息后，我们就调用回调函数，来把信息存储到_recv_head_node节点。
接下来实现我们的HandleReadHead函数
```cpp
void CSession::HandleReadHead(const boost::system::error_code& error, size_t  bytes_transferred, std::shared_ptr<CSession> shared_self) {
	if (!error) {
		if (bytes_transferred < HEAD_LENGTH) {
			cout << "read head lenth error";
			Close();
			_server->ClearSession(_uuid);
			return;
		}

		//头部接收完，解析头部
		short data_len = 0;
		memcpy(&data_len, _recv_head_node->_data, HEAD_LENGTH);
		cout << "data_len is " << data_len << endl;
		//此处省略字节序转换
		// ...
		//头部长度非法
		if (data_len > MAX_LENGTH) {
			std::cout << "invalid data length is " << data_len << endl;
			_server->ClearSession(_uuid);
			return;
		}
		//std::cout << _data << "\n";
		_recv_msg_node = make_shared<MsgNode>(data_len);
		boost::asio::async_read(_socket, boost::asio::buffer(_recv_msg_node->_data, _recv_msg_node->_total_len),
			std::bind(&CSession::HandleReadMsg, this,
				std::placeholders::_1, std::placeholders::_2, SharedSelf()));
	}
	else {
		std::cout << "handle read failed, error is " << error.what() << endl;
		Close();
		_server->ClearSession(_uuid);
	}
}
```
我们在回调函数里面把头结点的内容复制过去，然后接着读取消息本体，然后调用HandleReadMsg。
```cpp

void CSession::HandleReadMsg(const boost::system::error_code& error, size_t  bytes_transferred, std::shared_ptr<CSession> shared_self) {
	if (!error) {
		//std::cout << _data << "\n";
		PrintRecvData(_recv_msg_node->_data, _recv_msg_node->_total_len);
		std::chrono::milliseconds dura(2000);
		std::this_thread::sleep_for(dura);
		_recv_msg_node->_data[_recv_msg_node->_total_len] = '\0';
		cout << "receive data is " << _recv_msg_node->_data << endl;
		//此处可以调用Send发送测试
		Send(_recv_msg_node->_data, _recv_msg_node->_total_len);
		//再次接收头部数据
		_recv_head_node->Clear();
		boost::asio::async_read(_socket, boost::asio::buffer(_recv_head_node->_data, HEAD_LENGTH), std::bind(&CSession::HandleReadHead, this,
			std::placeholders::_1, std::placeholders::_2, SharedSelf()));
	}
	else {
		std::cout << "handle read failed, error is " << error.what() << endl;
		Close();
		_server->ClearSession(_uuid);
	}
}
```
其中**PrintRecvData(_recv_msg_node->_data, _recv_msg_node->_total_len);**是来打印接受信息的十六进制编码的。
```cpp

void CSession::PrintRecvData(char* data, int length) {
	stringstream ss;
	string result = "0x";
	for (int i = 0; i < length; i++) {
		string hexstr;
		ss << hex << std::setw(2) << std::setfill('0') << int(data[i]) << endl;
		ss >> hexstr;
		result += hexstr;
	}
	std::cout << "receive raw data is : " << result << endl;;
}
```
### 粘包测试
为了进行粘包的测试，我们需要把客户端的代码改写成如下的模样
```cpp
#include <iostream>
#include <boost/asio.hpp>
#include <thread>
using namespace std;
using namespace boost::asio::ip;
const int MAX_LENGTH = 1024 * 2;
const int HEAD_LENGTH = 2;
int main()
{
	try {
		//创建上下文服务
		boost::asio::io_context   ioc;
		//构造endpoint
		tcp::endpoint  remote_ep(address::from_string("127.0.0.1"), 10086);
		tcp::socket  sock(ioc);
		boost::system::error_code   error = boost::asio::error::host_not_found; ;
		sock.connect(remote_ep, error);
		if (error) {
			cout << "connect failed, code is " << error.value() << " error msg is " << error.message();
			return 0;
		}

		thread send_thread([&sock] {
			for (;;) {
				this_thread::sleep_for(std::chrono::milliseconds(2));
				const char* request = "hello";
				size_t request_length = strlen(request);
				char send_data[MAX_LENGTH] = { 0 };
				memcpy(send_data, &request_length, 2);
				memcpy(send_data + 2, request, request_length);
				boost::asio::write(sock, boost::asio::buffer(send_data, request_length + 2));
			}
			});

		thread recv_thread([&sock] {
			for (;;) {
				this_thread::sleep_for(std::chrono::milliseconds(2));
				cout << "begin to receive..." << endl;
				char reply_head[HEAD_LENGTH];
				size_t reply_length = boost::asio::read(sock, boost::asio::buffer(reply_head, HEAD_LENGTH));
				short msglen = 0;
				memcpy(&msglen, reply_head, HEAD_LENGTH);
				char msg[MAX_LENGTH] = { 0 };
				size_t  msg_length = boost::asio::read(sock, boost::asio::buffer(msg, msglen));

				std::cout << "Reply is: ";
				std::cout.write(msg, msglen) << endl;
				std::cout << "Reply len is " << msglen;
				std::cout << "\n";
			}
			});

		send_thread.join();
		recv_thread.join();
	}
	catch (std::exception& e) {
		std::cerr << "Exception: " << e.what() << endl;
	}
	return 0;
}
```
我们分两个线程来进行读和写，在每个线程都sleep_for两ms，这样可以减少CPU的占用率，这样可以避免频繁的IO操作。
运行结果：
![photo](https://github.com/FKlightdog/Images/blob/main/ReadAtLeast.png?raw=true)