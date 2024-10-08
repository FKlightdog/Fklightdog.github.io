本文是学习[**恋恋风辰Zack**](https://space.bilibili.com/271469206)asio网络编程的学习记录
### 逻辑层
我们之前的通信流程是这样的：
![图片](https://cdn.llfc.club/1685620269385.jpg)
现在我们要完善我们的框架，让我们的框架变成如下这个样子：
![图片](https://cdn.llfc.club/1685621283099.jpg)
我们在此基础上添加了逻辑层，当我们进行读的回调函数的时候把数据和Session投入到一个逻辑队列里面，然后Logic系统从逻辑队列里面读取数据，读取到数据后再调用Send函数进行发送，发送的时候调用async_send()把事件和回调函数注册到asio网络层里面。
对于逻辑层，我们采用的是单线程模式，并不会采用多线程。
假设如果我们队玩家给工会贡献物品这一行为采用了多线程，那每一个玩家给工会贡献物品的时候，我们都要对线程进行加锁和解锁，这样重复的加锁，我们还不如采用单线程的模式。
### 构造单例模版类
因为我们的服务器逻辑层是单线程模式，所以我们需要我们的逻辑层在多个线程的情况下也是唯一的。
```cpp
//Singleton.h
#pragma once
#include<iostream>
#include<memory>
#include<mutex>
using namespace std;
template<typename T>
class Singleton
{
protected:
	Singleton() = default;
	Singleton(const Singleton<T>&) = delete;
	Singleton& operator=(const Singleton<T>& st) = delete;
	static shared_ptr<T> _instance;
public:
	static shared_ptr<T> GetInstance()
	{
		static std::once_flag flag;
		std::call_once(flag, [&]() {
			_instance = shared_ptr<T>(new T);
			});

		return _instance;
	}
	void PrintAddress()
	{
		cout << _instance.get() << endl;
	}
	~Singleton()
	{
		std::cout << "Singleton Destructor\n"  ;
	}
};

template<typename T>
shared_ptr<T> Singleton<T>::_instance = nullptr;//单例模式的静态成员变量初始化
```
在这个类中我们定义了一个static的智能指针来表示单例。
随后在Public中我们有一个静态函数来返回我们的单例。在这个函数中我们先创建了静态的变量flag，其类型是once_flag，这保证了我们的flag在多个线程内只会被创建一次。
call_once类型的代码只会被运行一次,其取决于flag,就会执行后面的匿名函数,然后创建我们的单例。最后进行return
>[!NOTE]
>首先因为我们这个类是单例类所以我们要禁止构造函数、拷贝构造函数及其拷贝运算>符，并且为了其不被其他人调用，我们应该要将其设为prviate。但是我们的逻辑层要继承这个单利类，对于继承而言：构造函数是先构造父类然后再构造子类，而析构函数是先析构子类，再析构父类。因此我们应该将其设为protected。自然，因为上面的原则，因此我们要将析构函数放到public处。
### 实现我们的逻辑类
```cpp
//LogicSystem.h
#pragma once
#include "Singleton.h"
#include <queue>
#include <thread>
#include "CSession.h"
#include <queue>
#include <map>
#include <functional>
#include "const.h"
#include <json/json.h>
#include <json/value.h>
#include <json/reader.h>
class CSession;
class LogicNode;
typedef function<void(std::shared_ptr<CSession>, short msg_id, std::string msg_data)> FunCallBack;
class LogicSystem :public Singleton<LogicSystem>
{
	friend class Singleton<LogicSystem>;
public:
	~LogicSystem();
	void PostMsgQue(std::shared_ptr<LogicNode> logic_node);//发送到消息队列
private:
	LogicSystem();
	void DealMsg();
	void RegisterCallBacks();
	void HelloWordCallBacks(std::shared_ptr<CSession> session, short msg_id, std::string msg_data);
	std::thread _worker_thread;//工作线程
	std::queue<std::shared_ptr<LogicNode> > _msg_que;//逻辑队列
	std::mutex _mutex;//锁
	std::condition_variable _consume;//条件变量
	bool _b_stop;//从外部接受到的停止信号
	std::map<short, FunCallBack> _fun_callbacks;//id对应的回调函数
};
```
我们将回调函数的function类型用typedef重定义成了FunCallBack。
在public区域里，我们设置析构函数为公有的；同时声明了一个函数PostMsgQue，其主要执行的是将信息发送到消息队列的操作。
在private区域里：
* LogicSystem(): 构造逻辑层。
* DealMsg(): 处理队列中的信息
* RegisterCallBacks(): 注册回调函数
* HelloWordCallBacks()：对于HelloWorld类型的ID，我们进行处理
* __worker_thread: 工作线程
* _msg_que: 逻辑队列
* _mutex: 锁
* _consume: 条件变量
* _b_stop: 从外部接受到的停止信号
* _fun_callbacks: 用来记录id对应的回调函数
```cpp
//LogicSystem.cpp
LogicSystem::LogicSystem()
	:_b_stop{false}
{
	RegisterCallBacks();//注册回调函数
	_worker_thread = std::thread(&LogicSystem::DealMsg, this);//启动工作线程
}
```
在构造函数内，我们初始化的时候把_b_stop初始化为false，同时调用RegisterCallBacks()和工作线程，工作线程内启动DealMsg函数。
```cpp
void LogicSystem::RegisterCallBacks()
{
	_fun_callbacks[MSG_HELLO_WORD] = [this](std::shared_ptr<CSession> session, short msg_id, std::string msg_data)
		{
			HelloWordCallBacks(session, msg_id, msg_data);
		};
}
```
因为我们客户端和服务器的收发只涉及到HELLO_WORD类型的ID，所以我们在此注册一次。
```cpp

void LogicSystem::HelloWordCallBacks(std::shared_ptr<CSession> session, short msg_id, std::string msg_data)
{	//解析json数据
	Json::Reader read;
	Json::Value root;
	read.parse(msg_data, root);
	std::cout << "Msg id is: " << root["id"].asInt() << " " << "Msg data is: " 
		<< root["data"].asString() << "\n";
	root["data"] = "server has received msg, msg is" + root["data"].asString();
	std::string return_str = root.toStyledString();
	session->Send(return_str, root["id"].asInt());//返回数据
}
```
进行Json数据的收发。
```cpp
void LogicSystem::DealMsg()//处理消息
{
	for (;;)
	{
		std::unique_lock<std::mutex> unique_lk(_mutex);
		while (_msg_que.empty() && !_b_stop)
		{
			_consume.wait(unique_lk);
		}

		if (_b_stop)
		{
			while (!_msg_que.empty())
			{
				auto msgdata = _msg_que.front();
				std::cout << "msg id is: " << msgdata->_send_node->GetMsgId() << " " 
					<< "msg data is: " << msgdata->_send_node->_data << "\n";
				auto call_back_iter = _fun_callbacks.find(msgdata->_send_node->GetMsgId());
				if (call_back_iter == _fun_callbacks.end())
				{
					_msg_que.pop();
					continue;
				}
				call_back_iter->second(msgdata->_session, msgdata->_send_node->GetMsgId(),
					std::string(msgdata->_send_node->_data, msgdata->_send_node->_cur_len));
				_msg_que.pop();
			}
			break;
		}

		auto msgdata = _msg_que.front();
		std::cout << "msg id is: " << msgdata->_send_node->GetMsgId() << " "
			<< "msg data is: " << msgdata->_send_node->_data << "\n";
		auto call_back_iter = _fun_callbacks.find(msgdata->_send_node->GetMsgId());
		if (call_back_iter == _fun_callbacks.end())
		{
			_msg_que.pop();
			continue;
		}
		call_back_iter->second(msgdata->_session, msgdata->_send_node->GetMsgId(),
			std::string(msgdata->_send_node->_data, msgdata->_send_node->_cur_len));
		_msg_que.pop();
	}
}
```
对于消息的处理，我们在一个while循环内进行处理，我们先对其进行加锁，加锁的目的是为了防止主程序多线程造成的影响，从而使得我们的逻辑层是单线程的。
如果我们的队列为空同时没有接收到服务器停止的信号，那我们就让_consume.wait()，从而不浪费过多的CPU资源。
但如果我们接收到了服务器停止的信号，那我们就取出消息队列中的所有元素，同时在Set中找到我们对应ID的回调函数，然后运行这个回调函数，并将其从队列中弹出，但是如果我们没有找到这个ID的话，那我们就将其弹出，然后continue掉。
如果不是以上这两种情况的话，那我们接着处理，处理情况跟上面类似，只不过是上面多了个while循环而言。
```cpp
LogicSystem::~LogicSystem()
{
	_b_stop = true;
	_consume.notify_one();//唤醒线程
	_worker_thread.join();//等待全部执行完毕
}
```
对于析构函数，如果我们的逻辑层要被析构的话，那说明我们的服务器已经执行了关闭操作了，那我们就将_b_stop = true，然后唤醒我们的条件变量，然后等待工作线程全部执行完毕。
```cpp
void LogicSystem::PostMsgQue(std::shared_ptr<LogicNode> logic_node)
{
	std::unique_lock<std::mutex> unique_lk(_mutex);
	_msg_que.push(logic_node);
	if (_msg_que.size() == 1)
	{
		_consume.notify_one();
	}
}
```
当向逻辑层队列投递消息的时候，我们先对其加锁，然后再投递。如果我们的队列之前是空队列，然后此时恰好是我们的第一次投递，那此时队列长度就是1，此时我们就唤醒我们的条件变量。
```cpp
class LogicNode 
{
	friend class LogicSystem;
public:
	LogicNode(std::shared_ptr<CSession> session, std::shared_ptr<RecvNode> recv_node)
		:_session(session), _recv_node(recv_node)
	{
	}
	~LogicNode()
	{
		std::cout << "LogicNode destructed" << std::endl;
	}
private:
	std::shared_ptr<CSession> _session;
	std::shared_ptr<RecvNode> _recv_node;
};
```
我们的LogicNode包含两个数据，一个是我们的消息节点，另外一个是我们的会话.
```cpp
LogicSystem::GetInstance()->PostMsgQue(std::make_shared<LogicNode>(shared_from_this(), _recv_msg_node));
```
发送时候我们采用这个方式进行发送。
### 运行结果
![picture](https://github.com/FKlightdog/Images/blob/main/LogicRun.png?raw=true)