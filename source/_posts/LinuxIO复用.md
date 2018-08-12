---
title: LinuxIO复用
date: 2016-03-01 00:53:03
tags: LinuxAPI
categories: Linux
---

# 1.IO复用简介

I/O 多路复用技术是为了解决进程或线程阻塞到某个 I/O 系统调用而出现的技术，使进程不阻塞于某个特定的 I/O 系统调用。

`select()`，`poll()`，`epoll()`都是I/O多路复用的机制。I/O多路复用通过一种机制，可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪，就是这个文件描述符进行读写操作之前），能够通知程序进行相应的读写操作。但`select()`，`poll()`，`epoll()`本质上都是**同步I/O**，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而`异步I/O`则无需自己负责进行读写，`异步I/O`的实现会负责把数据从内核拷贝到用户空间。
与多线程和多进程相比，`I/O 多路复用`的最大优势是系统开销小，系统不需要建立新的进程或者线程，也不必维护这些线程和进程。

<!--more-->
# 2.select

## 2.1 api说明

```c
#include <sys/select.h>
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

监视并等待多个文件描述符的属性变化（`可读`、`可写`或`错误异常`）。`select()`函数监视的文件描述符分 3 类，分别是`writefds`、`readfds`、和 `exceptfds`。调用后 `select()`函数会阻塞，直到有描述符就绪（有数据可读、可写、或者有错误异常），或者超时（ timeout 指定等待时间），函数才返回。当`select()`函数返回后，可以通过遍历 `fdset`，来找到就绪的描述符。

* `nfds`: 要监视的文件描述符的范围，一般取监视的描述符数的最大值+1，如这里写 10， 这样的话，描述符 0，1, 2 …… 9 都会被监视，在 Linux 上最大值一般为1024。
* `readfd`: 监视的可读描述符集合，只要有文件描述符即将进行读操作，这个文件描述符就存储到这。
* `writefds`: 监视的可写描述符集合。
* `exceptfds`: 监视的错误异常描述符集合

中间的三个参数 readfds、writefds 和 exceptfds 指定我们要让内核监测读、写和异常条件的描述字。如果不需要使用某一个的条件，就可以把它设为空指针（ NULL ）。集合fd_set 中存放的是文件描述符，可通过以下四个宏进行设置：

```c
////清空集合
void FD_ZERO(fd_set *fdset); 
//将一个给定的文件描述符加入集合之中
void FD_SET(int fd, fd_set *fdset);
//将一个给定的文件描述符从集合中删除
void FD_CLR(int fd, fd_set *fdset);
// 检查集合中指定的文件描述符是否可以读写 
int FD_ISSET(int fd, fd_set *fdset); 
```

* `timeout`: 超时时间，它告知内核等待所指定描述字中的任何一个就绪可花多少时间。其 timeval 结构用于指定这段时间的秒数和微秒数。

```c
struct timeval{
	time_t tv_sec;//秒
	suseconds_t tv_usec;//微秒
};
```

这个参数有三种可能：

* `永远等待下去`：仅在有一个描述字准备好`I/O`时才返回。为此，把该参数设置为空指针 `NULL`。
* `等待固定时间`：在指定的固定时间（ timeval 结构中指定的秒数和微秒数）内，在有一个描述字准备好`I/O`时返回，如果时间到了，就算没有文件描述符发生变化，这个函数会返回 0。
* `根本不等待（不阻塞)`:检查描述字后立即返回，这称为轮询。为此，`struct timeval`变量的时间值指定为 0 秒 0 微秒，文件描述符属性无变化返回 0，有变化返回准备好的描述符数量。

返回值：

* `成功`：就绪描述符的数目，超时返回 0，
* `出错`：-1

## 2.3 示例

```
#include <stdio.h>
#include <stdlib.h>
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>

int mian(void){
	fd_set rfds;
	struct timeval tv;
	int retval;
	//监控标准输入stdin(0),看什么时候输入
	FD_ZERO(&rfds);
	FD_SET(0,&rfds);
	//等待5s
	tv.tv_sec=5;
	tv.tv_usec=0;
	retval=select(1,&rfds,NULL,NULL,NULL,&tv);
	//现在不用依赖tv的值了
	if(retval==-1){
		perror("select()");
	}else if(retval){
		printf("Data is available now \n");
		//FD_ISSET(0,&rfds)的值为true
	}else{
		printf("No data within five seconds");
	}
	exit(EXIT_SUCCESS);
}
```

## 2.4 缺点

1）每次调用`select()`，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大，同时每次调用`select()`都需要在内核遍历传递进来的`所有fd`，这个开销在fd很多时也很大。

2）单个进程能够监视的文件描述符的数量存在最大限制，在 Linux 上一般为 1024，可以通过修改宏定义甚至重新编译内核的方式提升这一限制，但是这样也会造成效率的降低。

# 3.poll
## 3.1 api说明
`select()`和`poll()`系统调用的本质一样，前者在 BSD UNIX 中引入的，后者在 System V 中引入的。`poll()`的机制与`select()`类似，与`select()`在本质上没有多大差别，管理多个描述符也是进行轮询，根据描述符的状态进行处理，但是`poll()`**没有最大文件描述符数量的限**（但是数量过大后性能也是会下降）。`poll()`和`select()`同样存在一个缺点就是，包含大量文件描述符的数组被整体复制于用户态和内核的地址空间之间，而不论这些文件描述符是否就绪，它的开销随着文件描述符数量的增加而线性增大。

```
#include <poll.h>
int poll(struct pollfd *fds,nfds_t nfds,int timeout);
```

监视并等待多个文件描述符的属性变化。

* 1.`fds`: 不同与`select()`使用三个位图来表示三个fdset的方式，`poll()`使用一个 `pollfd的指针`实现。一个pollfd结构体数组，其中包括了你想测试的`文件描述符`和`事件`, **事件由结构中事件域events来确定**，**调用后实际发生的时间将被填写在结构体的 revents域**。

```c
struct pollfd{
	int fd;//文件描述符
	short events;//等待的事件
	short reevents;//实际发生了的事件
};
```

* `fd`：每一个`pollfd`结构体指定了一个被监视的文件描述符，可以传递多个结构体，指示 `poll()`监视多个文件描述符。

* `events`：每个结构体的events域是监视该文件描述符的事件掩码，由用户来设置这个域。events 等待事件的掩码取值如下：
	* 处理输入：
		* `POLLIN`普通或优先级带数据可读
		* `POLLRDNORM` 普通数据可读
		* `POLLRDBAND` 优先级带数据可读
		* `POLLPRI` 高优先级数据可读
	* 处理输出
		* `POLLOUT`普通或优先级带数据可写
		* `POLLWRNORM`普通数据可写
		* `POLLWRBAND`优先级带数据可写
	* 处理错误
		* `POLLERR`发生错误
		* `POLLHUP`发生挂起
		* `POLLVAL`描述字不是一个打开的文件

`poll()`处理三个级别的数据，普通 normal，优先级带`priority band`，高优先级`high priority`，这些都是出于流的实现。

`POLLIN | POLLPRI`等价于`select()`的读事件,`POLLOUT | POLLWRBAND`等价于 select() 的写事件。`POLLIN`等价于 `POLLRDNORM | POLLRDBAND`，而`POLLOUT` 则等价于`POLLWRNORM`。例如，要同时监视一个文件描述符是否可读和可写，我们可以设置events为`POLLIN | POLLOUT`。

* `revents`：revents域是文件描述符的操作`结果事件掩码`，内核在调用返回时设置这个域。events 域中请求的任何事件都可能在 revents域中返回。

每个结构体的 events 域是由用户来设置，告诉内核我们关注的是什么，而 revents 域是返回时内核设置的，以说明对该描述符发生了什么事件。

* 2.`nfds`: 用来指定第一个参数数组元素个数。
* 3.`timeout`: 指定等待的毫秒数，无论`I/O`是否准备好，poll() 都会返回。当等待时间为 0 时，poll() 函数立即返回，为 -1 则使 poll() 一直阻塞直到一个指定事件发生。

**返回值**

* `成功时`，`poll()`返回结构体中revents 域不为 0 的文件描述符个数；如果在超时前没有任何事件发生，poll()返回 0；
* `失败时`，`poll()`返回 -1，并设置 errno 为下列值之一：
	* `EBADF`：一个或多个结构体中指定的文件描述符无效。
	* `EFAULT`：fds 指针指向的地址超出进程的地址空间。
	* `EINTR`：请求的事件之前产生一个信号，调用可以重新发起。
	* `EINVAL`：nfds 参数超出 PLIMIT_NOFILE值。
	* `ENOMEM`：可用内存不足，无法完成请求。

## 3.2 示例

```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <sys/types.h>
#include <stropts.h>
#include <sys/poll.h>
#include <sys/stropts.h>
#include <string.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <poll.h>

#define BUFSIZE 1024
int main(int argc,char* argv[]){
	char buf[BUFSIZE];
	int bytes,i=0,nummonitor=0,numready,errno;
	char * str;
	struct pollfd *pollfd;
	if(argc!=3){
		fprintf(stderr,"useage:t");
		exit(1);
	}
	//struct pollfd分配空间
	if((pollfd=(struct pollfd*)calloc(2,sizeof(struct pollfd)))==NULL){
		exit(-1);
	}
	//初始化struct pollfd结构
	for(i;i<2;i++){
		 str = (char*)malloc(14*sizeof(char));        
        memcpy(str,"/root/pro/",14);
        strcat(str,argv[i+1]);
        printf("str=%s\n",str);
        (pollfd+i)->fd = open(str,O_RDONLY);
        if((pollfd+i)->fd >= 0)
        fprintf(stderr, "open (pollfd+%d)->fd:%s\n", i, argv[i+1]);
        nummonitor++;
        (pollfd+i)->events = POLLIN;
	}
	printf("nummonitor=%d\n",nummonitor);
	while(nummonitor > 0){
        numready = poll(pollfd, 2, -1);
        //被信号中断，继续等待
        if ((numready == -1) && (errno == EINTR))
            continue;        
        else if (numready == -1)//poll真正错误，退出
            break; 
        printf("numready=%d\n",numready);
        for (i=0;nummonitor>0 && numready>0; i++)
        {
        //返回内核做判断
            if((pollfd+i)->revents & POLLIN)
            {
                bytes = read(pollfd[i].fd, buf, BUFSIZE);
                numready--;
                printf("pollfd[%d]->fd read buf:\n%s \n", i, buf);
                nummonitor--;
            }
        }
    }
    for(i=0; i<nummonitor; i++)
        close(pollfd[i].fd);
    free(pollfd);
    return 0;
}
```

## 3.3 缺点
`poll() `的实现和`select()`非常相似，只是描述`fd集合`的方式不同，`poll()`使用 `pollfd`结构而不是`select()`的`fd_set` 结构，其他的都差不多。

# 4.epoll
## 4.1 api说明

epoll是在 2.6 内核中提出的，是之前的 `select()`和`poll()`的增强版本。相对于 `select()`和`poll()`来说，epoll更加灵活，**没有描述符限制**。epoll使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在**用户空间和内核空间的copy只需一次**。

epoll 操作过程需要三个接口，分别如下：
```
#include <sys/epoll.h>
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

### `int epoll_create(int size)`

该函数生成一个 epoll 专用的文件描述符（创建一个 epoll 的句柄）。

* size: 用来告诉内核这个监听的数目一共有多大，参数 size 并不是限制了 epoll 所能监听的描述符最大个数，只是对内核初始分配内部数据结构的一个建议。自从 linux 2.6.8 之后，**size 参数是被忽略的**，也就是说可以填只有大于 0 的任意值。需要注意的是，当创建好 epoll 句柄后，它就是会占用一个 fd 值，在 linux 下如果查看 `/proc/` 进程 `id/fd/`，是能够看到这个 fd 的，所以在使用完 epoll 后，必须调用`close()`关闭，否则可能导致fd被耗尽。

返回值

* `成功`：epoll 专用的文件描述符
* `失败`：-1

### `int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);`

epoll的事件注册函数，它不同于`select()` 是在监听事件时告诉内核要监听什么类型的事件，而是在这里先注册要监听的事件类型。

* `epfd`: epoll专用的文件描述符，`epoll_create()`的返回值
* `op`: 表示动作，用三个宏来表示：
	* `EPOLL_CTL_ADD`：注册新的 fd 到 epfd 中；
	* `EPOLL_CTL_MOD`：修改已经注册的fd的监听事件；
	* `EPOLL_CTL_DEL`：从 epfd 中删除一个 fd；
* `fd`: 需要监听的文件描述符
* `event`: 告诉内核要监听什么事件，struct epoll_event 结构如下：

```
//保存触发事件某个文件描述符相关数据
typdef union epoll_data{
	void*ptr;
	int  fd;
	__uint32_t u32;
	__uint64_t u64;
}epoll_data_t;
//感兴趣的事件和被触发的事件
struct epoll_event{
	__uint32_t events;
	epoll_data_t data;
};
```

events 可以是以下几个宏的集合：
* `EPOLLIN`：表示对应的文件描述符可以读（包括对端 SOCKET 正常关闭）；
* `EPOLLOUT`：表示对应的文件描述符可以写；
* `EPOLLPRI`：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
* `EPOLLERR`：表示对应的文件描述符发生错误；
* `EPOLLHUP`：表示对应的文件描述符被挂断；
* `EPOLLET`：将 EPOLL 设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
* `EPOLLONESHOT`：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个 socket 的话，需要再次把这个 socket 加入到 EPOLL 队列里

返回值：

* `成功`：0
* `失败`：-1

### `int epoll_wait( int epfd, struct epoll_event * events, int maxevents, int timeout );`

等待事件的产生，收集在epoll监控的事件中已经发送的事件，类似于`select()`调用。

* `epfd`:epoll 专用的文件描述符，epoll_create()的返回值
* `events`: 分配好的epoll_event结构体数组，epoll 将会把发生的事件赋值到events 数组中（events 不可以是空指针，内核只负责把数据复制到这个 events 数组中，不会去帮助我们在用户态中分配内存）。
* `maxevents`: maxevents告知内核这个 events 有多大 。
* `timeout`: 超时时间，单位为毫秒，为 -1 时，函数为阻塞

返回值：

* `成功`：返回需要处理的事件数目，如返回 0 表示已超时。
* `失败`：-1

epoll对文件描述符的操作有两种模式：**LT（level trigger)**和**ET（edge trigger)**。LT模式是默认模式，LT模式与 ET 模式的区别如下：

* `LT 模式`：当 epoll_wait 检测到描述符事件发生并将此事件通知应用程序，应用程序可以不立即处理该事件。下次调用 epoll_wait 时，会再次响应应用程序并通知此事件。
* `ET 模式`：当 epoll_wait 检测到描述符事件发生并将此事件通知应用程序，应用程序必须立即处理该事件。如果不处理，下次调用 epoll_wait 时，不会再次响应应用程序并通知此事件。

`ET 模式`在很大程度上减少了 epoll 事件被**重复触发**的次数，因此效率要比`LT 模式高`。epoll 工作在 ET 模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。

## 4.2 示例

```c
#define MAX_EVENTS 10
struct epoll_events ev,events[MAX_EVENTS];
int listen_sock,conn_sock,nfds,epollfd;

//建立socket监听
//socket(),bind(),listen()
epollfd=epoll_create(10);
if(epoll_fd==-1){
	perror("epll_create");
	exit(EXIT_FAILURE);
}
ev.events=EPOLLIN;
ev.data.fd=listen_sock;
//注册描述符添加自己数据
if(epoll_ctl(epollfd,EPOLL_CTL_ADD,listen_sock,&ev)==-1){
	perror("epoll_ctl:listen_sock");
}
for(;;){
	nfds=epoll_wait(epollfd,events,MAX_EVENTS,-1);
	if(nfds==-1){
		perror("epoll_wait");
		exit(EXIT_FAILURE);
	}
	for(n=0;n<nfds;++n){
		if(events[n].data.fd==listen_sock){
			conn_sock=accept(listen_sock,(struct sockaddr *) &local, &addrlen);
			if(conn_sock==-1){
				(struct sockaddr *) &local, &addrlen
			}
			setnonblocking(conn_sock);
			ev.events=EPOLLIN|EPOLLET;
			ev.data.fd=conn_sock;
			if(epoll_ctl(epollfd,EPOLL_CTL_ADD,conn_sock,&ev)==-1){
				 perror("epoll_ctl: conn_sock");
                 exit(EXIT_FAILURE);
			}
		}else{
			  do_use_fd(events[n].data.fd);
		}
	}
}
```

在`select/poll`中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而`epoll()`事先通过`epoll_ctl()`来注册一个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似`callback`的回调机制(软件中断)，迅速激活这个文件描述符，当进程调用`epoll_wait()`时便得到通知。

## 4.3 优点

1）监视的描述符数量不受限制，它所支持的FD 上限是最大可以打开文件的数目，这个数字一般远大于2048,举个例子,在 1GB 内存的机器上大约是 10 万左右，具体数目可以`cat /proc/sys/fs/file-max`察看,一般来说这个数目和系统内存关系很大。`select()`的最大缺点就是进程打开的 fd 是有数量限制的。这对于连接数量比较大的服务器来说根本不能满足。虽然也可以选择多进程的解决方案( Apache 就是这样实现的)，不过虽然 Linux 上面创建进程的代价比较小，但仍旧是不可忽视的，加上进程间数据同步远比不上线程间同步的高效，所以也不是一种完美的方案。

2）I/O 的效率不会随着监视 fd 的数量的增长而下降。`select()`，`poll()` 实现需要自己不断轮询所有 fd 集合，直到设备就绪，期间可能要睡眠和唤醒多次交替。而 epoll 其实也需要调用`epoll_wait()`不断轮询就绪链表，期间也可能多次睡眠和唤醒交替，但是它是设备就绪时，调用回调函数，把就绪 fd 放入就绪链表中，并唤醒在`epoll_wait()`中进入睡眠的进程。虽然都要睡眠和交替，但是`select()`和 `poll()`在“醒着”的时候要遍历整个 fd 集合，而 epoll 在“醒着”的时候只要判断一下就绪链表是否为空就行了，这节省了大量的 CPU 时间。这就是回调机制带来的性能提升。
3）`select()`，`poll()`每次调用都要把 fd 集合从用户态往内核态拷贝一次，而 epoll 只要一次拷贝，这也能节省不少的开销。



