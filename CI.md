# 链路层

## MAC

# IP层

## IP数据报

## ICMP

# 传输层

## UDP

## TCP

# 应用层

## HTTP

## DNS



# IO多路复用

## select
select()可以同时等待多个套接字的变为可读，只要有任意一个套接字可读，那么就会立刻返回，处理已经准备好的套接字了。

```C++
#include <sys/select.h>

int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout);

void FD_CLR(int fd, fd_set *set);
int  FD_ISSET(int fd, fd_set *set);
void FD_SET(int fd, fd_set *set);
void FD_ZERO(fd_set *set);

int pselect(int nfds, fd_set *readfds, fd_set *writefds,
            fd_set *exceptfds, const struct timespec *timeout,
            const sigset_t *sigmask);
```



- int nfds：指定待测试的描述符的个数，它的值是待测试的最大描述符加1；
- fd_set readfds：指定要让内核测试读的描述符；
- fd_set writefds：指定要让内核测试写的描述符；
- fd_set exceptfds：指定要让内核测试异常的描述符；
- timeval timeout：告知内核等待所制定描述符中的任何一个准备就绪的超时时间

fd_set，用于存储描述符集，底层使用bitmap记录描述符的。

`fd_set`是一个bitmap，由内核固定设置的大小，**最大长度为1024**，这也限制了我们最多只能同时监听1024个描述符。

与之相关的4个宏：

- FD_CLR：清除fdset中的所有bit位；
- FD_SET：开启fdset中fd描述符对应的bit位；
- FD_ZERO：关闭fdset中fd描述符对应的bit位；
- FD_ISSET：判断fd描述符对应的bit位是否开启；

**优点**
非阻塞IO直接轮训查询数据是否准备好，每次查询都要切换内核态，轮训消耗CPU。而select函数则直接把查询多个描述符的动作交给了内核，这样避免了CPU消耗和减少了内核态的切换
**缺点**
1. fd_set中的bitmap是固定1024位的，也就是说最多只能监听1024个套接字。当然也可以改内核源码，不过代价比较大；
2. **fd_set每次传入内核之后，都会被改写，导致不可重用，每次调用select都需要重新初始化fd_set；**
3. 每次调用select都需要拷贝新的fd_set到内核空间，这里会做一个用户态到内核态的切换；
4. 拿到fd_set的结果后，应用进程需要遍历整个fd_set，才知道哪些文件描述符有数据可以处理。

### 往`SELECT`中传入要让内核测试读的描述符，然后阻塞等待内核返回

![图片](https://mmbiz.qpic.cn/mmbiz_png/HNsBBIgpU8f12p5brNhpe04hhXVKuLqMBa3D3IgQvYl3JddqUib54w03aX5Vl4AxOMT3L4f5JOWRy27FGkPSzBg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 应用进程调用了select之后，会把fd_set从用户空间**拷贝到内核空间**，随后应用进程进入阻塞；
- 内核根据fd_set得到需要处理的描述符，根据描述符是否准备好数据，给**fd_set进行置位**；
- **进程被唤醒**，拿到内核处理过后的fd_set，就可以通过**FD_ISSET**判断到套接字数据是否已经准备好了

## poll
基于select的缺点，于是出现了第二个系统调用，poll，poll与内核交互的数据有所不同，并且突破了文件描述符数量的限制。

```c++
#include <poll.h>

int poll(struct pollfd *fds, nfds_t nfds, int timeout);
              // 返回：若有就绪描述符则为其数目，若超时则为0，若出错则为1
```

poll函数参数：

- pollfd * fds：指向一个结构数组第一个元素的指针，每个元素，都是一个pollfd结构，用于指定测试某个给定描述符fd的条件，结构体格式如下：

- ```c++
  struct pollfd {
    int   fd;         /* 待检测的文件描述符 */
    short events;     /* 描述符上待检测的事件类型 */
    short revents;    /* 返回描述符对应事件的状态 */
  };
  ```

- int fd：为待检测的文件描述符；

- short events：为描述符上待检测的事件类型，这里用了short类型，具体的实现用二进制掩码位操作来完成，常用的事件类型如下：

- short revents：返回描述符对应事件的状态，在pollfd由系统调用返回之后，会响应具体的事件状态；

- nfds_t nfds：nfds指定fds数组的大小；

- int timeout：指定poll函数返回前需要等待多长时间。

**优点**

- 与select类似，非阻塞IO直接轮训查询数据是否准备好，每次查询都要切换内核态，轮训消耗CPU，而poll则是把查询多个描述符的动作交给了内核，避免了CPU消耗和减少了内核态的切换。
- 与select相比，这里不是用的bitmap，而是直接用poll_fd数组，没有1024个描述符的限制；
- 这里引入了poll_fd结构体，内核只是修改poll_fd结构体中的revents，这样每次读取数据的时候，重置revents，就可以复用poll_fd了，不用像select那样反复初始化一个新的rset

**缺点**

- 每次调用poll都需要拷贝新的poll_fd到内核空间，这里会做一个用户态到内核态的切换；
- 拿到poll_fd的结果后，应用进程需要遍历整个poll_fd，才知道哪些文件描述符有数据可以处理。

## epoll

与poll不同，epoll本身不是系统调用，而是一种`内核数据结构`，它允许进程在多个文件描述符上多路复用I / O。

![image-20220106171516355](/Users/chenlei/Library/Application Support/typora-user-images/image-20220106171516355.png)

### epoll的相关函数
1. EPOLL_CREATE

   ```c++
   #include <sys/epoll.h>
   
   int epoll_create(int size);
   ```

   size参数向内核指示进程要监视的文件描述符的数量，这有助于内核确定epoll实例的大小。从Linux 2.6.8开始，此参数将被忽略，因为epoll数据结构会随着文件描述符的添加或删除而动态调整大小。

   epoll_create系统调用将返回新创建的epoll内核数据结构的文件描述符。然后，调用过程中可以使用此文件描述符来添加，删除或修改其要监视的epoll实例的I/O的其他文件描述符。

2. EPOLL_CTL

   进程可以通过调用epoll_ctl将想要监视的文件描述符添加到epoll实例。

   向epoll实例注册的所有文件描述符统称为epoll的兴趣列表[1]，会包装成epitem结构体，放到一颗红黑树rbr中：

   ![图片](https://mmbiz.qpic.cn/mmbiz_png/HNsBBIgpU8f12p5brNhpe04hhXVKuLqM6nclhoGyVGUDLq1k4BcRZRXIwC2cEgUicunrY4RlOd3ibHllIT2GBufA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

   在上图中，用户进程向epoll实例注册了文件描述符fd1，fd2，fd3，fd4，这是该epoll实例的兴趣列表集。

   当任何已注册的文件描述符准备好进行I/O时，它们就被放入事件就绪队列。事件就绪队列是兴趣列表的一个子集。**内核在接收到I/O准备好的事件回调的时候，把rbr中的epitem移到事件就绪队列。**

   ```c++
   #include <sys/epoll.h>
   
   int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
   ```

   - int epfd：epoll_create返回的文件描述符，用于标识内核中的epoll实例；
   - int op：指要在文件描述符fd上执行的操作。通常，支持三种操作：
   - EPOLL_CTL_ADD：向epoll实例注册文件描述符对应的事件；
   - EPOLL_CTL_DEL：从epoll实例注销fd。这意味着该进程将不再获取有关该文件描述符上事件的任何通知。如果已将文件描述符添加到多个epoll实例，则关闭该文件描述符会将其从添加了该文件的所有epoll兴趣列表中删除
   - EPOLL_CTL_MOD：修改文件描述符对应的事件。
   - int fd：我们要添加到epoll兴趣列表的文件描述符；
   - struct epoll_event *event：指向名为epoll_event的结构的指针，该结构存储我们实际上要监视fd的事件。

   ```c++
   typedef union epoll_data { 
     void *ptr; 
     int fd;           /* 需要监视的文件描述符 */
     uint32_t u32; 
     uint64_t u64; 
   } epoll_data_t; 
   
   struct epoll_event { 
     uint32_t events;   /* 需要监视的Epoll事件，与poll一样，基于mask的事件类型 */ 
     epoll_data_t data; /* User data variable */ 
   };
   ```

   

3. EPOLL_WAIT

   可以通过调用epoll_wait系统调用来等到内核通知进程epoll实例的兴趣列表上发生的事件，该事件将阻塞直到被监视的任何描述符准备好进行I/O操作为止。

   ```c++
   #include <sys/epoll.h>
   
   int epoll_wait(int epfd, struct epoll_event *events,
                  int maxevents, int timeout);
   
   ```

   - int epfd：epoll实例描述符；
   - struct epoll_event *events：返回给用户空间需要处理的IO事件数组；
   - int maxevents：指定epoll_wait可以返回的最大事件值；
   - int timeout：指定阻塞调用超时时间。-1表示不超时，0表示立即返回。

   ### 边缘出发和条件出发
   **边缘触发**
   当一个添加到epoll实例的epoll_event设置为EPOLLET边缘触发(edge-triggered)之后，如果后续有描述符的事件准备好了，调用epoll_wait就会把对应的epoll_event返回给应用进程，注意，在边缘触发模式下，只会返回已准备好的描述符的epoll_evnet一次，也就是说程序只有一次的处理机会。

   **条件触发**
   EPOLLET边缘触发的效率要比EPOLLLT高效，因为对于每个准备就绪的套接字，只会通知应用进程一次，但是这也要求程序员必须小心处理，不会留多次机会给你去补偿处理套接字。

   **EPOLLET边缘触发的效率要比EPOLLLT高效**，因为对于每个准备就绪的套接字，只会通知应用进程一次，但是这也要求程序员必须小心处理，不会留多次机会给你去补偿处理套接字

**优点**

- epoll每次调用epoll_wait的时候，不像poll调用一样，每次都要传递结构体到内核空间，而是复用一个内核的epoll实例结构体，通过epfd进行引用，从而减小了系统开销；
- epoll底层是套接字一旦有事件，就调用回调立刻通知epoll实例，可以尽早的准备好事件就绪队列，执行epoll_wait的时候相应的更快；
- epoll底层基于红黑树维护兴趣事件列表，这样每次套接字有新事件触发回调的时候，可以更快的找到套接字的epitem进行后续的处理；
- 提供了性能更佳的边缘触发机制。

**正是因为epoll这么多的优点，很多技术都是基于epoll实现的，如nginx、redis，以及Linux下Java的NIO。**

**缺点**

它还不是真正的异步IO，还是要应用进程调用IO函数的时候，才把数据从内核拷贝到应用进程。