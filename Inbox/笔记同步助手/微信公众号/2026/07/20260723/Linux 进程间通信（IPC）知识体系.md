---
author: 程序媛MM
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzg4NzUyMjEzNw==&mid=2247491276&idx=1&sn=db3233cc11fa1f459984d492945250bb&chksm=ce6a28715cbbc77bf8c4e45498a9b2dd6106ff38d5914f13f2280c45f9816b782eb71a90c35d&mpshare=1&scene=1&srcid=0723LN0foT2tsNog7zRvnqHa&sharer_shareinfo=83f731968f966bc0fbb791b5326bc62c&sharer_shareinfo_first=83f731968f966bc0fbb791b5326bc62c#rd
saved: 2026-07-23 08:19:17
tags:
  - 笔记同步助手
id: 552c09ed-e737-42d9-a976-ac0f50b49cfb
---

公众号名称：女程序员的笔记本

作者名称：程序媛MM

发布时间：2026-07-23 00:01

Hello，大家好，我是程序媛MM。

本文约5700字，今天来根据《[一份靠谱的Linux终端产品应用层软件架构学习计划](https://mp.weixin.qq.com/s?__biz=Mzg4NzUyMjEzNw==&mid=2247491247&idx=1&sn=68bd91a8f1eb5b995fe7797043cdda7b&scene=21#wechat_redirect)》这篇帖子中的计划，来梳理Linux 进程间通信IPC的知识体系。

关注公众号, 即可获得与Linux相关的电子书籍以及常用开发工具，文末有文档清单。

---

# 进程间通信（Inter-Process Communication，IPC）是操作系统中至关重要的课题。Linux 作为类 UNIX 系统，提供了丰富且历史悠久的 IPC 机制，从经典的管道到 System V IPC，再到现代的 POSIX 标准和 UNIX 域套接字。本文将系统梳理 Linux 下六大核心 IPC 技术：**管道（Pipe）**、**命名管道（FIFO）**、**共享内存（SHM）**、**消息队列（MSG）**、**信号量（SEM）** 和 **UNIX 域套接字（Unix Domain Socket），**从原理、API、示例代码、适用场景和注意事项等方面深入剖析，全面掌握这些底层通信利器。

---

## 一 管道（Pipe）

### 1.1 基本原理

管道是最古老的 IPC 形式，本质是内核维护的一块环形缓冲区，提供**单向**、**字节流**的数据传输。它通过 `pipe()` 系统调用创建，返回两个文件描述符：`fd[0]` 用于读，`fd[1]` 用于写。数据从写端流入，从读端流出，遵循**先进先出**顺序。

**特点**：

-   半双工（单向），如需双向需创建两个管道。
    
-   仅适用于**父子进程**或**具有共同祖先**的进程间通信（因为文件描述符通过 fork 继承）。
    
-   数据无格式，即字节流，需双方约定协议。
    
-   默认阻塞读写：读空管道阻塞，写满管道阻塞（可设为非阻塞）。
    

### 1.2 核心 API

```
#include <unistd.h>
int pipe(int pipefd[2]);          // 成功返回0，失败-1
```

读写使用常规 `read()` / `write()` 系统调用，关闭用 `close()`。

### 1.3 示例：父子进程通信

```
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <sys/wait.h>

int main() {
    int fd[2];
    pid_t pid;
    char buf[256];

    if (pipe(fd) == -1) {
        perror("pipe");
        return1;
    }

    pid = fork();
    if (pid == -1) {
        perror("fork");
        return1;
    }

    if (pid == 0) { // 子进程：写入
        close(fd[0]);          // 关闭读端
        char *msg = "Hello world!";
        write(fd[1], msg, strlen(msg) + 1);
        close(fd[1]);
    } else {        // 父进程：读取
        close(fd[1]);          // 关闭写端
        read(fd[0], buf, sizeof(buf));
        printf("Parent received: %s\n", buf);
        close(fd[0]);
        wait(NULL);
    }
    return0;
}
```

### 1.4 适用场景

-   简单、短小的数据交换，尤其 shell 管道（如 `cmd1 | cmd2`）。
    
-   轻量级，无需额外命名，但局限于亲缘进程。
    

---

## 二 命名管道（FIFO）

### 2.1 基本原理

命名管道是对管道的扩展，通过**文件系统路径名**标识，允许**无亲缘关系**的进程访问。FIFO 在文件系统中是一个特殊文件（类型为 `p`），通过 `mkfifo` 创建。其通信行为和管道一致：单向字节流，阻塞读写。

### 2.2 核心 API

```
#include <sys/stat.h>
int mkfifo(const char *pathname, mode_t mode);   // 创建FIFO文件
```

打开 FIFO 使用 `open()`（需指定 O\_RDONLY 或 O\_WRONLY），之后用 `read`/`write` 通信。注意：打开 FIFO 时若对方未打开，默认阻塞（除非指定 `O_NONBLOCK`）。

### 2.3 示例：两端独立进程

**写端（writer.c）**：

```
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

int main() {
    int fd = open("/tmp/myfifo", O_WRONLY);
    if (fd == -1) { perror("open"); return 1; }
    char *msg = "Hello FIFO!";
    write(fd, msg, strlen(msg)+1);
    close(fd);
    return 0;
}
```

**读端（reader.c）**：

```
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    int fd = open("/tmp/myfifo", O_RDONLY);
    if (fd == -1) { perror("open"); return 1; }
    char buf[256];
    read(fd, buf, sizeof(buf));
    printf("Received: %s\n", buf);
    close(fd);
    return 0;
}
```

需先创建 FIFO：`mkfifo /tmp/myfifo 0666`（或使用 `mkfifo` 函数）。

### 2.4 适用场景

-   任意进程间通信（需文件系统路径）。
    
-   适用于生产者-消费者模型，数据流简单。
    
-   缺点：单向，且无消息边界。
    

---

## 三 共享内存（Shared Memory, SHM）

### 3.1 基本原理

共享内存是**最快**的 IPC 方式，因为它允许多个进程直接映射同一块物理内存区域，数据无需在内核和用户空间之间拷贝。进程可像访问普通内存一样读写该区域，但需要自行同步（常与信号量配合）。

Linux 下有两种实现：**System V 共享内存**（`shmget`、`shmat` 等）和 **POSIX 共享内存**（`shm_open`、`mmap`）。POSIX 更新、更简洁，本文侧重 POSIX。

### 3.2 POSIX 共享内存 API

```
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>

int shm_open(const char *name, int oflag, mode_t mode);  // 创建/打开共享内存对象，返回fd
int ftruncate(int fd, off_t length);                     // 设置大小
void *mmap(void *addr, size_t len, int prot, int flags, int fd, off_t offset); // 映射
int munmap(void *addr, size_t len);                      // 解除映射
int shm_unlink(const char *name);                        // 删除共享内存对象
```

### 3.3 示例：POSIX 共享内存

**写进程（writer）**：

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <unistd.h>

#define SHM_NAME "/myshm"
#define SHM_SIZE 1024

int main() {
    int fd = shm_open(SHM_NAME, O_CREAT | O_RDWR, 0666);
    if (fd == -1) { perror("shm_open"); exit(1); }
    ftruncate(fd, SHM_SIZE);  // 设置大小

    char *addr = mmap(NULL, SHM_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (addr == MAP_FAILED) { perror("mmap"); exit(1); }
    close(fd); // 映射后fd可关闭

    strcpy(addr, "Hello from POSIX SHM!");
    munmap(addr, SHM_SIZE);
    return0;
}
```

**读进程（reader）**：

```
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <unistd.h>

#define SHM_NAME "/myshm"
#define SHM_SIZE 1024

int main() {
    int fd = shm_open(SHM_NAME, O_RDWR, 0666);
    if (fd == -1) { perror("shm_open"); exit(1); }

    char *addr = mmap(NULL, SHM_SIZE, PROT_READ, MAP_SHARED, fd, 0);
    if (addr == MAP_FAILED) { perror("mmap"); exit(1); }
    close(fd);

    printf("Read: %s\n", addr);
    munmap(addr, SHM_SIZE);
    // 可选择 shm_unlink(SHM_NAME) 删除对象
    return0;
}
```

### 3.4 同步问题

共享内存自身不提供同步，若多进程同时读写需配合**信号量**（见第五节）或**互斥锁**（如基于共享内存的 `pthread_mutex_t` 且设置 `PTHREAD_PROCESS_SHARED`）。

### 3.5 适用场景

-   需要频繁交换大量数据（如图像、音视频帧）。
    
-   追求极致性能，减少拷贝开销。
    
-   注意：管理复杂，易出错，需谨慎同步。
    

---

## 四 消息队列（Message Queue, MSG）

### 4.1 基本原理

消息队列是**内核维护的消息链表**，每个消息具有**类型**和**正文**。进程通过消息类型选择性地读取，支持**多对多**通信，消息有边界，按优先级或类型检索。Linux 支持 System V 消息队列（`msgget`、`msgsnd`、`msgrcv`）和 POSIX 消息队列（`mq_open`、`mq_send`、`mq_receive`）。POSIX 更现代，接口更简洁，支持异步通知。

### 4.2 POSIX 消息队列 API

```
#include <mqueue.h>
mqd_t mq_open(const char *name, int oflag, mode_t mode, struct mq_attr *attr);
int mq_send(mqd_t mqdes, const char *msg_ptr, size_t msg_len, unsigned int msg_prio);
ssize_t mq_receive(mqd_t mqdes, char *msg_ptr, size_t msg_len, unsigned int *msg_prio);
int mq_close(mqd_t mqdes);
int mq_unlink(const char *name);
```

### 4.3 示例：POSIX 消息队列

**发送端**：

```
#include <stdio.h>
#include <stdlib.h>
#include <mqueue.h>
#include <string.h>

#define QUEUE_NAME "/myqueue"

int main() {
    mqd_t mq = mq_open(QUEUE_NAME, O_CREAT | O_WRONLY, 0666, NULL);
    if (mq == (mqd_t)-1) { perror("mq_open"); exit(1); }

    char *msg = "Hello Message Queue!";
    if (mq_send(mq, msg, strlen(msg)+1, 0) == -1) { perror("mq_send"); }
    mq_close(mq);
    return0;
}
```

**接收端**：

```
#include <stdio.h>
#include <stdlib.h>
#include <mqueue.h>

#define QUEUE_NAME "/myqueue"
#define MAX_MSG_SIZE 256

int main() {
    mqd_t mq = mq_open(QUEUE_NAME, O_RDONLY, 0666, NULL);
    if (mq == (mqd_t)-1) { perror("mq_open"); exit(1); }

    char buf[MAX_MSG_SIZE];
    unsignedint prio;
    ssize_t len = mq_receive(mq, buf, MAX_MSG_SIZE, &prio);
    if (len == -1) { perror("mq_receive"); }
    else {
        buf[len] = '\0';
        printf("Received (prio=%u): %s\n", prio, buf);
    }
    mq_close(mq);
    mq_unlink(QUEUE_NAME); // 可删除
    return0;
}
```

### 4.4 特性与限制

-   消息有最大长度和队列最大消息数，可通过 `mq_attr` 设置。
    
-   支持优先级（数值越大优先级越高）。
    
-   支持非阻塞和超时（`mq_timedsend`/`mq_timedreceive`）。
    
-   内核持久化：消息队列在系统重启前一直存在（除非 unlink）。
    

### 4.5 适用场景

-   需要可靠、有边界、带类型的消息传递。
    
-   适合异步处理、事件驱动系统。
    
-   相比共享内存速度稍慢，但更易用且自带同步（队列操作原子）。
    

---

## 五 信号量（Semaphore, SEM）

### 5.1 基本原理

信号量是一种用于**同步**和**互斥**的计数器，支持 P（等待/减）和 V（释放/加）操作。Linux 同样有两种：**System V 信号量**（`semget`、`semop`）和 **POSIX 命名/匿名信号量**（`sem_open`、`sem_wait`、`sem_post`）。POSIX 更简洁，本文重点。

信号量分为：

-   **二进制信号量**：类似互斥锁，值0或1。
    
-   **计数信号量**：允许多个资源并发访问。
    

### 5.2 POSIX 信号量 API

```
#include <semaphore.h>
// 命名信号量（用于无亲缘进程）
sem_t *sem_open(const char *name, int oflag, mode_t mode, unsigned int value);
int sem_wait(sem_t *sem);      // P操作，阻塞
int sem_trywait(sem_t *sem);   // 非阻塞
int sem_post(sem_t *sem);      // V操作
int sem_close(sem_t *sem);
int sem_unlink(const char *name);

// 匿名信号量（用于共享内存中的同步，或线程同步）
int sem_init(sem_t *sem, int pshared, unsigned int value);  // pshared=1表示进程共享
int sem_destroy(sem_t *sem);
```

### 5.3 示例：使用命名信号量同步共享内存

（结合共享内存示例，信号量保证写者先写，读者后读）**写者**：

```
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <semaphore.h>
#include <unistd.h>
#include <string.h>

#define SHM_NAME "/myshm"
#define SEM_NAME "/mysem"
#define SHM_SIZE 1024

int main() {
    // 创建信号量，初值为0
    sem_t *sem = sem_open(SEM_NAME, O_CREAT, 0666, 0);
    if (sem == SEM_FAILED) { perror("sem_open"); exit(1); }

    int fd = shm_open(SHM_NAME, O_CREAT | O_RDWR, 0666);
    ftruncate(fd, SHM_SIZE);
    char *addr = mmap(NULL, SHM_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    close(fd);

    strcpy(addr, "Data from writer");
    sem_post(sem); // 通知读者数据已就绪

    munmap(addr, SHM_SIZE);
    // 不关闭信号量，留给读者
    return0;
}
```

**读者**：

```
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <semaphore.h>
#include <unistd.h>

#define SHM_NAME "/myshm"
#define SEM_NAME "/mysem"
#define SHM_SIZE 1024

int main() {
    sem_t *sem = sem_open(SEM_NAME, 0); // 打开已存在的信号量
    if (sem == SEM_FAILED) { perror("sem_open"); exit(1); }

    int fd = shm_open(SHM_NAME, O_RDWR, 0666);
    char *addr = mmap(NULL, SHM_SIZE, PROT_READ, MAP_SHARED, fd, 0);
    close(fd);

    sem_wait(sem); // 等待写者完成
    printf("Read: %s\n", addr);

    munmap(addr, SHM_SIZE);
    sem_close(sem);
    sem_unlink(SEM_NAME);
    shm_unlink(SHM_NAME);
    return0;
}
```

### 5.4 注意事项

-   信号量的操作是**原子**的，适合多进程同步。
    
-   命名信号量存在于 `/dev/shm/` 或 `/run/shm/`（虚拟文件系统）。
    
-   匿名信号量需放在共享内存中才能跨进程。
    

### 5.5 适用场景

-   保护共享资源（互斥）。
    
-   控制并发数量（计数信号量）。
    
-   协调多进程执行顺序（如生产者-消费者）。
    

---

## 六 UNIX 域套接字（Unix Domain Socket）

### 6.1 基本原理

UNIX 域套接字（AF\_UNIX）是同一主机上进程间**全双工**、**可靠**的通信方式，语法与网络套接字类似，但地址是文件系统路径（或抽象命名空间）。它支持**流式（SOCK\_STREAM，类似 TCP）** 和**数据报（SOCK\_DGRAM，类似 UDP，但可靠且不丢包）**。UNIX 域套接字在内核中直接交换数据，无需网络协议栈，性能优异，且能传递**文件描述符**（通过 `SCM_RIGHTS`）和**凭证**（SCM\_CREDENTIALS）。

### 6.2 核心 API（与网络套接字基本相同）

```
#include <sys/socket.h>
#include <sys/un.h>

int socket(int domain, int type, int protocol);   // domain = AF_UNIX
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
int listen(int sockfd, int backlog);
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
ssize_t send/recv 或 write/read
```

地址结构：

```
struct sockaddr_un {
    sa_family_t sun_family;    // AF_UNIX
    char sun_path[108];        // 路径名
};
```

### 6.3 示例：流式 UNIX 域套接字（简单回射服务器）

**服务端（server.c）**：

```
#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <unistd.h>
#include <string.h>

#define SOCK_PATH "/tmp/uds_socket"

int main() {
    int sock = socket(AF_UNIX, SOCK_STREAM, 0);
    if (sock == -1) { perror("socket"); exit(1); }

    struct sockaddr_un addr;
    memset(&addr, 0, sizeof(addr));
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, SOCK_PATH, sizeof(addr.sun_path)-1);

    unlink(SOCK_PATH); // 确保旧文件不存在
    if (bind(sock, (struct sockaddr*)&addr, sizeof(addr)) == -1) {
        perror("bind"); exit(1);
    }
    listen(sock, 5);

    int conn = accept(sock, NULL, NULL);
    if (conn == -1) { perror("accept"); exit(1); }

    char buf[256];
    ssize_t n = read(conn, buf, sizeof(buf)-1);
    if (n > 0) {
        buf[n] = '\0';
        printf("Received: %s\n", buf);
        write(conn, buf, n); // echo back
    }
    close(conn);
    close(sock);
    unlink(SOCK_PATH);
    return0;
}
```

**客户端（client.c）**：

```
#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <unistd.h>
#include <string.h>

#define SOCK_PATH "/tmp/uds_socket"

int main() {
    int sock = socket(AF_UNIX, SOCK_STREAM, 0);
    if (sock == -1) { perror("socket"); exit(1); }

    struct sockaddr_un addr;
    memset(&addr, 0, sizeof(addr));
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, SOCK_PATH, sizeof(addr.sun_path)-1);

    if (connect(sock, (struct sockaddr*)&addr, sizeof(addr)) == -1) {
        perror("connect"); exit(1);
    }

    char *msg = "Hello UDS!";
    write(sock, msg, strlen(msg)+1);

    char buf[256];
    ssize_t n = read(sock, buf, sizeof(buf)-1);
    if (n > 0) {
        buf[n] = '\0';
        printf("Echo: %s\n", buf);
    }
    close(sock);
    return0;
}
```

### 6.4 高级特性

-   **传递文件描述符**：通过 `sendmsg`/`recvmsg` 和 `cmsg` 结构，可将一个进程打开的文件描述符传递给另一个进程，实现文件共享。
    
-   **传递凭证**：可获取对端进程的 PID、UID、GID，用于权限验证。
    

### 6.5 适用场景

-   需要全双工、可靠、面向连接的通信。
    
-   希望使用套接字编程接口，且无需网络。
    
-   需要在进程间传递文件描述符或凭证。
    
-   性能优于 TCP 本地环回（127.0.0.1），且更轻量。
    

---

## 七 对比总结与选型指南

| IPC 机制 | 方向 | 数据边界 | 同步机制 | 通信范围 | 性能 | 复杂度 |
| --- | --- | --- | --- | --- | --- | --- |
| 管道 (Pipe) | 单向 | 流 | 无需（阻塞） | 亲缘进程 | 中等 | 低 |
| 命名管道 (FIFO) | 单向 | 流 | 无需（阻塞） | 任意进程 | 中等 | 低 |
| 共享内存 (SHM) | 双向 | 无 | 需外部同步（如信号量） | 任意进程 | 最高 | 高 |
| 消息队列 (MSG) | 双向 | 消息 | 自带原子性 | 任意进程 | 较高 | 中 |
| 信号量 (SEM) | N/A | N/A | 内建原子操作 | 任意进程 | 很高 | 中 |
| UNIX 域套接字 | 全双工 | 流/数据报 | 自带（如TCP流控） | 任意进程 | 高 | 中高 |

**选型建议**：

-   **简单数据传递、亲缘进程**：优先管道。
    
-   **无亲缘进程、单向字节流**：FIFO。
    
-   **大量数据、高吞吐**：共享内存 + 信号量同步。
    
-   **有消息边界、带优先级、异步通信**：消息队列。
    
-   **仅需同步互斥**：信号量（或互斥锁）。
    
-   **复杂双向通信、需传递描述符/凭证、熟悉 socket 编程**：UNIX 域套接字。
    

---

## 八 实战注意事项

1.  **资源清理**：System V IPC 对象（shm、msg、sem）默认内核持久化，需手动删除（`ipcrm` 或 `shm_unlink` 等），否则可能泄漏。POSIX 对象位于 `/dev/shm/` 或 `/run/user/`，系统重启会自动清除，但最好显式 unlink。
    
2.  **权限管理**：所有 IPC 对象都有文件权限位，需合理设置，避免安全风险。
    
3.  **阻塞与非阻塞**：管道、FIFO、消息队列、套接字均支持非阻塞模式（`O_NONBLOCK`），信号量有 `sem_trywait`。
    
4.  **信号安全**：在信号处理函数中，仅可使用异步信号安全的 IPC 函数（如 `write` 到管道），谨慎使用复杂 IPC。
    
5.  **性能基准**：共享内存最快，但同步开销不容忽视；UNIX 域套接字在双向通信中表现优异，避免 TCP/IP 开销。
    
6.  **可移植性**：System V 是传统标准，POSIX 更现代，建议优先选用 POSIX 接口。
    

---

## 九、总结

Linux 提供了丰富的 IPC 机制，每种都有其设计哲学和适用舞台。理解它们的底层原理和 API 特性，能帮助我们根据具体需求做出最优选择。无论是轻量级的管道，还是高性能的共享内存，亦或是灵活强大的 UNIX 域套接字，掌握它们都是后端开发、系统编程和架构设计的重要基础。

希望本文能为你构建起完整的 IPC 知识体系,欢迎留言讨论！

## 往期文章（欢迎订阅技术分享栏目全部文章）：

[【从零开始撸内核驱动源码】：以ttyserial(串口驱动)为例，串联字符设备驱动基础知识点的学习计划](https://mp.weixin.qq.com/s?__biz=Mzg4NzUyMjEzNw==&mid=2247490000&idx=1&sn=b219117d0c60c20221230c96b2bdcb48&scene=21#wechat_redirect)

[Linux内核源码顶层 Makefile分析并单独编译调试内核自带的驱动](https://mp.weixin.qq.com/s?__biz=Mzg4NzUyMjEzNw==&mid=2247489944&idx=1&sn=5c13e4390fcb78b4e10da9de12c9e546&scene=21#wechat_redirect)

[【从零开始撸内核驱动源码】：ttynull驱动](https://mp.weixin.qq.com/s?__biz=Mzg4NzUyMjEzNw==&mid=2247489934&idx=1&sn=cb8461b10214fe60f497a88ab326aeed&scene=21#wechat_redirect)

[Linux内核驱动安装失败问题调试及解决方法](https://mp.weixin.qq.com/s?__biz=Mzg4NzUyMjEzNw==&mid=2247489880&idx=1&sn=3bfc6b7f94cc7fbdaa3a387625707d74&scene=21#wechat_redirect)

[Linux内核驱动源码走读之编译内核及外部驱动实操指南](https://mp.weixin.qq.com/s?__biz=Mzg4NzUyMjEzNw==&mid=2247489856&idx=1&sn=9d004f36426104bdaa53e1c1d6b4c4bc&scene=21#wechat_redirect)

![[Inbox/笔记同步助手/微信公众号/2026/07/images/20195a78c0816bd978be7a7b0a069c60_MD5.jpg]]“**谢谢你看到这里**”

**嵌入式Linux设备内部跨模块通信方案对比与统一消息架构设计**

**嵌入式Linux设备内部跨模块通信方案对比与统一消息架构设计**

**分享读书心得、工作经验，自我成长和生活方式。**

**希望我的文字能对你有所帮助**

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/dd6bbb3a_1784765956123?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzg4NzUyMjEzNw%3D%3D%26mid%3D2247491276%26idx%3D1%26sn%3Ddb3233cc11fa1f459984d492945250bb%26chksm%3Dce6a28715cbbc77bf8c4e45498a9b2dd6106ff38d5914f13f2280c45f9816b782eb71a90c35d%26mpshare%3D1%26scene%3D1%26srcid%3D0723LN0foT2tsNog7zRvnqHa%26sharer_shareinfo%3D83f731968f966bc0fbb791b5326bc62c%26sharer_shareinfo_first%3D83f731968f966bc0fbb791b5326bc62c%23rd&s=obsidian)