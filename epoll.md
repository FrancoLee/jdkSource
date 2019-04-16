## epoll

---

源码参考：`tool/perf/bench/epoll-ctl.c,tool/perf/bench/epoll-wait.c`(`mark`下，还没分析过源码)

[参考1](https://blog.csdn.net/u011671986/article/details/79449853)

[参考2](http://blog.chinaunix.net/uid-28541347-id-4273856.html)

![img](http://blog.chinaunix.net/attachment/201405/26/28541347_140111501437dD.jpg)

过程：

 	(1) epoll_wait调用ep_poll，当rdlist为空（无就绪fd）时挂起当前进程，知道rdlist不空时进程才被唤醒。 

 	(2) 文件fd状态改变（buffer由不可读变为可读或由不可写变为可写），导致相应fd上的回调函数ep_poll_callback()被调用。 

 	(3) ep_poll_callback将相应fd对应epitem加入rdlist，导致rdlist不空，进程被唤醒，epoll_wait得以继续执行。 

 	(4) ep_events_transfer函数将rdlist中的epitem拷贝到txlist中，并将rdlist清空。 

 	(5) ep_send_events函数（很关键），它扫描txlist中的每个epitem，调用其关联fd对用的poll方法（图中蓝线）。此时对poll的调用仅仅是取得fd上较新的events（防止之前events被更新），之后将取得的events和相应的fd发送到用户空间（封装在struct epoll_event，从epoll_wait返回）。之后如果这个epitem对应的fd是LT模式监听且取得的events是用户所关心的，则将其重新加入回rdlist（图中蓝线），否则（ET模式）不在加入rdlist。

与之前的`select`代理相比，`epoll`效率更高。`select`当`fc`状态发生改变后去轮询所有受监控的`链接`,而`epoll`是通过回调函数`ep_poll_callback`将`fd`(当`fd`状态改变)存入`read list`,使得`read lsit`不为空，`epoll`进程被唤醒。`epoll`通过遍历`read list`来处理`fd`状态改变。（`epoll`使用`rbTree`来存储所有的`socket`，`fd`是对应`app`通信时使用的`buffer`的句柄）