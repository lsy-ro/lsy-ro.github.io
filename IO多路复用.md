# IO多路复用

[IO多路复用](https://blog.csdn.net/u014183456/article/details/121441877?spm=1001.2014.3001.5501)

## 阻塞IO模型和非阻塞IO模型
- 阻塞在哪里？
- 什么来决定阻塞还是非阻塞

```c
//连接的fd
fcntl(c->fd, F_SETFL, O_NONBLOCK);
```

- 阻塞和非阻塞具体的差异是什么?

![image](https://img-blog.csdnimg.cn/06e2e9ba24e54b1491b976f39024358c.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYmlyYXRlX-Wwj-Wwj-S6uueUnw==,size_18,color_FFFFFF,t_70,g_se,x_16)
![image](https://img-blog.csdnimg.cn/4bda80278d7741df8df0f6fe49a801ef.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAYmlyYXRlX-Wwj-Wwj-S6uueUnw==,size_17,color_FFFFFF,t_70,g_se,x_16)
阻塞模型和非阻塞模型主要区别在数据准备阶段是否立刻返回。<br>
如果是在阻塞模型，数据准备阶段和数据拷贝阶段都会被阻塞，处理时间会比较长;<br>
在非阻塞模型中，如果在数据准备阶段，调用read/recv会立刻给一个结果(未准备好返回-1)。

## IO多路复用
I/O多路复用主要有slect, poll 和 epoll 三个主要的函数。这里我们主要介绍epoll函数。**实现的机制主要是使用一个线程来检测多个io事件**，把相应的事件fd添加到epoll中，使用epoll来管理。如果读写事件准备好时，epoll会触发相应的事件来通知到用户。

### epoll的API

epoll的核心是3个API，核心数据结构是：1个红黑树和1个链表组成。

- epoll_create

```c
int epoll_create(int size);
```
size参数告诉内核这个epoll对象会处理的事件⼤致数量，⽽不是能够处理的事件的最⼤数(同时，size不要传0，会报invalid argument错误)。
在现在linux版本中，这个size参数已经没有意义了；
返回： epoll对象句柄；之后针对该epoll的操作需要通过该句柄来标识该epoll对象；

- epoll_ctl
```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event* event);

typedef union epoll_data {
    void *ptr;      /*指向用户自定义数据*/
    int fd;         /*注册的文件描述符*/
    uint32_t u32;
    uint64_t u64;
}epoll_data_t;

struct epoll_event {
    uint32_t events;    /*描述epoll事件*/
    epoll_data_t data;
};

```
epoll_ctl向epoll对象添加、修改或删除事件；
返回： 0表示成功， -1表示错误，根据errno错误码判断错误类型。

**op类型:**
```c
EPOLL_CTL_ADD // 添加新的事件到epoll中
EPOLL_CTL_MOD // 修改epoll中的事件
EPOLL_CTL_DEL // 删除epoll中的事件
```

**event.events取值:**
```c
EPOLLIN         // 表示该连接上有数据可读（tcp连接远端主动关系连接，也是可读事件，因为需要处理发送来的FIN包，FIN包就是read返回0）
EPOLLOUT        // 表示该连接上可写发送（主动向上游服务器发起非阻塞tcp连接，连接建立成功事件相当于可写事件）
EPOLLRDHUP      // 表示tcp连接的远端关闭或半关闭连接
EPOLLPRI        // 表示连接上有紧急数据需要读
EPOLLERR        // 表示连接上发生错误
EROLLHUP        // 表示连接被挂起
EPOLLET         // 讲连接设置为边缘出发，系统默认为水平触发
EPOLLONESHOT    // 表示该事件只处理一次，下次需要处理时需重新加入epoll
```

- epoll_wait
```c
int epoll_wait(int epfd, struc epoll_event* events, int maxevents, int timeout);
```
收集 epoll 监控的事件中已经发⽣的事件，如果 epoll 中没有任何⼀个事件发⽣，则最多等待timeout 毫秒后返回。<br>
返回：表示当前发⽣的事件个数<br>
返回0表示本次没有事件发⽣；<br>
返回-1表示出现错误，需要检查errno错误码判断错误类型。

**注意：**

events 这个数组必须在⽤户态分配内存，内核负责把就绪事件复制到该数组中;<br>
maxevents 表示本次可以返回的最⼤事件数⽬，⼀般设置为 events 数组的⻓度;<br>
timeout表示在没有检测到事件发⽣时最多等待的时间；如果设置为0，检测到rdllist为空⽴刻返回；如果设置为-1，⼀直等待;<br>
所有添加到epoll中的事件都会与⽹卡驱动程序建⽴回调关系，相应的事件发⽣时会调⽤这⾥的回调⽅法（ep_poll_callback） ,它会把这样的事件放在rdllist双向链表中。
