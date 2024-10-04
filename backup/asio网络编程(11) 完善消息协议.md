本文是学习[**恋恋风辰Zack**](https://space.bilibili.com/271469206)asio网络编程的学习记录
原本我们的消息发送结构为消息长度+消息内容，但是这并不是完善的。例如在游戏服务器中，玩家进行的不同行为理应应该有属于自己的专门ID，这样子才能方便进行管理，后续我们将完善逻辑层。
### 改写消息节点
我们此处的思路是创建一个基类MsgNode，然后以此扩展出我们的发送节点和接收节点。（在visual studio添加新类即可）
```cpp
//MsgNode.h
#pragma once
#include<iostream>
#include<boost/asio.hpp>
#include<string>
#include"const.h"
class MsgNode
{
public:
	MsgNode(short max_len)
		:_total_len( max_len ), _cur_len( 0 )
	{
		_data = new char[_total_len + 1]();
		_data[_total_len] = '\0';
	}

	~MsgNode()
	{
		std::cout << "MsgNode destructed" << std::endl;
		delete[] _data;
	}

	void Clear()
	{
		std::memset(_data, 0, _total_len);
		_cur_len = 0;
	}
	char* _data;
	short _total_len;
	short _cur_len;
};

class SendNode : public MsgNode
{
public:
	SendNode(const char* msg, short max_len, short msg_id);
private:
	short _msg_id;
};

class RecvNode :public MsgNode
{
public:
	RecvNode(short max_len, short msg_id);
private:
	short _msg_id;
};
```
接着完善发送节点和接受节点
```cpp
//MsgNode.cpp
#include "MsgNode.h"
SendNode::SendNode(const char* msg, short max_len, short msg_id)
	:MsgNode(max_len + HEAD_TOTAL_LEN), _msg_id(msg_id)
{
	short msg_id_host = boost::asio::detail::socket_ops::host_to_network_short(msg_id);
	std::memcpy(_data, &msg_id_host, HEAD_ID_LEN);
	short msg_len_host = boost::asio::detail::socket_ops::host_to_network_short(max_len);
	std::memcpy(_data + HEAD_ID_LEN, &msg_len_host, HEAD_MSG_LEN);
	std::memcpy(_data + HEAD_TOTAL_LEN, msg, max_len);
}

RecvNode::RecvNode(short max_len, short msg_id)
	:MsgNode(max_len), _msg_id(msg_id)
{}
```
### 完善粘包处理
因为我们更新了信息格式，所以我们要完善我们的HandleRead，主要就是粘包的逻辑处理，逻辑处理跟之前的处理是大差不差的，直接放代码。
```cpp
void CSession::HandleRead(const boost::system::error_code& error, size_t  bytes_transferred, std::shared_ptr<CSession> shared_self){
	try {
		if (!error) {
			//已经移动的字符数
			int copy_len = 0;
			while (bytes_transferred > 0) {
				if (!_b_head_parse) {
					//收到的数据不足头部大小
					if (bytes_transferred + _recv_head_node->_cur_len < HEAD_TOTAL_LEN) {
						memcpy(_recv_head_node->_data + _recv_head_node->_cur_len, _data + copy_len, bytes_transferred);
						_recv_head_node->_cur_len += bytes_transferred;
						::memset(_data, 0, MAX_LENGTH);
						_socket.async_read_some(boost::asio::buffer(_data, MAX_LENGTH),
							std::bind(&CSession::HandleRead, this, std::placeholders::_1, std::placeholders::_2, shared_self));
						return;
					}
					//收到的数据比头部多
					//头部剩余未复制的长度
					int head_remain = HEAD_TOTAL_LEN - _recv_head_node->_cur_len;
					memcpy(_recv_head_node->_data + _recv_head_node->_cur_len, _data + copy_len, head_remain);
					//更新已处理的data长度和剩余未处理的长度
					copy_len += head_remain;
					bytes_transferred -= head_remain;
					//获取头部ID
					short head_id = 0;
					memcpy(&head_id, _recv_head_node->_data, HEAD_ID_LEN);
					head_id = boost::asio::detail::socket_ops::network_to_host_short(head_id);
					std::cout << "head_id is " << head_id << endl;
					if (head_id > MAX_LENGTH)
					{
						std::cout << "invalid head id is " << head_id << endl;
						_server->ClearSession(_uuid);
						return;
					}
					//获取头部数据
					short data_len = 0;
					memcpy(&data_len, _recv_head_node->_data + HEAD_ID_LEN, HEAD_MSG_LEN);
					//网络字节序转化为本地字节序
					data_len = boost::asio::detail::socket_ops::network_to_host_short(data_len);
					std::cout << "data_len is " << data_len << endl;
					//头部长度非法
					if (data_len > MAX_LENGTH) {
						std::cout << "invalid data length is " << data_len << endl;
						_server->ClearSession(_uuid);
						return;
					}
					_recv_msg_node = make_shared<RecvNode>(data_len, head_id);

					//消息的长度小于头部规定的长度，说明数据未收全，则先将部分消息放到接收节点里
					if (bytes_transferred < data_len) {
						memcpy(_recv_msg_node->_data + _recv_msg_node->_cur_len, _data + copy_len, bytes_transferred);
						_recv_msg_node->_cur_len += bytes_transferred;
						::memset(_data, 0, MAX_LENGTH);
						_socket.async_read_some(boost::asio::buffer(_data, MAX_LENGTH),
							std::bind(&CSession::HandleRead, this, std::placeholders::_1, std::placeholders::_2, shared_self));
						//头部处理完成
						_b_head_parse = true;
						return;
					}

					memcpy(_recv_msg_node->_data + _recv_msg_node->_cur_len, _data + copy_len, data_len);
					_recv_msg_node->_cur_len += data_len;
					copy_len += data_len;
					bytes_transferred -= data_len;
					_recv_msg_node->_data[_recv_msg_node->_total_len] = '\0';
					//cout << "receive data is " << _recv_msg_node->_data << endl;
					//此处可以调用Send发送测试
					Json::Reader reader;
					Json::Value root;
					reader.parse(std::string(_recv_msg_node->_data, _recv_msg_node->_total_len), root);
					std::cout << "recevie msg id  is " << root["id"].asInt() << " msg data is "
						<< root["data"].asString() << endl;
					root["data"] = "server has received msg, msg data is " + root["data"].asString();
					std::string return_str = root.toStyledString();
					Send(return_str,root["id"].asInt());
					//继续轮询剩余未处理数据
					_b_head_parse = false;
					_recv_head_node->Clear();
					if (bytes_transferred <= 0) {
						::memset(_data, 0, MAX_LENGTH);
						_socket.async_read_some(boost::asio::buffer(_data, MAX_LENGTH),
							std::bind(&CSession::HandleRead, this, std::placeholders::_1, std::placeholders::_2, shared_self));
						return;
					}
					continue;
				}

				//已经处理完头部，处理上次未接受完的消息数据
				//接收的数据仍不足剩余未处理的
				int remain_msg = _recv_msg_node->_total_len - _recv_msg_node->_cur_len;
				if (bytes_transferred < remain_msg) {
					memcpy(_recv_msg_node->_data + _recv_msg_node->_cur_len, _data + copy_len, bytes_transferred);
					_recv_msg_node->_cur_len += bytes_transferred;
					::memset(_data, 0, MAX_LENGTH);
					_socket.async_read_some(boost::asio::buffer(_data, MAX_LENGTH),
						std::bind(&CSession::HandleRead, this, std::placeholders::_1, std::placeholders::_2, shared_self));
					return;
				}
				memcpy(_recv_msg_node->_data + _recv_msg_node->_cur_len, _data + copy_len, remain_msg);
				_recv_msg_node->_cur_len += remain_msg;
				bytes_transferred -= remain_msg;
				copy_len += remain_msg;
				_recv_msg_node->_data[_recv_msg_node->_total_len] = '\0';
				//cout << "receive data is " << _recv_msg_node->_data << endl;
					//此处可以调用Send发送测试
				Json::Reader reader;
				Json::Value root;
				reader.parse(std::string(_recv_msg_node->_data, _recv_msg_node->_total_len), root);
				std::cout << "recevie msg id  is " << root["id"].asInt() << " msg data is "
					<< root["data"].asString() << endl;
				root["data"] = "server has received msg, msg data is " + root["data"].asString();
				std::string return_str = root.toStyledString();
				Send(return_str,root["id"].asInt());
				//继续轮询剩余未处理数据
				_b_head_parse = false;
				_recv_head_node->Clear();
				if (bytes_transferred <= 0) {
					::memset(_data, 0, MAX_LENGTH);
					_socket.async_read_some(boost::asio::buffer(_data, MAX_LENGTH),
						std::bind(&CSession::HandleRead, this, std::placeholders::_1, std::placeholders::_2, shared_self));
					return;
				}
				continue;
			}
		}
		else {
			std::cout << "handle read failed, error is " << error.what() << endl;
			Close();
			_server->ClearSession(_uuid);
		}
	}
	catch (std::exception& e) {
		std::cout << "Exception code is " << e.what() << endl;
	}
}

```
对于上面的代码，因为我们添加了消息ID，所以我们也更改了Send函数，添加了消息msgid，因此我们要修改Send函数。
```cpp

void CSession::Send(std::string msg, short msg_id)
{
	std::lock_guard<std::mutex> lock(_send_lock);
	int send_que_size = _send_que.size();
	if (send_que_size > MAX_SENDQUE) {
		std::cout << "session: " << _uuid << " send que fulled, size is " << MAX_SENDQUE << endl;
		return;
	}
	//std::cout << "The msg is" << msg << endl;
	_send_que.push(make_shared<SendNode>(msg.c_str(), msg.length(), msg_id));
	if (send_que_size > 0) {
		return;
	}
	auto& msgnode = _send_que.front();
	//std::cout << "send data is " << msgnode->_data << endl;
	boost::asio::async_write(_socket, boost::asio::buffer(msgnode->_data, msgnode->_total_len),
		std::bind(&CSession::HandleWrite, this, std::placeholders::_1, SharedSelf()));
}

void CSession::Send(char* msg, short max_length, short msg_id) {
	std::lock_guard<std::mutex> lock(_send_lock);
	int send_que_size = _send_que.size();
	if (send_que_size > MAX_SENDQUE) {
		std::cout << "session: " << _uuid << " send que fulled, size is " << MAX_SENDQUE << endl;
		return;
	}

	_send_que.push(make_shared<SendNode>(msg, max_length, msg_id));
	if (send_que_size>0) {
		return;
	}
	auto& msgnode = _send_que.front();
	boost::asio::async_write(_socket, boost::asio::buffer(msgnode->_data, msgnode->_total_len), 
		std::bind(&CSession::HandleWrite, this, std::placeholders::_1, SharedSelf()));
}
```
### 修改客户端代码
```cpp
#include <iostream>
#include <boost/asio.hpp>
#include <thread>
#include <json/json.h>
#include <json/value.h>
#include <json/reader.h>
using namespace std;
using namespace boost::asio::ip;
const int MAX_LENGTH = 1024 * 2;
const int HEAD_LENGTH = 2;
const int HEAD_TOTAL = 4;
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

		Json::Value root;
		root["id"] = 1001;
		root["data"] = "hello world";
		std::string request = root.toStyledString();
		size_t request_length = request.length();
		char send_data[MAX_LENGTH] = { 0 };
		int msgid = 1001;
		int msgid_host = boost::asio::detail::socket_ops::host_to_network_short(msgid);
		memcpy(send_data, &msgid_host, 2);
		//转为网络字节序
		int request_host_length = boost::asio::detail::socket_ops::host_to_network_short(request_length);
		memcpy(send_data + 2, &request_host_length, 2);
		memcpy(send_data + 4, request.c_str(), request_length);
		boost::asio::write(sock, boost::asio::buffer(send_data, request_length + 4));
		cout << "begin to receive..." << endl;

		char reply_head[HEAD_TOTAL];
		size_t reply_length = boost::asio::read(sock, boost::asio::buffer(reply_head, HEAD_TOTAL));

		msgid = 0;
		memcpy(&msgid, reply_head, HEAD_LENGTH);
		short msglen = 0;
		memcpy(&msglen, reply_head + 2, HEAD_LENGTH);
		//转为本地字节序
		msglen = boost::asio::detail::socket_ops::network_to_host_short(msglen);
		msgid = boost::asio::detail::socket_ops::network_to_host_short(msgid);
		char msg[MAX_LENGTH] = { 0 };
		size_t  msg_length = boost::asio::read(sock, boost::asio::buffer(msg, msglen));
		Json::Reader reader;
		reader.parse(std::string(msg, msg_length), root);
		std::cout << "msg id is " << root["id"] << " msg is " << root["data"] << endl;
		getchar();
	}
	catch (std::exception& e) {
		std::cerr << "Exception: " << e.what() << endl;
	}
	return 0;
}



```
