本文是学习[**恋恋风辰Zack**](https://space.bilibili.com/271469206)asio网络编程的学习记录
### 服务器的优雅退出
服务器的退出有两种类型，一种是C语言风格的退出，另外一种是用asio的退出。
我们先给出asio形式的退出
```cpp
#include <iostream>
#include "CServer.h"
using namespace std;
int main()
{
	try {
		boost::asio::io_context  io_context;
		boost::asio::signal_set signals(io_context, SIGINT, SIGTERM);
		signals.async_wait(
			[&io_context](auto, auto)
			{
				io_context.stop();
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
我们先定义了一个信号集合，包含了SIGINT与SIGTERM，其中SIGINT对应的是Ctrl + C，SIGTERM是请求程序中止的信号，我们可以用kill来发送这个信号。
接着我们让signals进行异步的等待，他会注册给底层，在io_context.run()的时候，一旦检测到了信号，我们就触发下面的Lambda函数，让服务器停止。
接着是C语言形式的退出
```cpp
#include <iostream>
#include "CServer.h"
#include "Singleton.h"
#include "LogicSystem.h"
#include <csignal>
#include <thread>
#include <mutex>
using namespace std;
bool bstop = false;
std::condition_variable cond_quit;
std::mutex mutex_quit;
void sig_handler(int sig)
{
	if (sig == SIGINT||sig == SIGTERM)
	{
		std::unique_lock<std::mutex>  lock_quit(mutex_quit);
		bstop = true;
		lock_quit.unlock();
		cond_quit.notify_one();
	}
}

int main()
{
	try {
		boost::asio::io_context  io_context;
		std::thread  net_work_thread([&io_context] {
			CServer s(io_context, 10086);
			io_context.run();
			});
		signal(SIGINT, sig_handler);
		while (!bstop) {
			std::unique_lock<std::mutex>  lock_quit(mutex_quit);
			cond_quit.wait(lock_quit);
		}
		io_context.stop();
		net_work_thread.join();

	}
	catch (std::exception& e) {
		std::cerr << "Exception: " << e.what() << endl;
	}
	
}
```
* bstop: 用来判定是否唤醒了信号
* mmutex_quit：线程锁
* cond_quit：条件变量，用来进行等待，避免消耗过多的CPU资源

整体逻辑：
我们先让服务器跑在子线程里面，然后注册信号类型及其回调函数，一旦信号触发，就进行回调函数。
然后我们进入while循环内，如果信号一直没有被唤醒，那我们先给线程上锁，避免其他线程进行干扰，然后让cond_quit进行wait()，避免消耗过多的资源。
一旦信号被触发，我们就触发回调函数，先给线程上锁，然后让bstop = true， 给锁进行解锁，然后唤醒条件变量。
这样我们就退出了while循环，执行io_context.stop(),最后为了避免我们服务器关闭的时候，服务器还在进行其他数据传输，那我们就让服务器执行完最后的数据传输后，再停止。