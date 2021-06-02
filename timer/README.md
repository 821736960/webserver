
定时器处理非活动连接
===============
由于非活跃连接占用了连接资源，严重影响服务器的性能，通过实现一个服务器定时器，处理这种非活跃连接，释放连接资源。利用alarm函数周期性地触发SIGALRM信号,该信号的信号处理函数利用管道通知主循环执行定时器链表上的定时任务.
> * 统一事件源
> * 基于升序链表的定时器
> * 处理非活动连接
> 
# 信号的异步
信号是在软件层次上对中断机制的一种模拟，在原理上，一个进程收到一个信号与处理器收到一个中断请求可以说是一样的。信号是异步的，一个进程不必通过任何操作来等待信号的到达，事实上，进程也不知道信号到底什么时候到达。

Linux下的信号采用的异步处理机制，信号处理函数和当前进程是两条不同的执行路线。具体的，当进程收到信号时，操作系统会中断进程当前的正常流程，转而进入信号处理函数执行操作，完成后再返回中断的地方继续执行。

为避免信号竞态现象发生，信号处理期间系统不会再次触发它，即屏蔽。所以，为确保该信号不被屏蔽太久，信号处理函数需要尽可能快地执行完毕。

一般的信号处理函数需要处理该信号对应的逻辑，当该逻辑比较复杂时，信号处理函数执行时间过长，会导致信号屏蔽太久。

这里的解决方案是，信号处理函数仅仅发送信号通知程序主循环，将信号对应的处理逻辑放在程序主循环中，由主循环执行信号对应的逻辑代码。

## 统一事件源：

创建管道，其中管道写端写入信号值，管道读端通过I/O复用系统epoll监测读事件

设置信号处理函数SIGALRM（时间到了触发）和SIGTERM（kill会触发，Ctrl+C）

通过struct sigaction结构体和sigaction函数注册信号捕捉函数（捕捉到了SIGALRM或者SIGTERM后向管道写入信号）

利用epoll I/O复用系统监听管道读端文件描述符的可读事件

信息值传递给主循环，主循环再根据接收到的信号值执行目标信号对应的逻辑代码

```cpp
 1//创建管道套接字，用来传递信号
 2ret = socketpair(PF_UNIX, SOCK_STREAM, 0, pipefd);
 3assert(ret != -1);
 4
 5//设置管道写端为非阻塞，为什么写端要非阻塞？
 6setnonblocking(pipefd[1]);
 7
 8//设置管道读端为ET非阻塞
 9addfd(epollfd, pipefd[0], false);
10
11//传递给主循环的信号值，这里只关注SIGALRM和SIGTERM
12addsig(SIGALRM, sig_handler, false);
13addsig(SIGTERM, sig_handler, false);
14
15//循环条件
16bool stop_server = false;
17
18//超时标志
19bool timeout = false;
20
21//每隔TIMESLOT时间触发SIGALRM信号
22alarm(TIMESLOT);
23
24while (!stop_server)
25{
26    //监测发生事件的文件描述符
27    int number = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1);
28    if (number < 0 && errno != EINTR)
29    {
30        break;
31    }
32
33    //轮询文件描述符
34    for (int i = 0; i < number; i++)
35    {
36        int sockfd = events[i].data.fd;
37
38        //管道读端对应文件描述符发生读事件
39        if ((sockfd == pipefd[0]) && (events[i].events & EPOLLIN))
40        {
41            int sig;
42            char signals[1024];
43
44            //从管道读端读出信号值，成功返回字节数，失败返回-1
45            //正常情况下，这里的ret返回值总是1，只有14和15两个ASCII码对应的字符
46            ret = recv(pipefd[0], signals, sizeof(signals), 0);
47            if (ret == -1)
48            {
49                // handle the error
50                continue;
51            }
52            else if (ret == 0)
53            {
54                continue;
55            }
56            else
57            {
58                //处理信号值对应的逻辑
59                for (int i = 0; i < ret; ++i)
60                {
61                    //这里面明明是字符
62                    switch (signals[i])
63                    {
64                    //这里是整型
65                    case SIGALRM:
66                    {
67                        timeout = true;
68                        break;
69                    }
70                    case SIGTERM:
71                    {
72                        stop_server = true;
73                    }
74                    }
75                }
76            }
77        }
78    }
79}
```
## 为什么管道写端要非阻塞？

send是将信息发送给套接字缓冲区，如果缓冲区满了，则会阻塞，这时候会进一步增加信号处理函数的执行时间，为此，将其修改为非阻塞。

## 没有对非阻塞返回值处理，如果阻塞是不是意味着这一次定时事件失效了？

是的，但定时事件是非必须立即处理的事件，可以允许这样的情况发生。

## 管道传递的是什么类型？switch-case的变量冲突？

信号本身是整型数值，管道中传递的是ASCII码表中整型数值对应的字符。

switch的变量一般为字符或整型，当switch的变量为字符时，case中可以是字符，也可以是字符对应的ASCII码。
