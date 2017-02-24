---
title: 在Nginx的worker进程fork之后
date: 2014-11-15 02:22:45
tags:
---
项目中遇到一个问题：CPU占用在某种条件下会到接近100%，当然肯定不是业务满载，查吧。

先看日志中的关键部分。
出问题的fd销毁时刻：
2014/10/31 21:12:10 [debug] 23574#23577: epoll: fd:28 ev:0019 d:00007F16ED6981C0
2014/10/31 21:12:10 [debug] 23574#23577: epoll_wait() error on fd:28 ev:0019
2014/10/31 21:12:10 [debug] 23574#23577: *2 recv: fd:28 -1 of 146
2014/10/31 21:12:10 [info ] 23574#23577: *2 recv() failed (104: Connection reset by peer), client: 192.168.147.75, server: 0.0.0.0:1935
2014/10/31 21:12:10 [debug] 23574#23577: *2 ngx_rtmp_put_session
2014/10/31 21:12:10 [debug] 23574#23577: *2 finalize session
该fd出问题时刻：
2014/11/04 09:47:50 [debug] 23574#23577: epoll: stale event 00007F16ED6981C0
2014/11/04 09:47:50 [debug] 23574#23577: process events delta: 0
2014/11/04 09:47:50 [debug] 23574#23577: accept mutex lock failed
2014/11/04 09:47:50 [debug] 23574#23577: epoll timer: 10
2014/11/04 09:47:50 [debug] 23574#23577: epoll: stale event 00007F16ED6981C0
2014/11/04 09:47:50 [debug] 23574#23577: process events delta: 0
2014/11/04 09:47:50 [debug] 23574#23577: accept mutex lock failed
2014/11/04 09:47:50 [debug] 23574#23577: epoll timer: 10
就是这个stale event导致epoll_wait马上返回，从而占用这么多CPU。

为了验证该问题，先看一个测试demo：
``` C
void fork_server_routine()
{
    int sock = socket(PF_INET, SOCK_STREAM, 0);
    if (sock &lt; 0) return;
    struct sockaddr_in addr = {0};
    addr.sin_family = PF_INET;
    addr.sin_port = htons(9999);
    int rc = 1;
    rc = setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, &amp;rc , sizeof(int));
    if (rc &lt; 0) {
        perror("setsockopt");
        return;
    }
    rc = bind(sock, (struct sockaddr *)&amp;addr, sizeof(addr));
    if (rc &lt; 0) {
        perror("bind");
        return;
    }
    rc = listen(sock, 32);
    if (rc &lt; 0) return;


    int ep = epoll_create(1);
    struct epoll_event events[32];
    events[0].data.fd = sock;
    events[0].events = EPOLLIN | EPOLLET;
    rc = epoll_ctl(ep, EPOLL_CTL_ADD, sock, &amp;events[0]);
    if (rc &lt; 0) return;

    rc = epoll_wait(ep, events, 32, -1);
    if (rc &lt; 0) return;
    printf("epoll_wait: %d\n", rc);

    socklen_t sal = sizeof(addr);
    int client = accept(sock, (struct sockaddr *)&amp;addr, &amp;sal);
    if (client &lt; 0) return;
    printf("accepted: %d\n", client);
    events[0].data.fd = client;
    events[0].events = EPOLLIN | EPOLLET;
    rc = epoll_ctl(ep, EPOLL_CTL_ADD, client, &amp;events[0]);
    if (rc &lt; 0) return;

    rc = fork();
    if (rc &lt; 0) return;
    if (rc == 0) {
        printf("child start\n");
        for (;;) {
            sleep(3);
        }
        printf("child quit\n");
    } else {
        printf("parent start\n");

        memset(events, 0, sizeof(events));
        rc = epoll_wait(ep, events, 32, -1);
        if (rc &lt; 0) {
            perror("parent: epoll_wait");
            return;
        }
        printf("parent: epoll1 return %d, fd = %d\n", rc, events[0].data.fd);
        memset(events, 0, sizeof(events));
        printf("close %d\n", client);
        epoll_ctl(ep, EPOLL_CTL_DEL, client, &amp;events[0]); // 注意这一行
        close(client);

        memset(events, 0, sizeof(events));
        rc = epoll_wait(ep, events, 32, -1);
        if (rc &lt; 0) {
            perror("parent: epoll_wait");
            return;
        }
        printf("parent: epoll2 return %d, fd = %d\n", rc, events[0].data.fd);
    }
}

void fork_client_routine()
{
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock &lt; 0) return;
    struct sockaddr_in addr = {0};
    addr.sin_family = PF_INET;
    addr.sin_port = htons(9999);
    addr.sin_addr.s_addr = inet_addr("127.0.0.1");
    int rc = connect(sock, (struct sockaddr *)&amp;addr, sizeof(addr));
    if (rc &lt; 0) return;
    for (;;) {
        printf("connected, press any\n");
        getchar();
        send(sock, &amp;addr, 1, 0);
    }
}

int
main (int argc, char **argv)
{
    if (argc &lt; 2) {
        printf("./t1 c or s\n");
        return 0;
    }
    if ('c' == argv[1][0]) {
        fork_client_routine();
    } else if ('s' == argv[1][0]) {
        fork_server_routine();
    } else {
        printf("./t1 c or s\n");
        return 0;
    }

    return 0;
}
```

当server-parent已经接受1个客户端后，fork出server-child，则server-child对client fd增加了1个引用（实际环境是父进程fork后马上exec，但是该client fd没有指定close-on-exec，所以情况和本测试demo相同）。这时如果server-parent close这个client fd，client进程是不会知道对端已经关闭的，因为在kernel里client fd的引用还没减到0，所以client进程照样可以正常发数据。
问题就出在server-parent close这个client fd上：server-parent到底需不需要用EPOLL_CTL_DEL把client fd删除呢？当然close肯定是要调用的。

下面是Nginx代码截取：
<a href="http://trac.nginx.org/nginx/browser/nginx/src/event/modules/ngx_epoll_module.c#L441" target="_blank">http://trac.nginx.org/nginx/browser/nginx/src/event/modules/ngx_epoll_module.c#L441</a>
``` C
static ngx_int_t
ngx_epoll_del_event(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags)
{
 int op;
 uint32_t prev;
 ngx_event_t *e;
 ngx_connection_t *c;
 struct epoll_event ee;

 /*
 * when the file descriptor is closed, the epoll automatically deletes
 * it from its queue, so we do not need to delete explicitly the event
 * before the closing the file descriptor
 */

 if (flags & NGX_CLOSE_EVENT) {
     ev->active = 0;
     return NGX_OK;
 }
```

可以看到nginx作者还专门加了注释说明这个情况，他的意思是：如果flags指明了NGX_CLOSE_EVENT，就不需要再EPOLL_CTL_DEL了，因为epoll会自动从队列中删掉他。
而NGX_CLOSE_EVENT就是为了这个功能而设定的：
``` C
/*
 * The event filter is deleted just before the closing file.
 * Has no meaning for select and poll.
 * kqueue, epoll, rtsig, eventport: allows to avoid explicit delete,
 * because filter automatically is deleted
 * on file close,
 *
 * /dev/poll: we need to flush POLLREMOVE event
 * before closing file.
 */
 #define NGX_CLOSE_EVENT 1
```

那下面就到了调用ngx_epoll_del_event的地方了：
<a href="http://trac.nginx.org/nginx/browser/nginx/src/core/ngx_connection.c#L911" target="_blank">http://trac.nginx.org/nginx/browser/nginx/src/core/ngx_connection.c#L911</a>
``` C
void
ngx_close_connection(ngx_connection_t *c)
{
 ngx_err_t err;
 ngx_uint_t log_error, level;
 ngx_socket_t fd;

 if (c->fd == (ngx_socket_t) -1) {
     ngx_log_error(NGX_LOG_ALERT, c-&gt;log, 0, "connection already closed");
     return;
 }

 if (c->read->timer_set) {
     ngx_del_timer(c->read);
 }

 if (c->write->timer_set) {
     ngx_del_timer(c->write);
 }

 if (ngx_del_conn) {
     ngx_del_conn(c, NGX_CLOSE_EVENT);
 }
```
可以看到flags确实传入了NGX_CLOSE_EVENT，所以现在可以知道了：nginx在close该client fd之前，并没有用EPOLL_CTL_DEL把client fd删除。

但是如果你用测试demo，把epoll_ctl(ep, EPOLL_CTL_DEL, client...那一行注释掉，你会惊奇的发现当client继续发数据，server-parent最后的那个epoll_wait仍然会把client fd返回到应用层（即epoll惊群问题）:
parent: epoll2 return 1, fd = 5
为什么呢？因为你没有EPOLL_CTL_DEL，把注释的那行打开就好了。
nginx中的epoll_wait如果再一次拿到这个他认为已经关闭的client fd，他就会当做stale event了（通过使用instance变量），然后什么都不做。但是下次epoll_wait还是会把这个client id返回，从而导致CPU占用100%，就不知道为啥了，gdb可以看到epoll_wait返回了EPOLLIN|EPOLLHUP|EPOLLERR，查了下资料：EPOLLHUP是在对端正常关闭时发生，EPOLLERR不知道什么情况下会发生。现在可以猜测只有：server-child在client fd关闭后还在不断得向client fd写数据，从而导致server-parent不断返回错误。

总结一下：我认为这首先应该不是nginx的bug，因为nginx的worker process不会fork，这样做也省去了一次EPOLL_CTL_DEL调用；如果在自己的项目中worker在运行过程中（而非初始化时）必须要fork&amp;exec，应该将不希望child继承的fd设置close-on-exec，当然这需要很小心且细心的处理。还存在一种情况就是worker在运行过程中只fork不exec，这种情况我认为就应该避免了。
另外，在多线程程序中调用fork不是很安全，可以参考的资料：
1.<a href="https://blog.kghost.info/2013/04/27/fork-multi-thread/" target="_blank">https://blog.kghost.info/2013/04/27/fork-multi-thread/</a>
2.<a href="http://blog.codingnow.com/2011/01/fork_multi_thread.html" target="_blank">http://blog.codingnow.com/2011/01/fork_multi_thread.html</a>

2014-11-15补:
今天阅读libuv的代码，发现libuv的作者针对这个问题做了EPOLL_CTL_DEL：
``` C
void uv__platform_invalidate_fd(uv_loop_t* loop, int fd) {
  struct uv__epoll_event* events;
  struct uv__epoll_event dummy;
  uintptr_t i;
  uintptr_t nfds;

  assert(loop-&gt;watchers != NULL);

  events = (struct uv__epoll_event*) loop-&gt;watchers[loop-&gt;nwatchers];
  nfds = (uintptr_t) loop-&gt;watchers[loop-&gt;nwatchers + 1];
  if (events != NULL)
    /* Invalidate events with same file descriptor */
    for (i = 0; i &lt; nfds; i++)
      if ((int) events[i].data == fd)
        events[i].data = -1;

  /* Remove the file descriptor from the epoll.
   * This avoids a problem where the same file description remains open
   * in another process, causing repeated junk epoll events.
   *
   * We pass in a dummy epoll_event, to work around a bug in old kernels.
   */
  if (loop-&gt;backend_fd &gt;= 0)
    uv__epoll_ctl(loop-&gt;backend_fd, UV__EPOLL_CTL_DEL, fd, &amp;dummy);
}
```
