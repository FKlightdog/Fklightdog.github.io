本文是学习[恋恋风辰Zack](https://space.bilibili.com/271469206)asio网络编程的学习记录
## socket的监听和连接
### 网络编程

<p align="center">
  <img src="https://cdn.llfc.club/1540562-20190417002428451-62583604.jpg" alt="图片描述">
  <br>
  网络编程流程图
</p>

对于服务器而言，流程如下：
* socket() --创建服务器套接字
* bind() --连接IP和端口号
* listen() --使套接字处于监听模式，使其准备接受来自客户端的连接
* accept() -- 如果传入连接的请求队列不为空，就从队列中取出一个连接请求，创建一个新的套接字，与客户端进行连接

对于客户端而言，流程如下：
* socket() --创建客户端套接字
* connect() -- 与服务器进行连接

两者共有的有：
* write() --进行写入操作
* read() --进行读取操作


### 终端节点的构造
在使用asio库进行网络编程的时候，我们应该优先构造终端节点，因为终端节点定义了网络通信的来源和目标，其包含了目标的ip和端口号。
我们使用如下函数来构造终端节点
```cpp
asio::ip::tcp::endpoint(const asio::ip::address& addr, unsigned short port);
```
对于上述的addr，我们不能直接使用原生的ip地址，我们需要进行转换，转换为asio库对应的address，可以调用以下函数：
```cpp
static address from_string(const std::string& str);
static address from_string(const std::string& str, asio::error_code& ec);
```
有了以上的要素，我们就可以构造出服务器和客户端对应的终端节点
```cpp
int clinet_end_point()
{
	std::string raw_ip_address{ "127.0.0.1" };
	unsigned short port_num{ 3333 };
	boost::system::error_code ec;
	asio::ip::address ip_address{ asio::ip::address::from_string(raw_ip_address, ec) };
	if (ec.value() != 0)
	{
		std::cout << "Failed to parse the IP address. Error code = " << ec.value()
			<< " The message is " << ec.message() << "\n";
		return ec.value();
	}

	asio::ip::tcp::endpoint ep(ip_address, port_num);
	return 0;
}


int server_end_point()
{
	unsigned short port_num{ 3333 };
	asio::ip::address ip_address{ asio::ip::address_v6::any() };
	asio::ip::tcp::endpoint ep(ip_address, port_num);
	return 0;
}
```

### socket的创建
在asio库中**asio::io_context**是一个很重要的概念，其在网络编程中起到了上下文的作用，它用于管理异步 I/O 操作，如网络连接的建立、数据的读写等。
我们使用以下语句来创建socket：
```cpp
asio::ip::tcp::socket sock(ioc);
asio::ip::tcp::acceptor acceptor(ioc, asio::ip::tcp::endpoint(asio::ip::tcp::v4(), unsigned short port_num));
asio::ip::tcp::acceptor acceptor(ioc, asio::ip::tcp::v4());
//asio::ip::tcp::v4()为网络协议Ipv4
```
客户端示例代码：
```cpp
int create_tcp_socket()
{
	asio::io_context ioc;
	asio::ip::tcp protocol{ asio::ip::tcp::v4() };
	asio::ip::tcp::socket sock(ioc);
	boost::system::error_code ec;
	sock.open(protocol, ec);
	if (ec.value() != 0)
	{
		std::cout << "Failed to open the socket! Error code = " << ec.value()
			<< ". Message: " << ec.message() << "\n";
		return ec.value();
	}
	return 0;
}
```
对于服务器示例代码能够更加简单一些：
```cpp

int create_acceptor_socket()
{
	asio::io_context ioc;
	//asio::ip::tcp protocol{ asio::ip::tcp::v4() };
	//asio::ip::tcp::acceptor acceptor(ioc);
	//boost::system::error_code ec;
	//acceptor.open(protocol, ec);
	//if (ec.value() != 0)
	//{
	//	std::cout << "Failed to open the acceptor socket! Error code = " << ec.value()
	//		<< ". Message: " << ec.message() << "\n";
	//	return ec.value();
	//}
	asio::ip::tcp::acceptor(ioc, asio::ip::tcp::endpoint(asio::ip::tcp::v4(), 3333));

	return 0;
}
```
其中注释掉的代码为boost老版本创建socket的模式，新版本可以用以下语句单一代替：
```cpp
asio::ip::tcp::acceptor(ioc, asio::ip::tcp::endpoint(asio::ip::tcp::v4(), 3333));
```
以上语句可以一口气的实现socket(),bind(),listen(),其传入连接的请求队列取决于操作系统，如果在以上流程中发生了错误，其会返回一个 **asio::system_error**
### 服务器bind
示例代码：
```cpp

int bind_acceptor_socket()
{
	unsigned short port_num{ 3333 };
	asio::io_context ioc;
	asio::ip::tcp::endpoint ep(asio::ip::address_v4::any(), port_num);
	asio::ip::tcp::acceptor acceptor(ioc, ep.protocol());
	boost::system::error_code ec;
	acceptor.bind(ep, ec);
	if (ec.value() != 0)
	{
		std::cout << "Failed to open the socket! Error code = " << ec.value()
			<< ". Message: " << ec.message() << "\n";
		return ec.value();
	}
	return 0;
}
```
只需要调用bind函数即可，参数为终端节点endpoint
### 客户端connect
示例代码：
```cpp

int connect_end_point()
{
	std::string raw_ip_address{ "127.0.0.1" };
	unsigned short port_num{ 3333 };
	try
	{
		asio::ip::tcp::endpoint ep(asio::ip::address::from_string(raw_ip_address), port_num);
		asio::io_context ioc;
		asio::ip::tcp::socket sock(ioc, ep.protocol());
		sock.connect(ep);
	}
	catch (system::system_error& e )
	{
		std::cout << "Error occured! Error code = " << e.code()
			<< ". Message: " << e.what() << "\n";
		return e.code().value();
	}
}
```
同样，只需要调用connect()即可
### 服务器accept
示例代码：
```cpp

int accept_new_connection()
{
	const int BACKLOG_SIZE = 30;
	unsigned short port_num{ 3333 };
	asio::io_context ioc;
	asio::ip::tcp::endpoint ep(asio::ip::address_v4::any(), port_num);
	try
	{
		asio::ip::tcp::acceptor acceptor(ioc, ep.protocol());
		acceptor.bind(ep);
		acceptor.listen(BACKLOG_SIZE);
		asio::ip::tcp::socket sock(ioc);
		acceptor.accept(sock);
	}
	catch (system::system_error& e)
	{
		std::cout << "Error occured! Error code = " << e.code()
			<< ". Message: " << e.what() << "\n";
		return e.code().value();
	}
}
```
上述代码为手动进行了bind()和listen(),通过调用listen(),我们可以设置出传入消息队列的长度。正如我们第一部分说的，服务器accept()时候，会创建一个新的套接字来进行连接,因此我们在代码里面也是创建了一个新的套接字，然后调用accpet()函数，把新的套接字传参传入进去.
