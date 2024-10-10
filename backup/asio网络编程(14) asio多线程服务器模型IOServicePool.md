本文是学习[**恋恋风辰Zack**](https://space.bilibili.com/271469206)asio网络编程的学习记录
### 多线程模型
对于asio的多线程模型而言，我们通常有两种模型：第一种模型是创建多个io_context，然后被多个线程使用；第二个模型是多个线程共用一个io_context。
我们先将我们之前的单线程模型放出来：
![photo](https://cdn.llfc.club/1685620269385.jpg)
这是我们本节的多线程模型:
![photo](https://cdn.llfc.club/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20230604151126.png)
IOServicePool模型的特点:
* 因为每一个io_context都跑在一个单独的线程，因此对于相同的socket和io_context，他的回调函数和注册函数都是对应于同一个线程的，因此这是线程安全的。
* 对于不同的socket，我们注册给不同的io_context，假设此时我们CPU的核数为8，那我们最多可以注册8个io_context，当第9个socket连接到服务器的时候，其对应的io_context为第一个io_context，当我们出现这种情况的时候，这是线程安全的，因为他们共用同一个事件队列和消息队列。
* 如果不同的Socket注册到不同的io_context的时候，就可能会出现线程安全问题，假设Session1对应的是玩家1，Session2对应的是玩家2，他们都要对工会做出贡献，那此时工会的仓库就是共享资源，那对于玩家1和玩家2，我们都要进行加锁。对于这个问题，我们可以采用两种方式:回调函数加锁或者是逻辑队列，我们之前解决的逻辑队列，就能够很好的解决这个问题。
* 多线程相比单线程，极大的提高了并发的能力，而且能够一定程度的避免：因为单个回调函数响应时间过长而导致后面的回调函数不执行。
### 声明IOServicePool
```cpp
#pragma once
#include<vector>
#include<boost/asio.hpp>
#include"Singleton.h"
class AsioIOServicePool :public Singleton<AsioIOServicePool>
{
	friend Singleton<AsioIOServicePool>;
public:
	using IOService = boost::asio::io_context;
	using Work = boost::asio::io_context::work;
	using WorkPtr = std::unique_ptr<Work>;
	~AsioIOServicePool();
	AsioIOServicePool(const AsioIOServicePool&) = delete;
	AsioIOServicePool& operator=(const AsioIOServicePool&) = delete;
	boost::asio::io_context& GetIOService();
	void Stop();
private:
	AsioIOServicePool(std::size_t size = std::thread::hardware_concurrency());
	std::vector<IOService> _io_services;//管理io_service
	std::vector<WorkPtr> _works;//防止io_service退出
	std::vector<thread> _thread;//管理线程
	std::size_t _next_io_service;//记录下一个io_service
};
```
* _io_services:用来管理io_service
* _workd：主要是防止让io_service退出,因为如果io_service在创建的时候，没有socket对其进行连接的时候，io_service会自动的退出，这个时候我们就可以用Work来防止其推出。
* _thread:我们用来管理线程
* _next_io_service:记录io_service的位置
* boost::asio::io_context& GetIOService()：返回io_context
* void Stop():停止线程池
### 定义IOServicePool
```cpp
AsioIOServicePool::AsioIOServicePool(std::size_t size )
	:_io_services(size), _works(size), _next_io_service(0)
{
	for (std::size_t i{}; i < size; i++)
	{
		_works[i] = std::unique_ptr<Work>(new Work(_io_services[i]));
	}

	for (std::size_t i{}; i < _io_services.size(); i++)
	{
		_thread.emplace_back([this, i]() {
			_io_services[i].run();
			});
	}
}
```
在初始化的时候，我们初始化_io_services、_works和_next_io_service。然后遍历vector，让_works来绑定每个io_context，创建每一个线程。
```cpp
boost::asio::io_context& AsioIOServicePool:: GetIOService()
{
	auto size = _io_services.size();
	auto& ret_io_service = _io_services[_next_io_service++];
	if (_next_io_service == size)
	{
		_next_io_service = 0;
	}
	return ret_io_service;
}
```
通过轮询的方式来返回io_contexte，当然，上面也可以采用取余的方式。
```cpp
sioIOServicePool::~AsioIOServicePool()
{
	std::cout << "AsioIOServicePool destruct" << endl;
}

void AsioIOServicePool::Stop()
{
	for (auto& t : _io_services)
	{
		t.stop();
	}

	for (auto& t : _thread)
	{
		t.join();
	}
}
```
Stop函数，让每个io_context停止，线程池结束的时候，让每个线程join。
### 修改main函数和CServer
```cpp
#include <iostream>
#include "CServer.h"
#include"AsioIOServicePool.h"
using namespace std;
int main()
{
	try {
		auto pool = AsioIOServicePool::GetInstance();
		boost::asio::io_context  io_context;
		boost::asio::signal_set signals(io_context, SIGINT, SIGTERM);
		signals.async_wait(
			[&io_context, pool](auto, auto)
			{
				io_context.stop();
				pool->Stop();
			});
		CServer s(io_context, 10086);
		io_context.run();
	}
	catch (std::exception& e) {
		std::cerr << "Exception: " << e.what() << endl;
	}
	boost::asio::io_context io_context;
}
```
我们先初始化线程池，等到检测到了信号，我们就把线程池关了。
```cpp
void CServer::HandleAccept(shared_ptr<CSession> new_session, const boost::system::error_code& error){
	if (!error) {
		new_session->Start();
		_sessions.insert(make_pair(new_session->GetUuid(), new_session));
	}
	else {
		cout << "session accept failed, error is " << error.what() << endl;
	}

	StartAccept();
}

void CServer::StartAccept() {
	auto& _io_context = AsioIOServicePool::GetInstance()->GetIOService();
	shared_ptr<CSession> new_session = make_shared<CSession>(_io_context, this);
	_acceptor.async_accept(new_session->GetSocket(), std::bind(&CServer::HandleAccept, this, new_session, placeholders::_1));
}
```
我们在CServer::StartAccept()中不断地轮询返回io_context，每返回一个io_contexte，那我们就创建一个新的会话，因为我们的StartAccept和其回调函数是闭环操作，所以我们可以让每个新来的socket都能够绑定上io_context。
### 客户端代码
```cpp
#include <iostream>
#include <boost/asio.hpp>
#include <thread>
#include <json/json.h>
#include <json/value.h>
#include <json/reader.h>
#include<chrono>
using namespace std;
using namespace boost::asio::ip;
const int MAX_LENGTH = 1024 * 2;
const int HEAD_LENGTH = 2;
const int HEAD_TOTAL = 4;

std::vector<thread> vec_thread;
int main()
{
	auto start = std::chrono::high_resolution_clock::now();
	for (int i{}; i < 100; i++)
	{
		vec_thread.emplace_back([]() {
			try
			{
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
				int i{};
				while (i < 500)
				{
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
					i++;

				}
			}
			catch (const std::exception&e)
			{
				std::cerr << "Exception: " << e.what() << endl;
			}
			});
		std::this_thread::sleep_for(std::chrono::milliseconds(10));
	}
	
	for (auto& t : vec_thread)
	{
		t.join();
	}

	auto end = std::chrono::high_resolution_clock::now();

	auto seconds = std::chrono::duration_cast<std::chrono::seconds>(end - start);
	std::cout << "Released time is:" << seconds.count() << " seconds\n";
	getchar();
	return 0;
}
```
我们在此处开辟了100个客户端，每个客户端发送500个包，因此服务器要处理1万个包的数量。
### 运行结果
![photo](https://github.com/FKlightdog/Images/blob/main/PoolRun.png?raw=true)
