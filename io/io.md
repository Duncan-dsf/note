- cbio哪些操作会阻塞 而nio如何优化 
  - bio阻塞：（apue P388 14.2）
    - 如果某些类型文件（读管道、终端设备和网络设备）的数据不存在，则读操作可能会使调用者永远阻塞（可能读的时候，网络数据还没有到来）
    - 如果某些数据不能被相同类型文件立即接受，则写操作可能会使调用者永远阻塞（可能写的时候，网络设备中的数据未完全写出去，需要排队）
    - 在某种条件发生之前打开某些类型文件，可能会发生阻塞（例如打开一个Socket，需要等待客户端的连接）
    - 磁盘io虽然也会阻塞调用者，但阻塞时间一般不会太长，也不可能永远阻塞。
  - nio如何避免：多路复用
    - select：apue 14.4.1
      - 函数声明：![](pic/select_call.png)
      - 传给select的参数：
        - 我们关心的描述符及其条件（readFds，writeFds, exceptFds异常）
        - 愿意等待时间
      - 函数返回时，内核告诉我们：
        - 已准备好的描述符总量 （返回值为int）
        - 读、写、异常三个条件的每一个，哪些描述符已准备好（入参fd_set是一个很大的位图，index为x对应值代表文件描述符为index是否准备好，准备好的话，值为1；如此，想要取到所有已准备好的数据，需要遍历位图，假如遍历整个位图太傻了，而入参第一位就是代表者最大的fd，只需遍历max_fd之前的fd即可）
      - pselect 最后一个参数的类型不同，假如当前系统支持纳米级别的时间精确度，pselect就可以支持（而select只支持微秒）
    - poll：apue 14.4.2
      - 函数声明：![](pic/poll_call.png)
      - 参数：
        - fdArray ：pollFd中三个字段：fd、events（关心的事件，由用户提供）、revents（发生的事件，由内核修改），events和revents都只能代表一个事件；**然而很奇怪，revents返回fd的事件，那么pollFd定义events干什么？假如对同一个fd注册多个事件该怎么办？？**
        - nfds 数组大小
        - timeout等待时间，单位毫秒
    - epoll 深入实现： http://blog.chinaunix.net/uid-28541347-id-4238524.html http://blog.chinaunix.net/uid-28541347-id-4273856.html 简单易懂：https://blog.csdn.net/shenya1314/article/details/73691088
      - 使用方法：
        1. epoll_create()创建epoll句柄，epoll文件中有eventpoll结构，之中有一个红黑树，红黑树中每个节点都是fd及其事件
           1. list_head **rdlist** 就绪队列
           2. rb_root **rbr**关系的事件通过ctl注册到rb树上
        2. epoll_ctl(epfd, op, fd, event) 将fd及对应关心的event增删改（由op决定）到epfd的eventpoll的红黑树上**rbr**，红黑树每个节点为epitem
           1. epitem的结构：
              1. fd
              2. 关心当前fd的什么事件
              3. next指针
              4. next组成的链表的head
              5. 当前节点所属的eventpoll
        3. epoll_wait(epfd, *events, maxevents, timeout)，
           1. 当rd树为空时挂起当前进程，直到红黑树不空时，唤醒进程。
           2. fd状态改变（buffer由不可读/写 变 可读/写），内核调用fd上的回调函数ep_poll_callback()
           3. ep_poll_callback将对应的fd的epitem加入rdList(就绪队列)，导致rdlist不空，进程唤醒，epoll_wait继续执行
           4. ep_events_transfer将rdList的epitem拷贝到txList，并将rdlist清空
           5. pe_send_events，扫描txlist中每个epitem，取出fd及其发生的events，并发送到用户空间(存到入参events数组中)。如果events是用户关心的，LT模式下将epitem重新加入relist，ET模式下不加入。