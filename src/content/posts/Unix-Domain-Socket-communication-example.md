---
title: Unix-Domain-Socket-communication-example
published: 2025-11-22
description: '利用UDS实现进程间通信'
image: ''
tags: []
category: ''
draft: false 
lang: ''
---

## server
基于epoll实现IO多路复用，能够连接多个前端。

创建一个socket文件`/tmp/iomultiplex_server.sock`，并监听它。

通过`SO_PEERCRED`获取client的pid。

当来的一个event的`fd==listen_fd`的时候，说明是一个新建立的连接，则向`epoll_fd`注册一个新的监听，也添加到`client_fds`这个布尔数组中，方便之后做资源的清理。

如果是其它fd,则说明已经连接过的，在实例代码中，也使用布尔数组检查了一下是否是这样的。

当连接断开的时候，会首先向`epoll_fd`删除这个`client_fd`之前的注册，然后关闭这个`client_fd`，最后也把`client_fds`中的布尔值归零。

```c
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <signal.h>
#include <stdbool.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include <sys/epoll.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <unistd.h>

#define SOCKET_PATH "/tmp/iomultiplex_server.sock"
#define MAX_EVENTS 10
#define BUF_SIZE 256

static volatile sig_atomic_t stop = 0;

void sigint_handler(int sig) {
  (void)sig; // depress `unsed` warning
  stop = 1;
}

int make_nonblocking(int fd) {
  int flags = fcntl(fd, F_GETFL, 0);
  if (flags == -1)
    return -1;
  return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

int main() {
  if (signal(SIGINT, sigint_handler) == SIG_ERR) {
    perror("signal handler");
    exit(1);
  }

  int listen_fd, epoll_fd;
  struct sockaddr_un addr;
  struct epoll_event ev, events[MAX_EVENTS] = {0};
  char buffer[BUF_SIZE];

  // delete old socket file
  unlink(SOCKET_PATH);

  // create socket listen
  listen_fd = socket(AF_UNIX, SOCK_STREAM, 0);
  if (listen_fd < 0) {
    perror("creating socket");
    exit(1);
  }

  addr.sun_family = AF_UNIX;
  strncpy(addr.sun_path, SOCKET_PATH, sizeof(addr.sun_path));

  if (bind(listen_fd, (struct sockaddr *)&addr, sizeof(addr)) == -1) {
    perror("bind");
    close(listen_fd);
    exit(1);
  }

  if (listen(listen_fd, 10) == -1) {
    perror("listen");
    close(listen_fd);
    exit(1);
  }

  // create epoll instance
  epoll_fd = epoll_create1(0);
  if (epoll_fd < 0) {
    perror("epoll_create1");
    close(listen_fd);
    exit(1);
  }

  // register listen socket to epoll
  ev.events = EPOLLIN;
  ev.data.fd = listen_fd;
  if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, listen_fd, &ev) == -1) {
    perror("epoll_ctl: listen fd");
    close(listen_fd);
    close(epoll_fd);
    exit(1);
  }

  printf("server listening on %s\n", SOCKET_PATH);

  bool client_fds[MAX_EVENTS + 1] = {0};
  while (!stop) {
    int nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
    if (nfds == -1) {
      if (errno == EINTR)
        continue;
      perror("epoll_wait");
      break;
    }

    for (int i = 0; i < nfds; i++) {
      if (events[i].data.fd == listen_fd) {
        // new connection
        int client_fd = accept(listen_fd, NULL, NULL);
        if (client_fd < 0) {
          if (errno == EAGAIN || errno == EWOULDBLOCK)
            continue;
          perror("accept");
          continue;
        }
        if (make_nonblocking(client_fd) == -1) {
          perror("make_nonblocking client fd");
          close(client_fd);
          continue;
        }
        ev.events = EPOLLIN | EPOLLRDHUP; // readable and close
        ev.data.fd = client_fd;
        if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, client_fd, &ev) == -1) {
          perror("epoll_ctl: client fd");
          close(client_fd);
        } else {
          printf("new client connected: fd=%d\n", client_fd);
          fflush(stdout);
        }
        client_fds[client_fd] = true;
      } else {
        // connected
        int client_fd = events[i].data.fd;
        if (client_fds[client_fd])
          printf("client: %d already connected!\n", client_fd);
        else
          fprintf(stderr,
                  "client_fd: %d not registered! this should never happen!\n",
                  client_fd);
        ssize_t count;
        if (events[i].events & (EPOLLHUP | EPOLLRDHUP)) {
          // client closed
          printf("client fd=%d disconnected\n", client_fd);
          epoll_ctl(epoll_fd, EPOLL_CTL_DEL, client_fd, NULL);
          close(client_fd);
          client_fds[client_fd] = false;
          continue;
        }
        count = read(client_fd, buffer, sizeof(buffer) - 1);
        if (count > 0) {
          buffer[count] = '\0';
          // get client pid by SO_PEERCRED
          struct ucred ucred;
          socklen_t len = sizeof(ucred);
          if (getsockopt(client_fd, SOL_SOCKET, SO_PEERCRED, &ucred, &len) ==
              0) {
            printf("%d:'%s'\n", (int)ucred.pid, buffer);
          } else {
            printf("pid: unknown: '%s'\n", buffer);
          }
          fflush(stdout);
        } else if (count == 0) {
          // client closed
          printf("client fd=%d closed connection\n", client_fd);
          close(client_fd);
          epoll_ctl(epoll_fd, EPOLL_CTL_DEL, client_fd, NULL);
          client_fds[client_fd] = false;
        } else {
          if (errno != EAGAIN && errno != EWOULDBLOCK) {
            perror("read");
            close(client_fd);
            epoll_ctl(epoll_fd, EPOLL_CTL_DEL, client_fd, NULL);
            client_fds[client_fd] = false;
          }
          // EAGIAN: no more data, nomal
        }
      }
    }
  }

  printf("shutting down server\n");
  for (int i = 0; i <= MAX_EVENTS; i++) {
    if (client_fds[i]) {
      close(i);
      epoll_ctl(epoll_fd, EPOLL_CTL_DEL, i, NULL);
    }
  }
  epoll_ctl(epoll_fd, EPOLL_CTL_DEL, listen_fd, NULL);
  close(listen_fd);
  close(epoll_fd);
  unlink(SOCKET_PATH);
  printf("all resource freed\n");
  return 0;
}
```

## client
通过`fork()`系统调用创建多个进程，同时连接到server。

如果在`write()`之后不`usleep(1)`一下的话，server可能会接收成连续的数据，即数据接收不太正确。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

#include <unistd.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <sys/types.h>
#include <sys/wait.h>

#define SOCKET_PATH "/tmp/iomultiplex_server.sock"
#define BUF_SIZE 256
#define MAX_PROCS 10
#define NUM_MSGS 10

int child(int id) {
  struct sockaddr_un addr;
  char msg[BUF_SIZE];
  pid_t my_pid = getpid();

  // sleep random time
  srand(my_pid ^ time(NULL));
  sleep(rand() % 4);

  int sock = socket(AF_UNIX, SOCK_STREAM, 0);
  if (sock == -1) {
    perror("socket");
    exit(1);
  }

  addr.sun_family = AF_UNIX;
  strncpy(addr.sun_path, SOCKET_PATH, strlen(SOCKET_PATH));

  if (connect(sock, (struct sockaddr*)&addr, sizeof(addr)) == -1) {
    perror("connect");
    close(sock);
    exit(1);
  }
  snprintf(msg, sizeof(msg), "Hello from client %d", (int)my_pid);
  if (write(sock, msg, strlen(msg)) == -1) {
    perror("write");
  }
  usleep(100);

  for (int i = 0; i < NUM_MSGS; i++) {
    snprintf(msg, sizeof(msg), "%d", rand() % 10);
    if (write(sock, msg, strlen(msg)) == -1) {
      perror("write");
    }
    usleep(1);
  }

  close(sock);
  return 0;
}

int main() {
  pid_t pids[MAX_PROCS] = {0};
  pid_t pid;
  for (int i = 0; i < MAX_PROCS; i++) {
    pid = fork();
    if (pid == 0) {
      // child
      exit(child(i));
    } else if (pid == -1) {
      // fork failed
      perror("fork");
      exit(1);
    } else {
      // parent
      pids[i] = pid;
    }
  }

  printf("forked pids: ");
  for (int i = 0; i < MAX_PROCS; i++) {
    printf("%d ", pids[i]);
  }
  printf("\n");

  for (int i = 0; i < MAX_PROCS; i++) {
    int status;
    waitpid(pids[i], &status, 0);
    if (WIFEXITED(status)) {
      printf("client #%d exited with status %d\n", i + 1, WEXITSTATUS(status));
    }
  }
  printf("all child stopped\n");
  return 0;
}
```