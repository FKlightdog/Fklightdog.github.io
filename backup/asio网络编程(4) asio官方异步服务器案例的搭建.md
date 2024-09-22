本文是学习[**恋恋风辰Zack**](https://space.bilibili.com/271469206)asio网络编程的学习记录
### 代码示例
本文服务器框架风格为**C++03的example**，C++11的example与C++03example相比，只是多了**Lambada表达式和smart_ptr**，大体上的风格还是非常相似的。
### Session类
```cpp
class Session
{
public:
	Session(boost::asio::io_context& ioc)
		:socket{ioc}
	{}
	boost::asio::ip::tcp::socket&  Socket()
	{
		return socket;
	}

	void start();
private:
	enum { max_length = 1024 };
	char data[max_length];
	boost::asio::ip::tcp::socket socket;
	void handle_write(const boost::system::error_code& error);
	void handle_read(const boost::system::error_code& error, std::size_t bytes_transferred);
};
```
Session主要是进行数据的收发，进行每一次与客户端的会话。
其内置的私有变量有：储存收发数据的data数组，数组的容量大小max_length，进行write和read操作的回调函数,以及进行连接的socket。
共有变量有：构造函数、返回socket的Socket函数，启动会话的函数。
下面逐渐来实现这个类：
```cpp
void Session::start()
{
	memset(data, 0, max_length);
	socket.async_read_some(boost::asio::buffer(data, max_length),
		[this](const boost::system::error_code& error, std::size_t bytes_transferred)
		{
			handle_read(error, bytes_transferred);
		});
}
```
在我们调用start函数时候，先把数组进行初始化为0,然后我们进行异步读操作，调用我们的回调函数handle_read()函数。其中建议用Lambda函数来替代bind函数，因为其可读性更强。
```cpp
void Session::handle_read(const boost::system::error_code& error, std::size_t bytes_transferred)
{
	if (error)
	{
		std::cout << "Read failed: The code is" << error.value() << " The message is:" << error.message()
			<< "\n";
		delete this;
	}
	else
	{
		std::cout << "server receive data is " << data << "\n";

		boost::asio::async_write(socket, boost::asio::buffer(data, bytes_transferred),
			[this](const boost::system::error_code& error, std::size_t size) {
				handle_write(error);  // 使用 Lambda 来捕获 this 并调用 handle_write
			});
	}
}
```
对于handle_read函数，我们优先检查错误，如果有错误就打印出value和message。接着，我们接着打印收到的消息，并调用async_write()函数来发送收到的消息给客户端，然后调用handle_write()。
对于此处用Lambda函数来代替bind函数的时候要注意一个坑，可以看看async_write()函数的源码：
```c++

template <typename AsyncWriteStream, typename ConstBufferSequence,
    BOOST_ASIO_COMPLETION_TOKEN_FOR(void (boost::system::error_code,
      std::size_t)) WriteToken>
inline auto async_write(AsyncWriteStream& s,
    const ConstBufferSequence& buffers, WriteToken&& token,
    constraint_t<
      is_const_buffer_sequence<ConstBufferSequence>::value
    >)
  -> decltype(
    async_initiate<WriteToken,
      void (boost::system::error_code, std::size_t)>(
        declval<detail::initiate_async_write<AsyncWriteStream>>(),
        token, buffers, transfer_all()))
```
我们可以看到async_write函数要求的回调函数要有两个参数,error_code和size_t类型的数据，因此Lambda表达式应该还要再填个size_t的参数，但是我们并不把其传入进去。
```cpp

void Session::handle_write(const boost::system::error_code& error)
{
	if (error)
	{
		std::cout << "Write failed, The code is :" << error.value() << " The message is :" << error.message() << "\n";
		delete this;
	}
	else
	{
		memset(data, 0, max_length);
		socket.async_read_some(boost::asio::buffer(data, max_length),
			[this](const boost::system::error_code& error, std::size_t bytes_transferred)
			{
				handle_read(error, bytes_transferred);
			});
	}
}
```
handle_write()函数的逻辑是这样的：在我们发送完后，我们再度进行data数组的初始化，然后再次从客户端读取内容，并再度调用回调函数handle_read()，以此我们达到了一组闭环的操作。

### Server类
```cpp

class Server
{
public:
	Server(boost::asio::io_context& ioc, unsigned short port_num);
private:
	boost::asio::ip::tcp::acceptor acceptor;
	boost::asio::io_context& ioc;
	void start_accept();
	void handle_accept(Session* new_session, const boost::system::error_code& error);
};
```
对于Server类，其私有成员有：服务器的socket，上下文ioc，进行连接的start_accept()以及其回调函数handle_accept()。
共有成员：构造函数Server()。
```cpp
Server::Server(boost::asio::io_context& ioc, unsigned short port_num)
	:ioc{ ioc }, acceptor{ ioc, boost::asio::ip::tcp::endpoint(boost::asio::ip::tcp::v4(), port_num) }
{
	std::cout << "Server is running on port " << port_num << std::endl;
	start_accept();
}
```
Server构造函数，先进行初始化上下文和acceptor，然后调用start_accept()函数来进行与服务器的连接。
```cpp
void Server::start_accept()
{
	Session* new_session = new Session(ioc);
	acceptor.async_accept(new_session->Socket(),
		[this, new_session](const boost::system::error_code& error)
		{
			handle_accept(new_session, error);
		});
}
```
start_accept()：我们先new出来一个会话，然后acceptor调用async_accept()来连接新的socket，然后调用回调函数。
```cpp
void Server::handle_accept(Session* new_session, const boost::system::error_code& error)
{
	if (error)
	{
		delete new_session;
	}
	else
	{
		new_session->start();
	}

	start_accept();
}
```
handle_accept()：如果发生了错误，比如对端关闭，那我们就delete这个会话，否则我们就让会话启动，开始我们的应答式读写操作，最后我们再创建新的会话，等待新的客户端的连接。
我们可以看到在服务器上，我们也完成了闭环式的操作。
### 运行结果
对于客户端，我们依旧采用之前的同步客户端。
![图片](https://github.com/FKlightdog/Images/blob/main/AsyncServer.png?raw=true)
### 案例的隐患
对于这个案例最大的隐患是进行读写操作判断**error**时候的**delete this**，因为我们这个服务器操作是闭环的，所以这个**delete this**是比较安全的。
但是现实中的服务器是**全双工通信**的，我们可以任意时候进行收发。
如果有这么一个场景：对端客户端在发送消息后就把客户端关闭了，现实场景可以这样想：我刚发完消息后，就有电话打进来了，客户端暂时关闭了。
那么此时服务器async_write后调用回调函数handle_write()，发现对端客户端关闭，此时**error == 1**，然后**delete this**。
如果此时我们恰好还有一个读取操作，调用了handle_read，那此时**error == 1**，我们又**delete this**，那此时我们进行了二次析构，这会导致服务器崩溃。
可以把代码改成这样，就可以模拟这个行为：
```cpp

void Session::handle_read(const boost::system::error_code& error, std::size_t bytes_transferred)
{
	if (error)
	{
		std::cout << "Read failed: The code is" << error.value() << " The message is:" << error.message()
			<< "\n";
		delete this;
	}
	else
	{
		std::cout << "server receive data is " << data << "\n";
		memset(data, 0, max_length);
		socket.async_read_some(boost::asio::buffer(data, max_length),
			[this](const boost::system::error_code& error, std::size_t bytes_transferred)
			{
				handle_read(error, bytes_transferred);
			});
		boost::asio::async_write(socket, boost::asio::buffer("Hello World"),
			[this](const boost::system::error_code& error, std::size_t size) {
				handle_write(error);  // 使用 Lambda 来捕获 this 并调用 handle_write
			});
	}
}

void Session::handle_write(const boost::system::error_code& error)
{
	if (error)
	{
		std::cout << "Write failed, The code is :" << error.value() << " The message is :" << error.message() << "\n";
		delete this;
	}
	else
	{

	}
}

```
我们来一下代码处打上断点：
```cpp
socket.async_read_some(boost::asio::buffer(data, max_length),
	[this](const boost::system::error_code& error, std::size_t bytes_transferred)
	{
		handle_read(error, bytes_transferred);
	});
delete this;//handle_read中的
delete this;//handle_write中的
```
下面我们来看看运行的结果：
![图片](https://github.com/FKlightdog/Images/blob/main/Crash_output.png?raw=true)
我们可以看到error出现了两次，也就说明了我们确实是**delete this了两次**
![图片](https://github.com/FKlightdog/Images/blob/main/Crash.png?raw=true)
