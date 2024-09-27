本文是学习[**恋恋风辰Zack**](https://space.bilibili.com/271469206)asio网络编程的学习记录
上一篇文章我们实现了全双工通信，但是我们还没有解决粘包问题，本文我们来解决粘包问题。
可以看[Zack的博客](https://llfc.club/articlepage?id=2PSqYnkrogKeDPjv3gdBUAcbN5P)，讲的很详细，我此刻就把代码和我的理解进行一下分享。
### 个人理解
粘包问题的出现是与TCP的底层有关的，客户端和服务器都有自己的发送缓冲区和接受缓冲区，如果此时服务器的接收缓冲区总共接受10个字节的。上一次还有5个字节还未发送，此时缓冲区还剩下5个字节，此时我们发送6个字节的数据，那可能此时收到的数据是未发生的5个字节 + 我们此时6个字节的前5个字节。
还有一种可能性是：客户端发送的频率大于此时服务器接收的频率，那此时也会出现粘包问题。
或者是客户端多频的发送小包，而TCP只有到一定范围内才会接受数据。

对于粘包问题，我们的解决办法就是定义一个协议，比如我们要发送的数据，前4个字节是长度，后面才是内容，我们通过解析这个协议，来进行拆包行为。
### 示例代码
```cpp
//CSession.h
#pragma once
#include<iostream>
#include<boost/asio.hpp>
#include<boost/uuid/uuid_generators.hpp>
#include<boost/uuid/uuid_io.hpp>
#include"CServer.h"
#include<memory>
#include<queue>
#include<mutex>
#define max_length 1024*2
#define HEAD_LENGTH 4

class CServer;

class MsgNode
{
	friend class CSession;
public:
	MsgNode(char* msg, int max_len)//发送用
		:_total_len{ max_len + HEAD_LENGTH }, _cur_len{ 0 }
	{
		_data = new char[_total_len + 1]();
		std::cout << "The total_len is: " << _total_len << "\n";
		memcpy(_data, &max_len, HEAD_LENGTH);
		memcpy(_data + HEAD_LENGTH, msg, max_len);
		_data[_total_len] = '\0';
	}

	MsgNode(int max_len)//收到用
		:_total_len{ max_len }, _cur_len{ 0 }
	{
		_data = new char[_total_len + 1]();
	}

	~MsgNode()
	{
		delete[] _data;
	}

	void Clear()
	{
		std::memset(_data, 0, _total_len);
		_cur_len = 0;
	}
private:
	char* _data;
	int _total_len;
	int _cur_len;
};

class CSession :public std::enable_shared_from_this<CSession>
{
public:
	CSession(boost::asio::io_context& ioc, CServer* server)
		:socket{ ioc }, _server{ server }, _b_head_parse{ false }
	{
		boost::uuids::uuid a_uuid = boost::uuids::random_generator()();
		_uuid = boost::uuids::to_string(a_uuid);
		_recv_hand_node = std::make_shared<MsgNode>(max_length);
	}

	~CSession()
	{
		std::cout << "Session is destroyed" << "\n";
	}

	boost::asio::ip::tcp::socket& Socket()
	{
		return socket;
	}

	std::string& Get_uuid()
	{
		return _uuid;
	}

	void start();
	void Send(char* msg, int max_len);
private:
	char data[max_length];
	CServer* _server;
	boost::asio::ip::tcp::socket socket;
	std::string _uuid;
	std::queue<std::shared_ptr<MsgNode>> _msg_queue;
	std::mutex _send_lock;
	std::shared_ptr<MsgNode> _recv_msg_node;//接收到的信息节点
	std::shared_ptr<MsgNode> _recv_hand_node;//接受到的头部节点
	bool _b_head_parse;//是否处理完头部信息
	void handle_write(const boost::system::error_code& error, std::shared_ptr<CSession> _self_Session);
	void handle_read(const boost::system::error_code& error, std::size_t bytes_transferred, std::shared_ptr<CSession> _self_Session);
};

```
我们修改了信息节点的构造函数，现在构造函数有发送用和接受用的。接受用的是没什么好说的，对于接受的：我们就按照字符串长度+字符串内容来读取信息。
```cpp

void CSession::handle_read(const boost::system::error_code& error, std::size_t bytes_transferred, std::shared_ptr<CSession> _self_Session)
{
	if (error)
	{
		std::cout << "Read failed: The code is " << error.value() << " " << "The message is:" << error.message()
			<< "\n";
		_server->CleanSessionMap(_uuid);
	}
	else
	{
		int copy_len = 0;
		while (bytes_transferred > 0)
		{
			if (!_b_head_parse)
			{
				if (bytes_transferred + _recv_hand_node->_cur_len < HEAD_LENGTH)
				{
					memcpy(_recv_hand_node->_data + _recv_hand_node->_cur_len, data + copy_len, bytes_transferred);
					_recv_hand_node->_cur_len += bytes_transferred;
					memset(data, 0, max_length);
					socket.async_read_some(boost::asio::buffer(data, max_length),
						[this, self = _self_Session](const boost::system::error_code& error, std::size_t bytes_transferred)
						{
							handle_read(error, bytes_transferred, self);
						});
					return;
				}

				int remain_len = HEAD_LENGTH - _recv_hand_node->_cur_len;
				memcpy(_recv_hand_node->_data + _recv_hand_node->_cur_len, data + copy_len, remain_len);
				bytes_transferred -= remain_len;
				copy_len += remain_len;
				int data_len = 0;
				memcpy(&data_len, _recv_hand_node->_data, HEAD_LENGTH);
				std::cout << "data_length is: " << data_len << "\n";
				//Send(_recv_hand_node->_data, HEAD_LENGTH);
				if (data_len > max_length)
				{
					std::cout << "Invalid data length is: " << data_len << "\n";
					_server->CleanSessionMap(_uuid);
					return;
				}
				_recv_msg_node = std::make_shared<MsgNode>(data_len);
				//剩余的数据小于一个数据包的长度
				if (bytes_transferred < data_len)
				{
					memcpy(_recv_msg_node->_data + _recv_msg_node->_cur_len, data + copy_len, bytes_transferred);
					_recv_msg_node->_cur_len += bytes_transferred;
					memset(data, 0, max_length);
					socket.async_read_some(boost::asio::buffer(data, max_length),
						[this, self = _self_Session](const boost::system::error_code& error, std::size_t bytes_transferred)
						{
							handle_read(error, bytes_transferred, self);
						});
					_b_head_parse = true;
					return;
				}
				//剩余的数据大于一个数据包的长度
				memcpy(_recv_msg_node->_data + _recv_msg_node->_cur_len, data + copy_len, data_len);
				_recv_msg_node->_cur_len += data_len;
				bytes_transferred -= data_len;
				copy_len += data_len;
				_recv_msg_node->_data[_recv_msg_node->_total_len] = '\0';
				std::cout << "received_data is: " << _recv_msg_node->_data << "The length is " << _recv_msg_node->_total_len << "\n";
				Send(_recv_msg_node->_data, _recv_msg_node->_total_len);
				_b_head_parse = false;
				_recv_hand_node->Clear();

				if (bytes_transferred <= 0)
				{
					memset(data, 0, max_length);
					socket.async_read_some(boost::asio::buffer(data, max_length),
						[this, self = _self_Session](const boost::system::error_code& error, std::size_t bytes_transferred)
						{
							handle_read(error, bytes_transferred, self);
						});
					return;
				}
				continue;
			}
			//处理已经读完头部，并且上次读到的数据小于一个数据包的长度
			int remain_length = _recv_msg_node->_total_len - _recv_msg_node->_cur_len;
			//剩余的数据小于一个数据包的长度
			if (bytes_transferred < remain_length)
			{
				memcpy(_recv_msg_node->_data + _recv_msg_node->_cur_len, data + copy_len, bytes_transferred);
				_recv_msg_node->_cur_len += bytes_transferred;
				memset(data, 0, max_length);
				socket.async_read_some(boost::asio::buffer(data, max_length),
					[this, self = _self_Session](const boost::system::error_code& error, std::size_t bytes_transferred)
					{
						handle_read(error, bytes_transferred, self);
					});
				return;
			}

			//剩余的数据大于一个数据包的长度
			memcpy(_recv_msg_node->_data + _recv_msg_node->_cur_len, data + copy_len, remain_length);
			_recv_msg_node->_cur_len += remain_length;
			bytes_transferred -= remain_length;
			copy_len += remain_length;
			_recv_msg_node->_data[_recv_msg_node->_total_len] = '\0';
			std::cout << "received_data is: " << _recv_msg_node->_data << "The length is " << _recv_msg_node->_total_len << "\n";
			Send(_recv_msg_node->_data, _recv_msg_node->_total_len);
			_b_head_parse = false;
			_recv_hand_node->Clear();
			if (bytes_transferred <= 0)
			{
				memset(data, 0, max_length);
				socket.async_read_some(boost::asio::buffer(data, max_length),
					[this, self = _self_Session](const boost::system::error_code& error, std::size_t bytes_transferred)
					{
						handle_read(error, bytes_transferred, self);
					});
				return;
			}
			continue;
		}

	}
}
```
我们改写Handle_read函数，增添拆包行为。
![流程图](https://cdn.llfc.club/1683373951566.jpg)
### 粘包测试代码
```cpp
//CSession.h
#pragma once
#include<iostream>
#include<boost/asio.hpp>
#include<boost/uuid/uuid_generators.hpp>
#include<boost/uuid/uuid_io.hpp>
#include"CServer.h"
#include<memory>
#include<queue>
#include<mutex>
#include<thread>
#include<iomanip>
#define max_length 1024*2
#define HEAD_LENGTH 4

class CServer;

class MsgNode
{
	friend class CSession;
public:
	MsgNode(char* msg, int max_len)//发送用
		:_total_len{ max_len + HEAD_LENGTH }, _cur_len{ 0 }
	{
		if (max_len <= 0) {
			std::cerr << "max_len must be greater than 0\n";
			return; // 或者抛出异常
		}

		_data = new char[_total_len + 1]();
		if (!_data) {
			std::cerr << "Failed to allocate memory for _data\n";
			return; // 或者抛出异常
		}
		std::cout << "The total_len is: " << _total_len << "\n";
		memcpy(_data, &max_len, HEAD_LENGTH);
		//std::cout << "构造函数内data: " << _data << "\n";
		memcpy(_data + HEAD_LENGTH, msg, max_len);
		_data[_total_len] = '\0';
		//std::cout << "构造函数内最后的data: " << _data << "\n";
	}

	MsgNode(int max_len)//收到用
		:_total_len{max_len}, _cur_len{0}
	{
		_data = new char[_total_len + 1]();
	}

	~MsgNode()
	{
		delete[] _data;
	}

	void Clear()
	{
		std::memset(_data, 0, _total_len);
		_cur_len = 0;
	}
private:
	char* _data;
	int _total_len;
	int _cur_len;
};

class CSession :public std::enable_shared_from_this<CSession>
{
public:
	CSession(boost::asio::io_context& ioc, CServer* server)
		:socket{ ioc }, _server{ server }, _b_head_parse{ false }
	{
		boost::uuids::uuid a_uuid = boost::uuids::random_generator()();
		_uuid = boost::uuids::to_string(a_uuid);
		_recv_hand_node = std::make_shared<MsgNode>(max_length);
	}

	~CSession()
	{
		std::cout << "Session is destroyed" << "\n";
	}

	boost::asio::ip::tcp::socket& Socket()
	{
		return socket;
	}

	std::string& Get_uuid()
	{
		return _uuid;
	}

	void start();
	void Send(char* msg, int max_len);
private:
	char data[max_length];
	CServer* _server;
	boost::asio::ip::tcp::socket socket;
	std::string _uuid;
	std::queue<std::shared_ptr<MsgNode>> _msg_queue;
	std::mutex _send_lock;
	std::shared_ptr<MsgNode> _recv_msg_node;//接收到的信息节点
	std::shared_ptr<MsgNode> _recv_hand_node;//接受到的头部节点
	bool _b_head_parse;//是否处理完头部信息
	void handle_write(const boost::system::error_code& error, std::shared_ptr<CSession> _self_Session);
	void handle_read(const boost::system::error_code& error, std::size_t bytes_transferred, std::shared_ptr<CSession> _self_Session);
	void PrintRecvData(char* data, int length);
};

```


```cpp
//CSession.cpp

void CSession::PrintRecvData(char* data, int length) {
	std::stringstream ss;
	std::string result = "0x";
	for (int i = 0; i < length; i++) {
		std::string hexstr;
		ss << std::hex << std::setw(2) << std::setfill('0') << int(data[i]) << std::endl;
		ss >> hexstr;
		result += hexstr;
	}
	std::cout << "receive raw data is : " << result << std::endl;
}
```

```cpp
//textclinet.cpp
#include <iostream>
#include <boost/asio.hpp>
#include <thread>
using namespace boost::asio::ip;
using namespace std;
const int MAX_LENGTH = 1024 * 2;
const int HEAD_LENGTH = 4;
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
				const char* request = "hello world!";
				size_t request_length = strlen(request);
				char send_data[MAX_LENGTH] = { 0 };
				memcpy(send_data, &request_length, 4);
				memcpy(send_data + 4, request, request_length);
				boost::asio::write(sock, boost::asio::buffer(send_data, request_length + 4));
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
