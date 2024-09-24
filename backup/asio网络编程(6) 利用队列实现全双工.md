本文是学习[**恋恋风辰Zack**](https://space.bilibili.com/271469206)asio网络编程的学习记录
之前的博客文章的异步服务器API，有提到利用队列写出来的API，所以此刻我们用队列来完善我们的服务器，实现全双工。
可以看[Zack的博客](https://llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF#!aid/2OmTLf4Ahbksklnru69zfSz6rg2)，讲的很详细，我此刻就把代码进行一下分享。
### 添加信息节点 MsgNode
```cpp
class MsgNode
{
	friend class CSession;
public:
	MsgNode(char* msg, int max_len)
	{
		_data = new char[max_len];
		memcpy(_data, msg, max_len);
	}
	~MsgNode()
	{
		delete[] _data;
	}
private:
	char* _data;
	int _max_len;
	int _cur_len;
};
```
### 增添Session类
```cpp

class CSession :public std::enable_shared_from_this<CSession>
{
public:
	CSession(boost::asio::io_context& ioc, CServer* server)
		:socket{ ioc }, _server{ server }
	{
		boost::uuids::uuid a_uuid = boost::uuids::random_generator()();
		_uuid = boost::uuids::to_string(a_uuid);
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
	enum { max_length = 1024 };
	char data[max_length];
	CServer* _server;
	boost::asio::ip::tcp::socket socket;
	std::string _uuid;
	std::queue<std::shared_ptr<MsgNode>> _msg_queue;
	std::mutex _send_lock;
	void handle_write(const boost::system::error_code& error, std::shared_ptr<CSession> _self_Session);
	void handle_read(const boost::system::error_code& error, std::size_t bytes_transferred, std::shared_ptr<CSession> _self_Session);
};
```
增加了Send函数、锁和队列。
### 定义并修改函数
```cpp

void  CSession::Send( char* msg, int max_len)
{
	bool pending = false;
	std::lock_guard<std::mutex> lock(_send_lock);
	if (_msg_queue.size() > 0)
	{
		pending = true;
	}
	_msg_queue.push(std::make_shared<MsgNode>(msg, max_len));
	if (pending)
	{
		return;
	}
	boost::asio::async_write(socket, boost::asio::buffer(msg, max_len),
		[this, self = shared_from_this()](const boost::system::error_code& error, std::size_t size)
		{
			handle_write(error, self);
		});
}

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
		std::cout << "server receive data is " << data << "\n";
		Send(data, bytes_transferred);
		memset(data, 0, max_length);
		socket.async_read_some(boost::asio::buffer(data, max_length),
			[this, self = _self_Session](const boost::system::error_code& error, std::size_t bytes_transferred)
			{
				handle_read(error, bytes_transferred, self);
			});
	}
}


void CSession::handle_write(const boost::system::error_code& error, std::shared_ptr<CSession> _self_Session)
{
	if (error)
	{
		std::cout << "Write failed, The code is : " << error.value() << " " << " The message is :" << error.message() << "\n";
		_server->CleanSessionMap(_uuid);
	}
	else
	{
		std::lock_guard<std::mutex> lock(_send_lock);
		_msg_queue.pop();
		if (_msg_queue.size() > 0)
		{
			auto& msg = _msg_queue.front();
			boost::asio::async_write(socket, boost::asio::buffer(msg->_data, msg->_max_len),
				[this, self = _self_Session](const boost::system::error_code& error, std::size_t size)
				{
					handle_write(error, self);
				});
		}
	}
}
```
