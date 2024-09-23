本文是学习[**恋恋风辰Zack**](https://space.bilibili.com/271469206)asio网络编程的学习记录
在上一节，我们可以看到双重析构的风险，因此我们在此节来改善这个问题，我们此处采用的方式是智能指针来延长我们Session的声明周期
下面我们来逐步修改,优先修改Server类
### Server类
我们先从start_server函数开始
```cpp
Session(boost::asio::io_context& ioc, Server* server)//此处的Session构造函数改了
void Server::start_accept()
{
	std::shared_ptr<Session> new_session = std::make_shared<Session>(ioc, this);
	acceptor.async_accept(new_session->Socket(),
		[this, new_session](const boost::system::error_code& error)
		{
			handle_accept(new_session, error);
		});
}
```
我们此处要用智能指针来代替最开始new出来的Session，顺带我们把回调函数也进行修改了，因为回调函数原本的new_session也是new出来的。
```cpp
void Server::handle_accept(std::shared_ptr<Session> new_session, const boost::system::error_code& error)
{
	if (error)
	{
		std::cout << "session accept failed, error is " << error.what() << "\n";
	}
	else
	{
		new_session->start();
		_SessionMap.insert(std::make_pair(new_session->Get_uuid(), new_session));
	}

	start_accept();
}
```
此处我们除了用智能指针之外，我们还创建了一个_SessionMap，来统计对于每一个连接过来的客户端和对应的Session，后续会加上踢人操作。
综上，我们此处的Server类变成了这个样子：
```cpp

class Server
{
public:
	Server(boost::asio::io_context& ioc, unsigned short port_num);
	void CleanSessionMap(std::string);
private:
	boost::asio::ip::tcp::acceptor acceptor;
	boost::asio::io_context& ioc;
	void start_accept();
	void handle_accept(std::shared_ptr<Session> new_session, const boost::system::error_code& error);
	std::map<std::string, std::shared_ptr<Session>> _SessionMap;
};
```
### Session类
接着顺着我们上面的逻辑，因为我们生成了一个map，那么我们需要每一个用户唯一的uid，boost库已经提供了我们的一个函数，这个函数的原理是雪花算法，我们在Session的初始化中，为其添加上去。
```cpp
#include<boost/uuid/uuid_generators.hpp>
#include<boost/uuid/uuid_io.hpp>
//上面为使用random_generator()()的头文件
	Session(boost::asio::io_context& ioc, Server* server)
		:socket{ioc},_server{server}
	{
		boost::uuids::uuid a_uuid = boost::uuids::random_generator()();
		_uuid = boost::uuids::to_string(a_uuid);
	}
```
顺着这个逻辑，我们此处已经要启动新的会话了，此时我们要调用new_session->start();
对于Session类，因为我们要用智能指针来延长生命的周期，因此我们要在Session的回调函数里面加入我们的智能指针，这样让我们的计数+1.
```cpp
void handle_write(const boost::system::error_code& error,std::shared_ptr<Session> _self_Session);
void handle_read(const boost::system::error_code& error, std::size_t bytes_transferred, std::shared_ptr<Session> _self_Session);

void Session::handle_read(const boost::system::error_code& error, std::size_t bytes_transferred, std::shared_ptr<Session> _self_Session)
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

		boost::asio::async_write(socket, boost::asio::buffer(data, bytes_transferred),
			[this, _self_Session](const boost::system::error_code& error, std::size_t size) {
				handle_write(error, _self_Session);  // 使用 Lambda 来捕获 this 并调用 handle_write
			});
	}
}


void Session::handle_write(const boost::system::error_code& error, std::shared_ptr<Session> _self_Session)
{
	if (error)
	{
		std::cout << "Write failed, The code is : " << error.value() << " " << " The message is :" << error.message() << "\n";
		_server->CleanSessionMap(_uuid);
	}
	else
	{
		memset(data, 0, max_length);
		socket.async_read_some(boost::asio::buffer(data, max_length),
			[this, _self_Session](const boost::system::error_code& error, std::size_t bytes_transferred)
			{
				handle_read(error, bytes_transferred, _self_Session);
			});
	}
}

```
> [!NOTE]
> 因为回调函数有自己的函数签名，所以Lambda函数必须要满足其函数签名才可以，不能多加或者减员。

每当出现错误的时候，例如：对端关闭、或者有人顶号，我们就把这个uuid在map中清除，清除的函数如下：
```cpp
void Server::CleanSessionMap(std::string uuid)
{
	_SessionMap.erase(uuid);
}
```
现在我们就只剩下Session::start()函数了，因为handle_read()函数有智能指针的参数，因此我们要传入一个智能指针，此时我们有两个方案：
```cpp

shared_ptr<Session> this
shared_from_this()
```
第一个相当于是裸指针，用这个裸指针当做智能指针的话，可能会导致计数不统一，如果this被其他的智能指针引用，就会导致计数不同步，因此或许会让Session提前被析构了。
第二个函数会返回一个智能指针，该智能指针和其他管理这块内存的智能指针共享引用计数。
如果我们要调用这个函数的话，那我们需要让Session类继承另外一个类。
```cpp
class Session :public std::enable_shared_from_this<Session>
```
综上，我们的代码就修改完成了，让我们把总体代码放出来
### 修改后的代码
```cpp
//Session.h
#pragma once
#include<iostream>
#include<boost/asio.hpp>
#include<map>
#include<boost/uuid/uuid_generators.hpp>
#include<boost/uuid/uuid_io.hpp>
class Server;
class Session :public std::enable_shared_from_this<Session>
{
public:
	Session(boost::asio::io_context& ioc, Server* server)
		:socket{ioc},_server{server}
	{
		boost::uuids::uuid a_uuid = boost::uuids::random_generator()();
		_uuid = boost::uuids::to_string(a_uuid);
	}

	~Session()
	{
		std::cout << "Session is destroyed" << "\n";
	}

	boost::asio::ip::tcp::socket&  Socket()
	{
		return socket;
	}

	std::string& Get_uuid()
	{
		return _uuid;
	}

	void start();
private:
	enum { max_length = 1024 };
	char data[max_length];
	Server* _server;
	boost::asio::ip::tcp::socket socket;
	std::string _uuid;
	void handle_write(const boost::system::error_code& error,std::shared_ptr<Session> _self_Session);
	void handle_read(const boost::system::error_code& error, std::size_t bytes_transferred, std::shared_ptr<Session> _self_Session);
};


class Server
{
public:
	Server(boost::asio::io_context& ioc, unsigned short port_num);
	void CleanSessionMap(std::string);
private:
	boost::asio::ip::tcp::acceptor acceptor;
	boost::asio::io_context& ioc;
	void start_accept();
	void handle_accept(std::shared_ptr<Session> new_session, const boost::system::error_code& error);
	std::map<std::string, std::shared_ptr<Session>> _SessionMap;
};
```
```cpp
//Session.cpp
#include "Session.h"

void Session::start()
{
	memset(data, 0, max_length);
	socket.async_read_some(boost::asio::buffer(data, max_length),
		[this, self = shared_from_this()](const boost::system::error_code& error, std::size_t bytes_transferred)
		{
			handle_read(error, bytes_transferred, self);
		});
}

void Session::handle_read(const boost::system::error_code& error, std::size_t bytes_transferred, std::shared_ptr<Session> _self_Session)
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

		boost::asio::async_write(socket, boost::asio::buffer(data, bytes_transferred),
			[this, _self_Session](const boost::system::error_code& error, std::size_t size) {
				handle_write(error, _self_Session);  // 使用 Lambda 来捕获 this 并调用 handle_write
			});
	}
}


void Session::handle_write(const boost::system::error_code& error, std::shared_ptr<Session> _self_Session)
{
	if (error)
	{
		std::cout << "Write failed, The code is : " << error.value() << " " << " The message is :" << error.message() << "\n";
		_server->CleanSessionMap(_uuid);
	}
	else
	{
		memset(data, 0, max_length);
		socket.async_read_some(boost::asio::buffer(data, max_length),
			[this, _self_Session](const boost::system::error_code& error, std::size_t bytes_transferred)
			{
				handle_read(error, bytes_transferred, _self_Session);
			});
	}
}

Server::Server(boost::asio::io_context& ioc, unsigned short port_num)
	:ioc{ ioc }, acceptor{ ioc, boost::asio::ip::tcp::endpoint(boost::asio::ip::tcp::v4(), port_num) }
{
	std::cout << "Server is running on port " << port_num << std::endl;
	start_accept();
}

void Server::start_accept()
{
	std::shared_ptr<Session> new_session = std::make_shared<Session>(ioc, this);
	acceptor.async_accept(new_session->Socket(),
		[this, new_session](const boost::system::error_code& error)
		{
			handle_accept(new_session, error);
		});
}

void Server::handle_accept(std::shared_ptr<Session> new_session, const boost::system::error_code& error)
{
	if (error)
	{
		std::cout << "session accept failed, error is " << error.what() << "\n";
	}
	else
	{
		new_session->start();
		_SessionMap.insert(std::make_pair(new_session->Get_uuid(), new_session));
	}

	start_accept();
}

void Server::CleanSessionMap(std::string uuid)
{
	_SessionMap.erase(uuid);
}
```
### 运行结果
接下来我们看看代码运行结果
![图片](https://github.com/FKlightdog/Images/blob/main/edited_run.png?raw=true)
我们可以看到服务器是正常运行的。
### 风险测试
上一篇文章，我们看到了两次**delete this**造成的风险，因此我们对修改后的代码进行测试，代码只需要这样改即可,同样打上跟上次类似的断点
```cpp

void Session::handle_read(const boost::system::error_code& error, std::size_t bytes_transferred, std::shared_ptr<Session> _self_Session)
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

		memset(data, 0, max_length);
		socket.async_read_some(boost::asio::buffer(data, max_length),
			[this, self = shared_from_this()](const boost::system::error_code& error, std::size_t bytes_transferred)
			{
				handle_read(error, bytes_transferred, self);
			});
		boost::asio::async_write(socket, boost::asio::buffer("Hello Client"),
			[this, _self_Session](const boost::system::error_code& error, std::size_t size) {
				handle_write(error, _self_Session);  // 使用 Lambda 来捕获 this 并调用 handle_write
			});
	}
}
```
![图片](https://github.com/FKlightdog/Images/blob/main/Rightt.png?raw=true)
我们可以看到程序只析构了一次，安全的退出了。
我们可以这样子理解，我们的两次回调函数加上_SessionMap，使得我们的智能指针计数保底为3，当read的回调出错时候，计数-1，移除map，计数-1，当write回调函数结束时候，计数彻底为0，此时我们的Session才被析构，这样我们就用smart_ptr起到了伪闭包的效果，延长了我们的生命周期。