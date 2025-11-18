这个错误信息 **"Resource temporarily unavailable"** 对应的是 `EAGAIN` 错误（在有些系统上是 `EWOULDBLOCK`）。

## 错误来源

这个错误通常来自以下几个系统调用：

```cpp
// 读取操作
ssize_t n = read(sockfd, buffer, size);
ssize_t n = recv(sockfd, buffer, size, flags);
ssize_t n = recvfrom(sockfd, buffer, size, flags, addr, addrlen);

// 写入操作  
ssize_t n = write(sockfd, buffer, size);
ssize_t n = send(sockfd, buffer, size, flags);
ssize_t n = sendto(sockfd, buffer, size, flags, addr, addrlen);

// 接受连接
int client_fd = accept(sockfd, addr, addrlen);
```

## 触发原因

### 对于**读操作**触发：
```cpp
ssize_t n = read(sockfd, buffer, size);
if (n == -1 && errno == EAGAIN) {
    // 触发条件：TCP接收缓冲区为空，没有数据可读
}
```

**原因**：
- socket设置为非阻塞模式（`O_NONBLOCK`）
- 接收缓冲区中暂时没有数据可读
- 对方还没有发送数据，或者数据还在网络传输中

### 对于**写操作**触发：
```cpp
ssize_t n = write(sockfd, buffer, size);
if (n == -1 && errno == EAGAIN) {
    // 触发条件：TCP发送缓冲区已满
}
```

**原因**：
- socket设置为非阻塞模式
- 发送缓冲区已满，无法立即容纳更多数据
- 网络拥塞或接收方处理速度慢

### 对于**accept操作**触发：
```cpp
int client_fd = accept(sockfd, NULL, NULL);
if (client_fd == -1 && errno == EAGAIN) {
    // 触发条件：没有等待接受的连接
}
```

**原因**：
- 监听socket设置为非阻塞模式
- 没有新的连接请求在等待接受

## 诊断方法

```cpp
#include <iostream>
#include <cstring>
#include <errno.h>
#include <sys/socket.h>

void check_socket_error(int result, const char* operation) {
    if (result == -1) {
        std::cout << operation << " failed: " << strerror(errno) 
                  << " (errno=" << errno << ")" << std::endl;
        
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            std::cout << "这是EAGAIN错误，表示资源暂时不可用" << std::endl;
            
            // 检查socket是否处于非阻塞模式
            int flags = fcntl(sockfd, F_GETFL, 0);
            if (flags & O_NONBLOCK) {
                std::cout << "socket处于非阻塞模式，这是正常情况" << std::endl;
            }
        }
    }
}

// 使用示例
void read_from_socket(int sockfd) {
    char buffer[1024];
    ssize_t n = read(sockfd, buffer, sizeof(buffer));
    
    check_socket_error(n, "read");
    
    if (n == -1 && (errno == EAGAIN || errno == EWOULDBLOCK)) {
        std::cout << "没有数据可读，应该稍后重试或等待可读事件" << std::endl;
    }
}
```

## 解决方案

### 方案1：使用I/O多路复用（推荐）
```cpp
#include <sys/select.h>

void wait_for_readable(int sockfd, int timeout_sec) {
    fd_set readfds;
    struct timeval timeout;
    
    FD_ZERO(&readfds);
    FD_SET(sockfd, &readfds);
    
    timeout.tv_sec = timeout_sec;
    timeout.tv_usec = 0;
    
    int ready = select(sockfd + 1, &readfds, NULL, NULL, &timeout);
    
    if (ready > 0) {
        // socket可读，现在读取不会返回EAGAIN
        char buffer[1024];
        ssize_t n = read(sockfd, buffer, sizeof(buffer));
        // 这次读取应该成功（除非有真正错误）
    }
}
```

### 方案2：正确的非阻塞I/O处理模式
```cpp
class NonBlockingSocketHandler {
public:
    void handle_read(int sockfd) {
        while (true) {
            char buffer[1024];
            ssize_t n = read(sockfd, buffer, sizeof(buffer));
            
            if (n > 0) {
                // 成功读取到数据
                process_data(buffer, n);
            } else if (n == 0) {
                // 对方关闭连接
                close(sockfd);
                break;
            } else {
                if (errno == EAGAIN || errno == EWOULDBLOCK) {
                    // 没有更多数据了，退出循环
                    break;
                } else if (errno == EINTR) {
                    // 被信号中断，继续重试
                    continue;
                } else {
                    // 真正的错误
                    perror("read error");
                    close(sockfd);
                    break;
                }
            }
        }
    }
};
```

### 方案3：设置合适的socket选项
```cpp
// 增大接收缓冲区大小
int recv_buf_size = 64 * 1024;  // 64KB
setsockopt(sockfd, SOL_SOCKET, SO_RCVBUF, &recv_buf_size, sizeof(recv_buf_size));

// 增大发送缓冲区大小  
int send_buf_size = 64 * 1024;  // 64KB
setsockopt(sockfd, SOL_SOCKET, SO_SNDBUF, &send_buf_size, sizeof(send_buf_size));
```

## 总结

- **"Resource temporarily unavailable"** = `EAGAIN`/`EWOULDBLOCK`
- **根本原因**：非阻塞socket上的I/O操作无法立即完成
- **这不是错误**，而是非阻塞I/O的正常行为
- **解决方案**：使用select/poll/epoll等待socket就绪，或者在有数据时重试操作

这个"错误"实际上告诉你："我现在没法立即完成这个操作，请稍后再试或换种方式处理"。
