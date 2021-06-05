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
```cpp
//循环读取客户数据，直到无数据可读或对方关闭连接
//非阻塞ET工作模式下，需要一次性将数据读完，读取到m_read_buffer中
bool http_conn::read_once()
{
    if (m_read_idx >= READ_BUFFER_SIZE)//m_read_idx 已经读到buf的数据
    {
        return false;
    }
    int bytes_read = 0;

#ifdef connfdLT //LT模式，不用一次性读完
    //从内核buffer拷贝数据到用户层buffer
    bytes_read = recv(m_sockfd, m_read_buf + m_read_idx, READ_BUFFER_SIZE - m_read_idx, 0);
    m_read_idx += bytes_read;//更新已经读到buf的数据

    if (bytes_read <= 0)
    {
        return false;
    }

    return true;

#endif

#ifdef connfdET //ET模式,循环读取直到读完
    while (true)
    {   //成功执行时，返回接收到的字节数;另一端已关闭则返回0;失败返回-1，errno被设为以下的某个值 ：https://blog.csdn.net/mercy_ps/article/details/82224128
        //https://www.cnblogs.com/ellisonzhang/p/10412021.html
        bytes_read = recv(m_sockfd, m_read_buf + m_read_idx, READ_BUFFER_SIZE - m_read_idx, 0);
        if (bytes_read == -1)
        {
            if (errno == EAGAIN || errno == EWOULDBLOCK)//非阻塞模式下的两种情况，表明et已经读完
                break;
            return false;
        }
        else if (bytes_read == 0)//另一端已关闭则返回0
        {
            return false;
        }
        m_read_idx += bytes_read;
    }
    return true;
#endif
}
```
# recv函数
int recv( SOCKET s, char FAR *buf, int len, int flags);

每个TCP socket在内核中都有一个发送缓冲区和一个接收缓冲区，TCP的全双工的工作模式以及TCP的流量(拥塞)控制便是依赖于这两个独立的buffer以及buffer的填充状态。接收缓冲区把数据缓存入内核，应用进程一直没有调用recv()进行读取的话，此数据会一直缓存在相应socket的接收缓冲区内。再啰嗦一点，不管进程是否调用recv()读取socket，对端发来的数据都会经由内核接收并且缓存到socket的内核接收缓冲区之中。recv()所做的工作，就是把内核缓冲区中的数据拷贝到应用层用户的buffer里面，并返回，仅此而已。  
默认 socket 是阻塞的，阻塞与非阻塞recv返回值没有区分，都是 <0 出错， =0 连接关闭， >0 接收到数据大小。

失败返回-1，errno被设为以下的某个值 ：
EAGAIN：套接字已标记为非阻塞，而接收操作被阻塞或者接收超时 
EBADF：sock不是有效的描述词 
ECONNREFUSE：远程主机阻绝网络连接 
EFAULT：内存空间访问出错 
EINTR：操作被信号中断 
EINVAL：参数无效 
ENOMEM：内存不足 
ENOTCONN：与面向连接关联的套接字尚未被连接上 
ENOTSOCK：sock索引的不是套接字 当返回值是0时，为正常关闭连接；
EWOULDBLOCK：用于非阻塞模式，不需要重新读或者写


# 工作线程中的主从状态机
各子线程通过process函数对任务进行处理，调用process_read函数和process_write函数分别完成报文解析与报文响应两个任务。

 # process 函数
 ```cpp
 1void http_conn::process()
 2{
 3    HTTP_CODE read_ret=process_read();
 4
 5    //NO_REQUEST，表示请求不完整，需要继续接收请求数据
 6    if(read_ret==NO_REQUEST)
 7    {
 8        //注册并监听读事件
 9        modfd(m_epollfd,m_sockfd,EPOLLIN);
10        return;
11    }
12
13    //调用process_write完成报文响应
14    bool write_ret=process_write(read_ret);
15    if(!write_ret)
16    {
17        close_conn();
18    }
19    //注册并监听写事件
20    modfd(m_epollfd,m_sockfd,EPOLLOUT);
21}
```

 # process_read() 函数
 从状态机负责读取报文的一行，主状态机负责对该行数据进行解析，主状态机内部调用从状态机，从状态机驱动主状态机
 通过while循环，将主从状态机进行封装，对报文的每一行进行循环处理。
 
 HTTP_CODE含义:  表示HTTP请求的处理结果，在头文件中初始化了八种情形，在报文解析时只涉及到四种。

NO_REQUEST:请求不完整，需要继续读取请求报文数据  
GET_REQUEST:获得了完整的HTTP请求  
BAD_REQUEST:HTTP请求报文有语法错误  
INTERNAL_ERROR:服务器内部错误，该结果在主状态机逻辑switch的default下，一般不会触发  
#  状态机总逻辑
 ```cpp
 //
 //m_start_line是行在buffer中的起始位置，将该位置后面的数据赋给text
 //此时从状态机已提前将一行的末尾字符\r\n变为\0\0，所以text可以直接取出完整的行进行解析
 char* get_line(){
     return m_read_buf+m_start_line;
 }
 
http_conn::HTTP_CODE http_conn::process_read()
{    //初始化从状态机状态、HTTP请求解析结果
    LINE_STATUS line_status = LINE_OK;
    HTTP_CODE ret = NO_REQUEST;
    char *text = 0;
    //这里为什么要写两个判断条件？第一个判断条件为什么这样写？\
    //parse_line为从状态机的具体实现
    while ((m_check_state == CHECK_STATE_CONTENT && line_status == LINE_OK) || ((line_status = parse_line()) == LINE_OK))
    {
        text = get_line();
        //m_start_line是每一个数据行在m_read_buf中的起始位置
       //m_checked_idx表示从状态机在m_read_buf中读取的位置,都是相对位置
        m_start_line = m_checked_idx;
        LOG_INFO("%s", text);
        Log::get_instance()->flush();
        
        //主状态机的三种状态转移逻辑
        switch (m_check_state)
        {
        case CHECK_STATE_REQUESTLINE: //解析请求行
        {
            ret = parse_request_line(text);
            if (ret == BAD_REQUEST)
                return BAD_REQUEST;
            break;
        }
        case CHECK_STATE_HEADER://解析请求头
        {
            ret = parse_headers(text);
            if (ret == BAD_REQUEST)
                return BAD_REQUEST;
             //get请求没有主体，返回GET_REQUEST，完整解析GET请求后，跳转到报文响应函数
            else if (ret == GET_REQUEST)
            {
                return do_request();
            }
            break;
        }
        case CHECK_STATE_CONTENT://解析消息体
        {   
            ret = parse_content(text);
            if (ret == GET_REQUEST)//完整解析POST请求后，跳转到报文响应函数
                return do_request();
            //解析完消息体即完成报文解析，避免再次进入循环，更新line_status
            line_status = LINE_OPEN;
            break;
        }
        default:
            return INTERNAL_ERROR;
        }
    }
    return NO_REQUEST;
}
```
那么，这里的判断条件为什么要写成这样呢？

在GET请求报文中，每一行都是\r\n作为结束，所以对报文进行拆解时，仅用从状态机的状态line_status=parse_line())==LINE_OK语句即可。

但，在POST请求报文中，消息体的末尾没有任何字符，所以不能使用从状态机的状态，这里转而使用主状态机的状态作为循环入口条件。

那后面的&& line_status==LINE_OK又是为什么？

解析完消息体后，报文的完整解析就完成了，但此时主状态机的状态还是CHECK_STATE_CONTENT，也就是说，符合循环入口条件，还会再次进入循环，这并不是我们所希望的。

为此，增加了该语句，并在完成消息体解析后，将line_status变量更改为LINE_OPEN，此时可以跳出循环，完成报文解析任务。

# 从状态机逻辑

在HTTP报文中，每一行的数据由\r\n作为结束字符，空行则是仅仅是字符\r\n。因此，可以通过查找\r\n将报文拆解成单独的行进行解析，项目中便是利用了这一点。

从状态机负责读取buffer中的数据，将每行数据末尾的\r\n置为\0\0，并更新从状态机在buffer中读取的位置m_checked_idx，以此来驱动主状态机解析。

从状态机从m_read_buf中逐字节读取，判断当前字节是否为\r

接下来的字符是\n，将\r\n修改成\0\0，将m_checked_idx指向下一行的开头，则返回LINE_OK

接下来达到了buffer末尾，表示buffer还需要继续接收，返回LINE_OPEN

否则，表示语法错误，返回LINE_BAD

当前字节不是\r，判断是否是\n（一般是上次读取到\r就到了buffer末尾，没有接收完整，再次接收时会出现这种情况）

如果前一个字符是\r，则将\r\n修改成\0\0，将m_checked_idx指向下一行的开头，则返回LINE_OK

当前字节既不是\r，也不是\n

表示接收不完整，需要继续接收，返回LINE_OPEN
```cpp
//从状态机，用于分析出一行内容
//返回值为行的读取状态，有LINE_OK,LINE_BAD,LINE_OPEN
//m_read_idx指向缓冲区m_read_buf的数据末尾的下一个字节,即buf长度
 //m_checked_idx指向从状态机当前正在分析的字节
 
http_conn::LINE_STATUS http_conn::parse_line()
{
    char temp;//temp为将要分析的字节
    for (; m_checked_idx < m_read_idx; ++m_checked_idx)
    {
        temp = m_read_buf[m_checked_idx];
        if (temp == '\r')//如果当前是\r字符，则有可能会读取到完整行
        {
            if ((m_checked_idx + 1) == m_read_idx)//下一个字符达到了buffer结尾，则接收不完整，需要继续接收
                return LINE_OPEN;
            else if (m_read_buf[m_checked_idx + 1] == '\n')//下一个字符是\n，将\r\n改为\0\0
            {
                m_read_buf[m_checked_idx++] = '\0';
                m_read_buf[m_checked_idx++] = '\0';
                return LINE_OK;
            }
            return LINE_BAD;//如果都不符合，则返回语法错误
        }
        else if (temp == '\n')//如果当前字符是\n，也有可能读取到完整行
       //一般是上次读取到\r就到buffer末尾了，没有接收完整，再次接收时会出现这种情况
        {
            if (m_checked_idx > 1 && m_read_buf[m_checked_idx - 1] == '\r')//前一个字符是\r，则接收完整
            {
                m_read_buf[m_checked_idx - 1] = '\0';
                m_read_buf[m_checked_idx++] = '\0';
                return LINE_OK;
            }
            return LINE_BAD;
        }
    }
    //并没有找到\r\n，需要继续接收
    return LINE_OPEN;
}

```

# 主状态机逻辑
主状态机初始状态是CHECK_STATE_REQUESTLINE，通过调用从状态机来驱动主状态机，在主状态机进行解析前，从状态机已经将每一行的末尾\r\n符号改为\0\0，以便于主状态机直接取出对应字符串进行处理。
## CHECK_STATE_REQUESTLINE

主状态机的初始状态，调用parse_request_line函数解析请求行

解析函数从m_read_buf中解析HTTP请求行，获得请求方法、目标URL及HTTP版本号

解析完成后主状态机的状态变为CHECK_STATE_HEADER



## CHECK_STATE_HEADER
解析完请求行后，主状态机继续分析请求头。在报文中，请求头和空行的处理使用的同一个函数，这里通过判断当前的text首位是不是\0字符，若是，则表示当前处理的是空行，若不是，则表示当前处理的是请求头。

调用parse_headers函数解析请求头部信息

判断是空行还是请求头，若是空行，进而判断content-length是否为0，如果不是0，表明是POST请求，则状态转移到CHECK_STATE_CONTENT，否则说明是GET请求，则报文解析结束。

若解析的是请求头部字段，则主要分析connection字段，content-length字段，其他字段可以直接跳过，各位也可以根据需求继续分析。

connection字段判断是keep-alive还是close，决定是长连接还是短连接

content-length字段，这里用于读取post请求的消息体长度

## CHECK_STATE_CONTENT

仅用于解析POST请求，调用parse_content函数解析消息体

用于保存post请求消息体，为后面的登录和注册做准备


