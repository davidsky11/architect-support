# Linux中的IO模型

## IO模型

Linux系统IO分为内核准备数据和将数据从内核拷贝到用户空间两个阶段。

![数据拷贝](https://github.com/davidsky11/architect-support/blob/master/03%20%E9%AB%98%E7%BA%A7%E7%AF%87/0305%20%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E7%9F%A5%E8%AF%86/IO%E6%A8%A1%E5%9E%8B/00_data_copy.png)

## 用户空间、内核空间

>对于32位操作系统而言，它的寻址空间（虚拟存储空间）为4G（2的32次方）。操作系统的核心是内核，独立于普通的应用程序，可以访问受保护的内存空间，也有访问底层硬件设备的所有权限。为了保证用户进程不能直接操作内核，保证内核的安全，操作系统将虚拟空间划分为两个部分，一个部分是内核空间，一部分是用户空间。

    Window 32位操作系统，默认的用户空间：内核空间的比例是1：1，而在32位Linux系统中的默认比例是3：1（3G用户空间，1G内核空间）。

## 进程切换

进程切换的过程，会经过下面这些变化：
 1. 保持处理机上下文，包括程序计数器和其他寄存器；  
 2. 更新PCB信息；  
 3. 将进程的PCB移入相应的队列，如就绪、在某事件阻塞等队列；  
 4. 选择另外一个进程执行，并更新PCB；  
 5. 更新内存管理的数据结构；  
 6. 恢复处理机上下文。  

## 缓存IO

在Linux的缓存IO机制中，操作系统会将IO的数据缓存在文件系统的页缓存（page cache）。也就是说，数据会先被拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓存区拷贝到应用程序的地址空间中。

    缺点：需要在应用程序地址空间和内核进行多次拷贝，拷贝动作带来的CPU以及内存开销是非常大的。

## 同步、异步、阻塞、非阻塞

 - 同步与异步：描述的是用户线程与内核的交互方式，同步指用户线程发起IO请求后需要等待或者轮询内核IO操作完成后才能继续执行；而异步是指用户线程发起IO请求后仍然继续执行，当内核IO操作完成后会通知用户线程，或者调用用户线程注册的回调函数。
 - 阻塞与非阻塞：描述是用户线程调用内核IO操作的方式，阻塞是指IO操作需要彻底完成后才返回到用户空间；而非阻塞是指IO操作被调用后立即返回给用户一个状态值，无需等到IO操作彻底完成。

## Linux IO模型

### 阻塞IO模型

进程会一直阻塞，直到数据拷贝完成。

![阻塞IO模型](https://github.com/davidsky11/architect-support/blob/master/03%20%E9%AB%98%E7%BA%A7%E7%AF%87/0305%20%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E7%9F%A5%E8%AF%86/IO%E6%A8%A1%E5%9E%8B/01_%E9%98%BB%E5%A1%9EIO%E6%A8%A1%E5%9E%8B.jpg)

### 非阻塞IO模型

非阻塞IO通过进程反复调用IO函数（多次系统调用，并马上返回）；在数据拷贝的过程中，进程是阻塞的

![非阻塞IO](https://github.com/davidsky11/architect-support/blob/master/03%20%E9%AB%98%E7%BA%A7%E7%AF%87/0305%20%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E7%9F%A5%E8%AF%86/IO%E6%A8%A1%E5%9E%8B/02_%E9%9D%9E%E9%98%BB%E5%A1%9EIO%E6%A8%A1%E5%9E%8B.jpg)

### IO多路复用模型

主要是select和epoll；对一个IO端口，两次调用，两次返回，比阻塞IO并没有什么优越性；关键是能实现同时对多个IO端口进行监听；
    
![IO多路复用模型](https://github.com/davidsky11/architect-support/blob/master/03%20%E9%AB%98%E7%BA%A7%E7%AF%87/0305%20%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E7%9F%A5%E8%AF%86/IO%E6%A8%A1%E5%9E%8B/03_IO%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8%E6%A8%A1%E5%9E%8B.jpg)

### 信号驱动IO

两次调用，两次返回

![信号驱动IO](https://github.com/davidsky11/architect-support/blob/master/03%20%E9%AB%98%E7%BA%A7%E7%AF%87/0305%20%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E7%9F%A5%E8%AF%86/IO%E6%A8%A1%E5%9E%8B/04_%E4%BF%A1%E5%8F%B7%E9%A9%B1%E5%8A%A8IO%E6%A8%A1%E5%9E%8B.jpg)

### 异步IO模型

数据拷贝的时候进程无需阻塞

![异步IO模型](https://github.com/davidsky11/architect-support/blob/master/03%20%E9%AB%98%E7%BA%A7%E7%AF%87/0305%20%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E7%9F%A5%E8%AF%86/IO%E6%A8%A1%E5%9E%8B/05_%E5%BC%82%E6%AD%A5IO%E6%A8%A1%E5%9E%8B.jpg)

## 五个IO模型的比较

![模型比较](https://github.com/davidsky11/architect-support/blob/master/03%20%E9%AB%98%E7%BA%A7%E7%AF%87/0305%20%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E7%9F%A5%E8%AF%86/IO%E6%A8%A1%E5%9E%8B/00_%E4%BA%94%E4%B8%AAIO%E6%A8%A1%E5%9E%8B%E6%AF%94%E8%BE%83.jpg)


----------


## IO多路复用 select、poll、epoll
    
IO多路复用是通过一种机制，一个进程可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通过程序进行相应的读写操作。

select，poll，epoll本质上都是同步IO，因为它们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而异步IO则无需自己负责进行读写，异步IO的实现会负责把数据从内核拷贝到用户空间。

### select 轮询方式

`int select (int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);`

select 函数监视的文件描述符分3类，分别是writefds、readfds、和exceptfds。调用后select函数会阻塞，直到有描述副就绪（有数据 可读、可写、或者有except），或者超时（timeout指定等待时间，如果立即返回设为null即可），函数返回。当select函数返回后，可以通过遍历fdset，来找到就绪的描述符。
    
 **缺点**：单个进程能够监视的文件描述符的数量存在最大限制，在Linux上一般为1024。

### poll

`int poll (struct pollfd *fds, unsigned int nfds, int timeout);`

不同与select使用三个位图来表示三个fdset的方式，poll使用一个 pollfd的指针实现。

```
struct pollfd {
    int fd; /* file descriptor */
    short events; /* requested events to watch */
    short revents; /* returned events witnessed */
};
```

pollfd结构包含了要监视的event和发生的event，不再使用select“参数-值”传递的方式。同时，pollfd并没有最大数量限制（但是数量过大后性能也是会下降）。 和select函数一样，poll返回后，需要轮询pollfd来获取就绪的描述符。
    
### epoll

在2.6内核中提出的，是之前的select和poll的增强版本。相对于select和poll来说，epoll更加灵活，没有描述符限制。epoll使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的copy只需一次。

#### epoll操作过程

```
int epoll_create(int size)；//创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)； //对指定描述符fd执行op操作
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout); //等待epfd上的io事件，最多返回maxevents个事件
```

#### 工作模式
    
epoll对文件描述符的操作有两种模式：LT（level trigger）和ET（edge trigger）。LT模式是默认模式，LT模式与ET模式的区别如下：  

LT模式：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序可以不立即处理该事件。下次调用epoll_wait时，会再次响应应用程序并通知此事件。  

ET模式：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序必须立即处理该事件。如果不处理，下次调用epoll_wait时，不会再次响应应用程序并通知此事件。  

 **LT模式**

LT(level triggered)是缺省的工作方式，并且同时支持block和no-block socket.在这种做法中，内核告诉你一个文件描述符是否就绪了，然后你可以对这个就绪的fd进行IO操作。如果你不作任何操作，内核还是会继续通知你的。

 **ET模式**

ET(edge-triggered)是高速工作方式，只支持no-block socket。在这种模式下，当描述符从未就绪变为就绪时，内核通过epoll告诉你。然后它会假设你知道文件描述符已经就绪，并且不会再为那个文件描述符发送更多的就绪通知，直到你做了某些操作导致那个文件描述符不再为就绪状态了（比如，你在发送，接收或者接收请求，或者发送接收的数据少于一定量时导致了一个EWOULDBLOCK错误）。但是请注意，如果一直不对这个fd作IO操作(从而导致它再次变成未就绪)，内核不会发送更多的通知(only once)

ET模式在很大程度上减少了epoll事件被重复触发的次数，因此效率要比LT模式高。epoll工作在ET模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。  

#### epoll总结

在 select/poll中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而epoll事先通过epoll_ctl()来注册一个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait() 时便得到通知。(此处去掉了遍历文件描述符，而是通过监听回调的的机制。这正是epoll的魅力所在。  

**特点：**  
 1. epoll和select、poll的调用接口上的不同  
 2. 使用mmap加速内核与用户空间的消息传递  
 3. 调用后无需轮询判断描述符事件是否就绪  
 4. 监视描述符没有个数上限  
 5. IO效率不随FD数目增加而线性下降  

**优点：**

 1. 监视的描述符数量不受限制，它所支持的FD上限是最大可以打开文件的数目。  
    如果没有大量的idle -connection或者dead-connection，epoll的效率并不会比select/poll高很多，但是当遇到大量的idle- connection，就会发现epoll的效率大大高于select/poll。  
 2. select，poll实现需要自己不断轮询所有fd集合，直到设备就绪，期间可能要睡眠和唤醒多次交替。而epoll其实也需要调用epoll_wait不断轮询就绪链表，期间也可能多次睡眠和唤醒交替，但是它是设备就绪时，调用回调函数，把就绪fd放入就绪链表中，并唤醒在epoll_wait中进入睡眠的进程。虽然都要睡眠和交替，但是select和poll在“醒着”的时候要遍历整个fd集合，而epoll在“醒着”的时候只要判断一下就绪链表是否为空就行了，这节省了大量的CPU时间。这就是回调机制带来的性能提升。  
 3. select，poll每次调用都要把fd集合从用户态往内核态拷贝一次，并且要把current往设备等待队列中挂一次，而epoll只要一次拷贝，而且把current往等待队列上挂也只挂一次（在epoll_wait的开始，注意这里的等待队列并不是设备等待队列，只是一个epoll内部定义的等待队列）。这也能节省不少的开销。  

    ![select_poll_epoll](https://github.com/davidsky11/architect-support/blob/master/03%20%E9%AB%98%E7%BA%A7%E7%AF%87/0305%20%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E7%9F%A5%E8%AF%86/IO%E6%A8%A1%E5%9E%8B/00_select_poll_epoll.png)

