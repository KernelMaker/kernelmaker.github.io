---
layout: post
title: pika导致redis客户端使用pipeline卡死的问题
---

有人在github提issue反馈redis客户端使用pipeline往pika灌数据，当数据量稍大一些，客户端会卡死，我找了下原因，这里总结一下吧


## Pipeline

什么是Pipeline？假设现在有3条命令要执行，正常的请求方式是：客户端发送第一条命令给服务端，服务端处理，返回结果给客户端，客户端发送第二条命令给服务端......以此类推，可以看到这三条命令从开始到结束，伴随着3次网络的RTT（Round Trip Time），试想如果将这三条命令打包，一起发给服务端，全发出去之后再接收服务端对这三条命令执行结果的打包返回，这样RTT从3次减少成1次，减少了整体命令响应延迟，提高效率

Pipeline的支持主要是在客户端这里，hiredis就很好地支持了Pipeline，其实也不难，通过redisAppendCommand来追加命令到buffer，这个时候没有真正的发送给服务端，直到调用redisGetReply的时候，才真正发送给服务端并且等待读取服务端的结果

## 原因定位

我用hiredis写了一个简单的pipeline测试程序，就是打包1000个Ping命令发送给pika，并且期待接收服务端返回的1000个Pong，一跑，没问题。。。把Ping的个数调大到30万，果不其然客户端卡死了，通过htop看了下pika的状态，worker线程还是在正常的空刷`epoll_wait`，并且可以处理别的客户端的请求，看来pika这里没有问题，再看下客户端，发现客户端卡死到了write调用上，证明此时客户端的命令发送不出去，“命令少没问题，命令多会卡死到write调用上”，猜测是客户端这里的TCP发送缓冲区写满了，为什么会满呢？难道Pika那端的TCP接收缓冲区也满了导致客户端TCP窗口中的报文迟迟收不到对端确认接收的ACK所以撑爆了？不会啊，如果pika端TCP接收缓冲区有消息的话在刚才htop里对应的worker线程不可能一直在空刷`epoll_wait`，至少有个read调用什么的，推测到这里感觉有点走不下去了。反正目前结论就是有可能pika端TCP接收缓冲区和客户端TCP发送缓冲区都满了

硬着头皮接着瞎猜，因为pipeline下，客户端在没有发送完所有命令之前，是不可能去接收服务端返回的结果，所以在上面的情况下（客户端卡死在write调用上证明命令还没全部发送完）pika的TCP发送缓冲区也有可能写满，嗯，只能靠试试了，我改了下pika的代码，调大了TCP发送缓冲区到1M，居然没问题了。。。难道redis调整了这个缓冲区大小了吗？在redis代码里搜了一下`SO_SNDBUF`，没看到它有调整，应该用的是默认值，那为什么同样使用默认值的pika就不行了呢？

该不会pika事件处理那里写的有问题？看下代码

```cpp
if (pfe->mask_ & EPOLLIN) { //读事件
  in_conn = static_cast<Conn *>(iter->second);
  ReadStatus getRes = in_conn->GetRequest();
  in_conn->set_last_interaction(now);
  log_info("now: %d, %d", now.tv_sec, now.tv_usec);
  log_info("in_conn->is_reply() %d", in_conn->is_reply());
  if (getRes != kReadAll && getRes != kReadHalf) {
    // kReadError kReadClose kFullError kParseError
    should_close = 1;
  } else if (in_conn->is_reply()) {
    pink_epoll_->PinkModEvent(pfe->fd_, 0, EPOLLOUT); // <-问题在这里
  } else {
    continue;
  }
}
if (pfe->mask_ & EPOLLOUT) { //写事件
  in_conn = static_cast<Conn *>(iter->second);
  log_info("in work thead SendReply before");
  WriteStatus write_status = in_conn->SendReply();
  log_info("in work thead SendReply after");
  if (write_status == kWriteAll) {
    in_conn->set_is_reply(false);
    pink_epoll_->PinkModEvent(pfe->fd_, 0, EPOLLIN);
  } else if (write_status == kWriteHalf) {
    continue;
  } else if (write_status == kWriteError) {
    should_close = 1;
  }
}
```

原来真有问题，之前认为pika一定是开始只监听fd读事件，当解析出来一条命令后执行并生成返回结果然后开始只监听fd写事件，直到结果发送给客户端后，再重新监听fd读事件。这样的逻辑在常规的客户端读写都不会有问题，不过在pipeline下，由于客户端迟迟不读服务端的返回结果（直到发送完全部命令），很容易导致pika的TCP发送缓冲区变满，按照上面的逻辑，此时pika不认为把返回结果成功发送给了客户端，所以不断地监听只fd写事件，而这个时候该fd不可写（缓冲区满），所以迟迟来不了fd可写事件，然后此时又不关注fd可读事件，不能再读取客户端发来的后续命令，导致客户端的TCP发送缓冲区也满了，产生死锁，所以卡死在write调用上。将有问题的那一行代码改成：

```cpp
pink_epoll_->PinkModEvent(pfe->fd_, EPOLLIN, EPOLLOUT);
```
表示持续关注fd可读事件，问题迎刃而解。其实redis也是这么做的，一直关注fd可读事件，只在需要的时候打开fd写事件的监听。

## 总结
1. 对微不足道的细节问题也要足够关心，尤其网络和事件这里
2. pika开源的收获还挺多的，不少同学反映了很多在公司内部使用pika没有遇见的问题，帮助pika更好的完善

PS: 今天是我生日啊，纪念一下，哈哈哈^^
