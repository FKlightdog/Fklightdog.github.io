## 同步服务器和客户端的搭建
本文是学习[恋恋风辰Zack](https://space.bilibili.com/271469206)asio网络编程的学习记录
### 客户端的搭建
示例代码：
```cpp
#include<boost/asio.hpp>
#include <iostream>
constexpr int MAX_LENGTH = 1024;
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
		boost::asio::write(socket, boost::asio::buffer(request, request_length));

		char reply[MAX_LENGTH];
		std::size_t reply_length = boost::asio::read(socket, boost::asio::buffer(reply, request_length));
		std::cout << "Reply is: ";
		std::cout.write(reply, reply_length);
		std::cout << "\n";
		getchar();

	}
	catch (const std::exception& e)
	{
		std::cerr << "Expection:" << e.what() << "\n";
	}
	return 0;
}
```
对于客户端而言我们先创建上下文io_context，利用上下文来创建socket，然后我们用endpoint来创建终端节点，来connect服务器,最后我们调用boost::asio::read和boost::asio::write函数来进行读写操作
### 服务器的搭建
服务器的操作相比较客户端而言，复杂得多，我们先给出示例代码,然后再进行分析
```cpp
#include<boost/asio.hpp>
#include <iostream>
#include<cstdlib>
#include<set>
#include<memory>
using namespace boost::asio::ip;
using namespace std;
constexpr int max_length = 1024;
typedef shared_ptr<tcp::socket> socket_ptr; //定义一个socket指针类型
std::set<shared_ptr<std::thread>> thread_set;//定义一个线程集合

void session(socket_ptr sock)
{
	try
	{	
		while (1)
		{
			char data[max_length];
			memset(data, '\0', max_length);
			boost::system::error_code error;
			//如果调用read()函数，会直到读取到max_length个字节或者对方关闭连接才会返回，这样会导致阻塞
			size_t length = sock->read_some(boost::asio::buffer(data, max_length), error);
			if (error == boost::asio::error::eof)
			{
				std::cout << "The connection closed by pear" << "\n";
				break;
			}
			else if (error)
			{
				throw boost::system::system_error(error);
			}

			std::cout << "Received form the ip:" << sock->remote_endpoint().address().to_string() << "\n";
			std::cout << "The message is" << data << "\n";
			boost::asio::write(*sock, boost::asio::buffer(data, length));
		}
	}
	catch (const std::exception& e)
	{
		std::cerr << "Exception in thread: " << e.what() << "\n" << std::endl;
	}
}

void server(boost::asio::io_context& ioc, unsigned short port_num)
{
	tcp::acceptor acceptor(ioc, tcp::endpoint(tcp::v4(), port_num));
	while (1)
	{
		socket_ptr socket(new tcp::socket(ioc));
		acceptor.accept(*socket);
		auto t = std::make_shared<std::thread>(session, socket);
		thread_set.insert(t);
	}
}

int main()
{
	try
	{
		boost::asio::io_context ioc;
		unsigned short port_num{ 10086 };
		server(ioc, port_num);

		for (auto& t : thread_set)
		{
			t->join();
		}
	}
	catch (const std::exception& e)
	{
		std::cerr << "Exception: " << e.what() << "\n";
	}
	return 0;
}
```
整个代码可以分成三部分：
* session()函数，对每一个进行连接的客户端创建一个会话
* server()函数，启动服务器，分配线程
* main()函数，调用server函数，最后手动的关闭所有正常进行的线程

#### session()函数部分：
```cpp

void session(socket_ptr sock)
{
	try
	{	
		while (1)
		{
			char data[max_length];
			memset(data, '\0', max_length);
			boost::system::error_code error;
			//如果调用read()函数，会直到读取到max_length个字节或者对方关闭连接才会返回，这样会导致阻塞
			size_t length = sock->read_some(boost::asio::buffer(data, max_length), error);
			if (error == boost::asio::error::eof)
			{
				std::cout << "The connection closed by pear" << "\n";
				break;
			}
			else if (error)
			{
				throw boost::system::system_error(error);
			}

			std::cout << "Received form the ip:" << sock->remote_endpoint().address().to_string() << "\n";
			std::cout << "The message is" << data << "\n";
			boost::asio::write(*sock, boost::asio::buffer(data, length));
		}
	}
	catch (const std::exception& e)
	{
		std::cerr << "Exception in thread: " << e.what() << "\n" << std::endl;
	}
}
```
总体的逻辑是：我们对一个客户端一直进行读写操作，直到发生了异常。
在代码层面我们要注意的是
```cpp
size_t length = sock->read_some(boost::asio::buffer(data, max_length), error);
```
在此处我们最好不要选择调用asio::read()函数，如果调用read()函数，会直到读取到max_length个字节或者对方关闭连接才会返回，这样会导致阻塞。对方有可能只会发送几个字节而已。
#### server()部分
```cpp
void server(boost::asio::io_context& ioc, unsigned short port_num)
{
	tcp::acceptor acceptor(ioc, tcp::endpoint(tcp::v4(), port_num));
	while (1)
	{
		socket_ptr socket(new tcp::socket(ioc));
		acceptor.accept(*socket);
		auto t = std::make_shared<std::thread>(session, socket);
		thread_set.insert(t);
	}
}
```
我们先创建服务器套接字，该套接字只要是ipv4的任何地址都可以进行访问，于是进入while循环。在该循环内只要有一个客户端进行连接，我们都会创建一个新的套接字，然后与其进行连接，对于每个连接的客户端，我们都创建一个新的线程来执行session()函数，最后把新创建的线程指针放入线程集合，来让shared_pointer的计数增加，从而避免因为可行域问题，使得线程消失而引起我们的Session()函数并没有执行完毕。
#### main函数部分：
```c++
int main()
{
	try
	{
		boost::asio::io_context ioc;
		unsigned short port_num{ 10086 };
		server(ioc, port_num);

		for (auto& t : thread_set)
		{
			t->join();
		}
	}
	catch (const std::exception& e)
	{
		std::cerr << "Exception: " << e.what() << "\n";
	}
	return 0;
}
```
我们创建出上下文ioc和端口号，将其传给server()函数，注意到我们在退出服务器的时候，选择了在所有进程都结束后，再进行退出。
这是因为：有可能哪一天服务器的accept崩溃了使得server函数立马退出了，但是你其中还有一些进程正常进行与客户端的交互，这对于用户的体验而言是不友好的，因此我们选择在所有进程都执行完毕后，我们再退出服务器。

### 程序执行结果

![图片](https://github.com/FKlightdog/Images/blob/main/SyncServer.png?raw=true)

从图上可以看出来我们实现了服务器和客户端的交互
