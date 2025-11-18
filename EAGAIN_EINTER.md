在C++中读取非阻塞socket时，`EINTR`和`EAGAIN`是两个常见的错误，它们有不同的含义和处理方式：

## EINTR (Interrupted System Call)

### 原因
- 系统调用被信号中断
- 通常在程序收到信号（如SIGINT、SIGALRM等）时发生

### 处理方式
```cpp
#include <sys/socket.h>
#include <errno.h>
#include <unistd.h>

ssize_t read_nonblocking(int sockfd, void* buf, size_t count) {
    ssize_t n;
    
    while (true) {
        n = read(sockfd, buf, count);
        if (n >= 0) {
            return n;  // 读取成功
        }
        
        if (errno == EINTR) {
            continue;  // 被信号中断，重试读取
        }
        
        // 处理其他错误
        return -1;
    }
}
```

## EAGAIN / EWOULDBLOCK

### 原因
- 非阻塞socket上没有立即可用的数据
- 对于读操作：接收缓冲区为空
- 对于写操作：发送缓冲区已满

### 处理方式
```cpp
ssize_t read_nonblocking_with_retry(int sockfd, void* buf, size_t count) {
    ssize_t n = read(sockfd, buf, count);
    
    if (n >= 0) {
        return n;  // 读取成功
    }
    
    if (errno == EAGAIN || errno == EWOULDBLOCK) {
        return 0;  // 没有数据可用，但不是错误
    }
    
    if (errno == EINTR) {
        // 可以选择重试或返回特定值
        return -2;  // 表示被中断
    }
    
    return -1;  // 真正的错误
}
```

## 完整的非阻塞socket读取示例

```cpp
#include <sys/socket.h>
#include <sys/select.h>
#include <errno.h>
#include <unistd.h>
#include <fcntl.h>

class NonBlockingSocket {
private:
    int sockfd;
    
public:
    // 设置socket为非阻塞模式
    bool set_nonblocking() {
        int flags = fcntl(sockfd, F_GETFL, 0);
        if (flags == -1) return false;
        return fcntl(sockfd, F_SETFL, flags | O_NONBLOCK) != -1;
    }
    
    // 带超时的读取
    ssize_t read_with_timeout(void* buf, size_t count, int timeout_sec) {
        fd_set readfds;
        struct timeval timeout;
        
        while (true) {
            FD_ZERO(&readfds);
            FD_SET(sockfd, &readfds);
            
            timeout.tv_sec = timeout_sec;
            timeout.tv_usec = 0;
            
            int ready = select(sockfd + 1, &readfds, nullptr, nullptr, &timeout);
            
            if (ready == -1) {
                if (errno == EINTR) {
                    continue;  // 被信号中断，重试select
                }
                return -1;  // select错误
            }
            
            if (ready == 0) {
                return 0;  // 超时
            }
            
            // socket可读
            ssize_t n = read(sockfd, buf, count);
            
            if (n >= 0) {
                return n;  // 读取成功
            }
            
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                // 虽然select说可读，但实际读取时没有数据
                // 这可能在多线程环境中发生
                continue;
            }
            
            if (errno == EINTR) {
                continue;  // 被信号中断，重试读取
            }
            
            return -1;  // 其他错误
        }
    }
    
    // 简单的非阻塞读取
    ssize_t read_nonblocking(void* buf, size_t count) {
        while (true) {
            ssize_t n = read(sockfd, buf, count);
            
            if (n >= 0) {
                return n;
            }
            
            switch (errno) {
                case EINTR:
                    continue;  // 重试
                    
                case EAGAIN:
                case EWOULDBLOCK:
                    return 0;  // 没有数据可用
                    
                default:
                    return -1;  // 真正的错误
            }
        }
    }
};
```

## 在实际应用中的建议

1. **EINTR处理**：通常应该重试操作，除非有特定的信号处理需求
2. **EAGAIN处理**：应该返回并让出CPU，等待下次可读事件
3. **使用I/O多路复用**：结合select/poll/epoll来避免忙等待
4. **错误日志**：记录真正的错误，但不要将EAGAIN视为错误

```cpp
// 在实际网络编程中的典型模式
void handle_socket_io(int sockfd) {
    char buffer[1024];
    
    while (true) {
        ssize_t n = read(sockfd, buffer, sizeof(buffer));
        
        if (n > 0) {
            // 处理接收到的数据
            process_data(buffer, n);
        } else if (n == 0) {
            // 对方关闭连接
            close(sockfd);
            break;
        } else {
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                // 没有更多数据，返回事件循环
                break;
            } else if (errno == EINTR) {
                // 被信号中断，继续读取
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
```

记住，在非阻塞socket编程中，`EAGAIN`是正常情况而不是错误，应该优雅地处理它。
