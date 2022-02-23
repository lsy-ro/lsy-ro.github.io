# IO多路复用

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

epoll的核心是3个API，核心数据结构是：1个红黑树和1个双向链表rdllist组成。

- epoll_create

```c
int epoll_create(int size);
```
size参数告诉内核这个epoll对象会处理的事件⼤致数量，⽽不是能够处理的事件的最⼤数(同时，size不要传0，会报invalid argument错误)。
在现在linux版本中，这个size参数已经没有意义了；
返回： epoll对象句柄；之后针对该epoll的操作需要通过该句柄来标识该epoll对象；

当某一进程调用epoll_create方法时，Linux内核会创建一个eventpoll结构体，这个结构体中有两个成员与epoll的使用方式密切相关，如下所示：

```c
struct eventpoll {
 ...
 /*红黑树的根节点，这棵树中存储着所有添加到epoll中的事件，
　　也就是这个epoll监控的事件*/
 struct rb_root rbr;
 /*双向链表rdllist保存着将要通过epoll_wait返回给用户的、满足条件的事件*/
 struct list_head rdllist;
 ...
};
```
我们在调用epoll_create时，内核除了帮我们在epoll文件系统里建了个file结点，在内核cache里建了个红黑树用于存储以后epoll_ctl传来的socket外，还会再建立一个rdllist双向链表，用于存储准备就绪的事件，当epoll_wait调用时，仅仅观察这个rdllist双向链表里有没有数据即可。有数据就返回，没有数据就sleep，等到timeout时间到后即使链表没数据也返回。所以，epoll_wait非常高效。

所有添加到epoll中的事件都会与设备(如网卡)驱动程序建立回调关系，也就是说相应事件的发生时会调用这里的回调方法。这个回调方法在内核中叫做ep_poll_callback，它会把这样的事件放到上面的rdllist双向链表中。

在epoll中对于每一个事件都会建立一个epitem结构体，如下所示：

```c
struct epitem {
 ...
 //红黑树节点
 struct rb_node rbn;
 //双向链表节点
 struct list_head rdllink;
 //事件句柄等信息
 struct epoll_filefd ffd;
 //指向其所属的eventepoll对象
 struct eventpoll *ep;
 //期待的事件类型
 struct epoll_event event;
 ...
}; // 这里包含每一个事件对应着的信息。
```

当调用epoll_wait检查是否有发生事件的连接时，只是检查eventpoll对象中的rdllist双向链表是否有epitem元素而已，如果rdllist链表不为空，则这里的事件复制到用户态内存（**使用共享内存提高效率**）中，同时将事件数量返回给用户。因此epoll_waitx效率非常高。epoll_ctl在向epoll对象中添加、修改、删除事件时，从rbr红黑树中查找事件也非常快，也就是说epoll是非常高效的，它可以轻易地处理百万级别的并发连接。

![image](https://pic2.zhimg.com/80/v2-82fe3907c6b7ed936c607b27f19586d1_720w.jpg)

【总结】：

一颗红黑树，一张准备就绪句柄链表，少量的内核cache，就帮我们解决了大并发下的socket处理问题。

执行epoll_create()时，创建了红黑树和就绪链表；

- epoll_ctl

执行epoll_ctl()时，如果增加socket句柄，则检查在红黑树中是否存在，存在立即返回，不存在则添加到树干上，然后向内核注册回调函数，用于当中断事件来临时向准备就绪链表中插入数据；
执行epoll_wait()时立刻返回准备就绪链表里的数据即可。

![image](https://pic2.zhimg.com/80/v2-c2091dcfbb96c073d6dd727613b24c25_720w.jpg)

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

## epoll的两种触发方式

epoll监控多个文件描述符的I/O事件。epoll支持边缘触发(edge trigger，ET)或水平触发（level trigger，LT)，通过epoll_wait等待I/O事件，如果当前没有可用的事件则阻塞调用线程。ET模式可以理解为状态的改变(无数据->有数据， 有数据->无数据)， 而LT可理解为一直持续的某种状态(数据不为空或者不满)。
select和poll只支持LT工作模式，epoll的默认的工作模式是LT模式。

- **水平触发的时机**

1. 对于读操作，只要缓冲内容不为空，LT模式返回读就绪
2. 对于写操作，只要缓冲区还不满，LT模式会返回写就绪。

当被监控的文件描述符上有可读写事件发生时，epoll_wait()会通知处理程序去读写。如果这次没有把数据一次性全部读写完(如读写缓冲区太小)，那么下次调用 epoll_wait()时，它还会通知你在上没读写完的文件描述符上继续读写，当然如果你一直不去读写，它会一直通知你。如果系统中有大量你不需要读写的就绪文件描述符，而它们每次都会返回，这样会大大降低处理程序检索自己关心的就绪文件描述符的效率。
**LT模式适合一次性读大数据。**

- **边缘触发的时机**

1. 当缓冲区由不可读变为可读的时候，即缓冲区由空变为不空的时候。
2. 当有新数据到达时，即缓冲区中的待读数据变多的时候。
3. 当缓冲区有数据可读，且应用进程对相应的描述符进行EPOLL_CTL_MOD 修改EPOLLIN事件时。

当被监控的文件描述符上有可读写事件发生时，epoll_wait()会通知处理程序去读写。如果这次没有把数据全部读写完(如读写缓冲区太小)，那么下次调用epoll_wait()时，它不会通知你，也就是它只会通知你一次，直到该文件描述符上出现第二次可读写事件才会通知你。这种模式比水平触发效率高，系统不会充斥大量你不关心的就绪文件描述符。
**ET模式适合读少数据**。

【epoll为什么要有EPOLLET触发模式？】：

如果采用EPOLLLT模式的话，系统中一旦有大量你不需要读写的就绪文件描述符，它们每次调用epoll_wait都会返回，这样会大大降低处理程序检索自己关心的就绪文件描述符的效率.。而采用EPOLLET这种边缘触发模式的话，当被监控的文件描述符上有可读写事件发生时，epoll_wait()会通知处理程序去读写。如果这次没有把数据全部读写完(如读写缓冲区太小)，那么下次调用epoll_wait()时，它不会通知你，也就是它只会通知你一次，直到该文件描述符上出现第二次可读写事件才会通知你！！！这种模式比水平触发效率高，系统不会充斥大量你不关心的就绪文件描述符。

## epoll与select、poll的对比

1. **用户态将文件描述符传入内核的方式**
    1. select：创建3个文件描述符(**bitmap**)集并拷贝到内核中，分别监听读、写、异常动作。这里受到单个进程可以打开的fd数量限制，默认是1024。
    2. poll：将传入的struct pollfd**结构体数组**拷贝到内核中进行监听。
    3. epoll：执行epoll_create会在内核的高速cache区中建立一颗**红黑树**以及**就绪链表**(该链表位于用户态与内核态的共享内存中，并存储已经就绪的文件描述符)。接着用户执行的epoll_ctl函数添加文件描述符会在红黑树上增加相应的结点。
    
2.  **内核态检测文件描述符读写状态的方式**
    1. select：采用轮询方式，遍历所有fd，最后返回一个描述符读写操作是否就绪的mask掩码，根据这个掩码给fd_set赋值。
    2. poll：同样采用轮询方式，查询每个fd的状态，如果就绪则在等待队列中加入一项并继续遍历。
    3. epoll：采用回调机制。在执行epoll_ctl的add操作时，不仅将文件描述符放到红黑树上，而且也注册了回调函数，内核在检测到某文件描述符可读/可写时会调用回调函数，该回调函数将文件描述符放在就绪链表中。
    
3. **到就绪的文件描述符并传递给用户态的方式**
    1. select：将之前传入的fd_set拷贝传出到用户态并返回就绪的文件描述符总数。用户态并不知道是哪些文件描述符处于就绪态，需要遍历来判断
    2. poll：将之前传入的fd数组拷贝传出用户态并返回就绪的文件描述符总数。用户态并不知道是哪些文件描述符处于就绪态，需要遍历来判断。
    3. epoll：epoll_wait只用观察就绪链表中有无数据即可，最后将链表的数据返回给数组并返回就绪的数量。内核将就绪的文件描述符放在传入的数组中，所以只用遍历依次处理即可。这里返回的文件描述符是通过mmap让内核和用户空间共享同一块内存实现传递的，减少了不必要的拷贝。
    
4. **重复监听的处理方式**
    1. select：将新的监听文件描述符集合拷贝传入内核中，继续以上步骤。
    2. poll：将新的struct pollfd结构体数组拷贝传入内核中，继续以上步骤。
    3. epoll：无需重新构建红黑树，直接沿用已存在的即可。
    
## epoll更高效的原因

1. select和poll的动作基本一致，只是select采用结构体数组来进行文件描述符的存储，而poll采用bitmap fd标注位来存放，所以select会受到最大连接数的限制，而poll不会。
2. select、poll、epoll虽然都会返回就绪的文件描述符数量。但是select和poll并不会明确指出是哪些文件描述符就绪，而epoll会。造成的区别就是，系统调用返回后，调用select和poll的程序需要遍历监听的整个文件描述符找到是谁处于就绪，而epoll则直接处理即可。
3. select、poll都需要将有关文件描述符的数据结构拷贝进内核，最后再拷贝出来。而epoll创建的有关文件描述符的数据结构本身就存于内核态中，系统调用返回时利用mmap()文件映射内存加速与内核空间的消息传递：即epoll使用mmap减少复制开销。
4. select、poll采用轮询的方式来检查文件描述符是否处于就绪态，而epoll采用回调机制。造成的结果就是，随着fd的增加，select和poll的效率会线性降低，而epoll不会受到太大影响，除非活跃的socket很多。
5. epoll的边缘触发模式效率高，系统不会充斥大量不关心的就绪文件描述符

虽然epoll的性能最好，但是在连接数少并且连接都十分活跃的情况下，select和poll的性能可能比epoll好，毕竟epoll的通知机制需要很多函数回调。

## ep->wq 的作用是什么？

wq 是一个等待队列，用来保存对某一个 epoll 实例调用 epoll_wait()的所有进程。

一个进程调用 epoll_wait()后，如果当前还没有任何事件发生，需要让当前进程挂起等待（放到 ep->wq 里）；当 epoll 实例监视的文件上有事件发生后，需要唤醒 ep->wq 上的进程去继续执行用户态的业务逻辑。之所以要用一个等待队列来维护关注这个 epoll 的进程，是因为有时候调用 epoll_wait()的不只一个进程，当多个进程都在关注同一个 epoll 实例时，休眠的进程们通过这个等待队列就可以逐个被唤醒了。

多个进程关注同一个 epoll 实例，那么有事件发生后先唤醒谁？后唤醒谁？还是一起全唤醒？这涉及到一个称为“惊群效应”的问题。

## ep->rdllist 的作用是什么？

epoll 实例中包含就绪事件的 fd 组成的链表。

通过扫描 ep->rdllist 链表，内核可以轻松获取当前有事件触发的 fd。而不是像 select()/poll() 那样全量扫描所有被监视的 fd，再从中找出有事件就绪的。因此可以说这一点决定了 epoll 的性能是远高于 select/poll 的。

看到这里你可能又产生了一个小小的疑问：为什么 epoll 中事件就绪的 fd 会“主动”跑到 rdllist 中去，而不用全量扫描就能找到它们呢？ 这是因为每当调用 epoll_ctl 新增一个被监视的 fd 时，都会注册一下这个 fd 的回调函数 ep_poll_callback， 当网卡收到数据包会触发一个中断，中断处理函数再回调 ep_poll_callback 将这个 fd 所属的“epitem”添加至 epoll 实例中的 rdllist 中。

## epmutex、ep->mtx、ep->lock 3 把锁的区别是？
锁的粒度和使用目的不同。

epmutex 是一个全局互斥锁，epoll 中一共只有 3 个地方用到这把锁。分别是 ep_free() 销毁一个 epoll 实例时、eventpoll_release_file() 清理从 epoll 中已经关闭的文件时、epoll_ctl() 时避免 epoll 间嵌套调用时形成死锁。我的理解是 epmutex 的锁粒度最大，用来处理跨 epoll 实例级别的同步操作。

ep->mtx 是一个 epoll 内部的互斥锁，在 ep_scan_ready_list() 扫描就绪列表、eventpoll_release_file() 中执行 ep_remove()删除一个被监视文件、ep_loop_check_proc()检查 epoll 是否有循环嵌套或过深嵌套、还有 epoll_ctl() 操作被监视文件增删改等处有使用。可以看出上述的函数里都会涉及对 epoll 实例中 rdllist 或红黑树的访问，因此我的理解是 ep->mtx 是一个 epoll 实例内的互斥锁，用来保护 epoll 实例内部的数据结构的线程安全。

ep->lock 是一个 epoll 实例内部的自旋锁，用来保护 ep->rdllist 的线程安全。自旋锁的特点是得不到锁时不会引起进程休眠，所以在 ep_poll_callback 中只能使用 ep->lock，否则就会丢事件。

>[简谈epoll](https://blog.csdn.net/u014183456/article/details/121441877?spm=1001.2014.3001.5501)
>
>[epoll原理](https://zhuanlan.zhihu.com/p/381646420)
