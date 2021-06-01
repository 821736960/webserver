# epoll
epoll_ctl函数
```cpp
#include <sys/epoll.h>
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)
```
epfd：为epoll_creat的句柄

op：表示动作，用3个宏来表示：

EPOLL_CTL_ADD (注册新的fd到epfd)，

EPOLL_CTL_MOD (修改已经注册的fd的监听事件)，

EPOLL_CTL_DEL (从epfd删除一个fd)；

event：告诉内核需要监听的事件
```cpp
struct epoll_event {
__uint32_t events; /* Epoll events */
epoll_data_t data; /* User data variable */
};
```
events描述事件类型，其中epoll事件类型有以下几种

EPOLLIN：表示对应的文件描述符可以读（包括对端SOCKET正常关闭连接）

EPOLLOUT：表示对应的文件描述符可以写

EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）

EPOLLERR：表示对应的文件描述符发生错误

EPOLLHUP：表示对应的文件描述符被挂断；

EPOLLET：将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于默认的水平触发(Level Triggered)而言的

EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里

# LT ET
水平触发 Lt
1. 对于读操作
只要缓冲内容不为空，LT模式返回读就绪，直到你读完。

2. 对于写操作
只要缓冲区还不满，LT模式会返回写就绪。

边缘触发 ET
1. 对于读操作，
（1）当缓冲区由不可读变为可读的时候，即缓冲区由空变为不空的时候。

（2）当有新数据到达时，即缓冲区中的待读数据变多的时候。

（3）当缓冲区有数据可读，且应用进程对相应的描述符进行EPOLL_CTL_MOD 修改EPOLLIN事件时。

2. 对于写操作
（1）当缓冲区由不可写变为可写时。

（2）当有旧数据被发送走，即缓冲区中的内容变少的时候。

（3）当缓冲区有空间可写，且应用进程对相应的描述符进行EPOLL_CTL_MOD 修改EPOLLOUT事件时。

一个管道收到了1kb的数据,epoll会立即返回,此时读了512字节数据,然后再次调用epoll.
这时如果是水平触发的,epoll会立即返回,因为有数据准备好了.
如果是边缘触发的不会立即返回,因为此时虽然有数据可读但是已经触发了一次通知,在这次通知到现在还没有新的数据到来,直到有新的数据到来epoll才会返回,此时老的数据和新的数据都可以读取到(当然是需要这次你尽可能的多读取).
所以当我们写epoll网络模型时，如果我们用水平触发不用担心数据有没有读完因为下次epoll返回时，没有读完的socket依然会被返回，但是要注意这种模式下的写事件，因为是水平触发，每次socket可写时epoll都会返回，当我们写的数据包过大时，一次写不完，要多次才能写完或者每次socket写都写一个很小的数据包时，每次写都会被epoll检测到，因此长期关注socket写事件会无故cpu消耗过大甚至导致cpu跑满，

为何 epoll 的 ET 模式一定要设置为非阻塞IO？
ET模式下每次write或read需要循环write或read直到返回EAGAIN错误。以读操作为例，这是因为ET模式只在socket描述符状态发生变化时才触发事件，如果不一次把socket内核缓冲区的数据读完，会导致socket内核缓冲区中即使还有一部分数据，该socket的可读事件也不会被触发
根据上面的讨论，若ET模式下使用阻塞IO，则程序一定会阻塞在最后一次write或read操作，因此说ET模式下一定要使用非阻塞IO
# EPOLLONESHOT

一个线程读取某个socket上的数据后开始处理数据，在处理过程中该socket上又有新数据可读，此时另一个线程被唤醒读取，此时出现两个线程处理同一个socket
我们期望的是一个socket连接在任一时刻都只被一个线程处理，通过epoll_ctl对该文件描述符注册epolloneshot事件，一个线程处理socket时，其他线程将无法处理，当该线程处理完后，需要通过epoll_ctl重置epolloneshot事件

# 有限状态机
有限状态机，是一种抽象的理论模型，它能够把有限个变量描述的状态变化过程，以可构造可验证的方式呈现出来。比如，封闭的有向图。
有限状态机可以通过if-else,switch-case和函数指针来实现，从软件工程的角度看，主要是为了封装逻辑。

# http连接处理流程

浏览器端发出http连接请求，服务器端主线程创建http对象接收请求并将所有数据读入对应buffer，将该对象插入任务队列后，工作线程从任务队列中取出一个任务进行处理。

各工作线程通过process函数对任务进行处理，调用process_read函数和process_write函数分别完成报文解析与报文响应两个任务。

process_read通过while循环，将【主 从】状态机进行封装，从buf读出报文，对报文的每一行进行循环处理。
     > * 从状态机读取数据,更新自身状态和接收数据,传给主状态机
     > * 主状态机根据从状态机状态,更新自身状态,决定响应请求还是继续读取

判断条件

主状态机转移到CHECK_STATE_CONTENT，该条件涉及解析消息体

从状态机转移到LINE_OK，该条件涉及解析请求行和请求头部

两者为或关系，当条件为真则继续循环，否则退出

循环体

从状态机读取数据

调用get_line函数，通过m_start_line将从状态机读取数据间接赋给text

主状态机解析text

# http类
该类对象由主线程创建，read_once()函数读取浏览器发来的全部数据存到buffer
（1）调用构造函数初始化这对象，记录sockfd等属性；
（2）read_once()函数读取浏览器发来的全部数据存到buffer

