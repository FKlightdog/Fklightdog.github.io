本文是学习[**恋恋风辰Zack**](https://space.bilibili.com/271469206)asio网络编程的学习记录
本文建议可以直接阅读[**Zack的博客**](https://llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF#!aid/2O0QikntG7ktdgARRndYzNg8R55)，说的很详细。
一下仅记录一下我仿照写的源代码：
### Session.h
```cpp
#pragma once
#include<boost/asio.hpp>
#include<iostream>
#include<memory>
#include<queue>
using namespace boost;
using namespace std;
const int RECVSIZE{ 1024 };
//管理发送和接收的数据
class MsgNode
{
public:
	MsgNode(const char* msg, int total_len)
		:_total_len{ total_len }, _cur_len{ 0 }
	{
		_msg = new char[total_len];
		memcpy(_msg, msg, total_len);
	}

	MsgNode(int total_len)
		:_total_len{ total_len }, _cur_len{ 0 }
	{
		_msg = new char[total_len];
	}

	~MsgNode()
	{
		delete[] _msg;
	}
	//消息首地址
	char* _msg;
	//总长度
	int _total_len;
	//当前长度
	int _cur_len;
};

class Session
{
public:
	void ReadFromSocket();
	void ReadCallBack(const boost::system::error_code& ec, std::size_t bytes_transferred);
	void WriteAllToSocket(const std::string& buf);
	void WriteAllCallBack(const boost::system::error_code& ec, std::size_t bytes_transferred);
	void WriteCallBack(const boost::system::error_code& ec, std::size_t bytes_transferred);
	void WriteToSocket(const std::string& buf);
	void WriteCallBackErr(const boost::system::error_code& ec, std::size_t bytes_transferred
	, std::shared_ptr<MsgNode> msg_node);
	void WriteToSocketErr(const std::string& buf);
	Session(std::shared_ptr<asio::ip::tcp::socket> socket);
	void connect(const asio::ip::tcp::endpoint& ep);
private:
	std::shared_ptr<asio::ip::tcp::socket> _socket;
	std::shared_ptr<MsgNode> _send_node;
	std::shared_ptr<MsgNode> _recv_node;
	std::queue<std::shared_ptr<MsgNode>> _send_queue;
	bool _send_pending;
	bool _recev_pending;
};


```

### Session.cpp
```cpp
#include "Session.h"
//Session的初始化
Session::Session(std::shared_ptr<asio::ip::tcp::socket> socket)
	:_socket(socket), _send_pending(false), _recev_pending(false)
{
}
//服务器当客户端进行连接时，调用connect函数
void Session::connect(const asio::ip::tcp::endpoint& ep)
{
	_socket->connect(ep);
}

void Session::WriteCallBackErr(const boost::system::error_code& ec, std::size_t bytes_transferred, std::shared_ptr<MsgNode> msg_node)
{
	if (bytes_transferred + msg_node->_cur_len < msg_node->_total_len)
	{
		_send_node->_cur_len += bytes_transferred;
		this->_socket->async_write_some(asio::buffer(_send_node->_msg + _send_node->_cur_len, _send_node->_total_len - _send_node->_cur_len),
			[this, msg_node = _send_node](const boost::system::error_code& ec, std::size_t bytes_transferred)
			{
				this->WriteCallBackErr(ec, bytes_transferred, msg_node);
			});
	}
}

void Session::WriteToSocketErr(const std::string& buf)
{
	_send_node = make_shared<MsgNode>(buf.c_str(), buf.length());
	this->_socket->async_write_some(asio::buffer(_send_node->_msg, _send_node->_total_len),
		[this, msg_node = _send_node](const boost::system::error_code& ec, std::size_t bytes_transferred)
		{
			this->WriteCallBackErr(ec, bytes_transferred, msg_node);
		});
}

void Session::WriteCallBack(const boost::system::error_code& ec, std::size_t bytes_transferred)
{
	if (ec.value() != 0)
	{
		std::cout << "Error , code is " << ec.value() << " . Message is " << ec.message();
		return;
	}

	auto& send_data = _send_queue.front();
	send_data->_cur_len += bytes_transferred;
	if (send_data->_cur_len < send_data->_total_len)
	{
		this->_socket->async_write_some(asio::buffer(send_data->_msg + send_data->_cur_len, send_data->_total_len - send_data->_cur_len),
			[this](const boost::system::error_code& ec, std::size_t bytes_transferred)
			{
				this->WriteCallBack(ec, bytes_transferred);
			});
	}

	_send_queue.pop();

	if (_send_queue.empty())
	{
		_send_pending = false;
	}

	if (!_send_queue.empty())
	{
		auto& send_data = _send_queue.front();
		this->_socket->async_write_some(asio::buffer(send_data->_msg, send_data->_total_len),
			[this](const boost::system::error_code& ec, std::size_t bytes_transferred)
			{
				this->WriteCallBack(ec, bytes_transferred);
			});
	}
}

void Session::WriteToSocket(const std::string& buf)
{
	_send_queue.emplace(new MsgNode(buf.c_str(), buf.length()));
	if (_send_pending)
	{
		return;
	}
	this->_socket->async_write_some(asio::buffer(buf),
		[this](const boost::system::error_code& ec, std::size_t bytes_transferred)
		{
			this->WriteCallBack(ec, bytes_transferred);
		});
	_send_pending = true;
}

void Session::WriteAllToSocket(const std::string& buf)
{
	_send_queue.emplace(new MsgNode(buf.c_str(), buf.length()));
	if (_send_pending)
	{
		return;
	}
	this->_socket->async_send(asio::buffer(buf),
		[this](const boost::system::error_code& ec, std::size_t bytes_transferred)
		{
			this->WriteAllCallBack(ec, bytes_transferred);
		});
	_send_pending = true;
}

void Session::WriteAllCallBack(const boost::system::error_code& ec, std::size_t bytes_transferred)
{
	if (ec.value() != 0)
	{
		std::cout << "Error , code is " << ec.value() << " . Message is " << ec.message();
		return;
	}

	_send_queue.pop();
	if (_send_queue.empty())
	{
		_send_pending = false;
	}

	if (!_send_queue.empty())
	{
		auto& send_data = _send_queue.front();
		this->_socket->async_send(asio::buffer(send_data->_msg, send_data->_total_len),
			[this](const boost::system::error_code& ec, std::size_t bytes_transferred)
			{
				this->WriteAllCallBack(ec, bytes_transferred);
			});
	}
}

void Session::ReadFromSocket()
{
	if (_recev_pending)
	{
		return;
	}
	_recv_node = std::make_shared<MsgNode>(RECVSIZE);
	this->_socket->async_read_some(asio::buffer(_recv_node->_msg, _recv_node->_total_len),
		[this](const boost::system::error_code& ec, std::size_t bytes_transferred)
		{
			this->ReadCallBack(ec, bytes_transferred);
		});
	_recev_pending = true;
}

void Session::ReadCallBack(const boost::system::error_code& ec, std::size_t bytes_transferred)
{
	_recv_node->_cur_len += bytes_transferred;
	if (_recv_node->_cur_len < _recv_node->_total_len)
	{
		this->_socket->async_read_some(asio::buffer(_recv_node->_msg + _recv_node->_cur_len, _recv_node->_total_len - _recv_node->_cur_len),
			[this](const boost::system::error_code& ec, std::size_t bytes_transferred)
			{
				this->ReadCallBack(ec, bytes_transferred);
			});
	}

	_recev_pending = false;
	_recv_node = nullptr;
}
```
上述代码最大的不同是bind函数，因为自C++14后由于Lambda表达式的扩展，我们可以用Lambda表达式来写bind函数，而且可读性比bind函数更好。