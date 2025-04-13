---
title: "Building a High-Performance Chat Server with C++ and epoll"
datePublished: Sun Apr 13 2025 07:19:43 GMT+0000 (Coordinated Universal Time)
cuid: cm9fbe80n000c0ajm4qrn5rzp
slug: cpp-chat-server
tags: cpp, backend, epoll, network-programming, chat-server

---

After implementing a [simple chat server in Go](https://davidzhang.hashnode.dev/go-chat-server) using goroutines for client handling, I wanted to explore an alternative approach using C++ and the `epoll` event notification mechanism. While Go abstracts away many of the complexities of concurrent programming with goroutines, C++ with `epoll` gives us fine-grained control over I/O operations, potentially leading to better performance for high-load network applications.

In this article, I'll explore how to implement an efficient, event-driven chat server in C++ that handles multiple clients concurrently through a single-threaded event loop—an interesting contrast to Go's goroutine-based approach.

**Why We Don't Use Thread-Per-Connection in C++**

The Thread-Per-Connection model assigns one OS thread to each client connection, providing a simple code flow. However, it has the following disadvantages:

1. **Memory Consumption**: OS threads consume 1-2MB each, meaning 1,000 connections could require 2GB of memory just for stacks.
    
2. **Context Switching Overhead**: Threads require expensive CPU register saves/restores and cache flushes when switching between them.
    
3. **Scheduler Limitations**: As thread count grows (thousands of them at most), the OS scheduler becomes a bottleneck, causing performance degradation and unpredictable latency.
    

## **Architecture Overview: Event-Driven vs. Goroutine-Based**

Unlike the Go implementation that spawns a goroutine for each client, our C++ server uses an event loop with epoll to monitor multiple file descriptors simultaneously. This approach can achieve impressive performance with minimal resource usage, especially for I/O-bound applications with many connections.

```plaintext
Go Server:               C++ Server:
+------------+          +------------+
| Accept Loop|          | Event Loop |
+-----+------+          +-----+------+
      |                       |
+-----v------+          +-----v------+
| Goroutine  |          | epoll_wait |
| per client |          | for events |
+------------+          +-----+------+
      |                       |
+-----v------+          +-----v------+
| Shared Map |          | Handle I/O |
| with Mutex |          | on ready FD|
+------------+          +------------+
```

## **Client Representation**

First, let's define our Client class, which encapsulates a connected user and implements non-blocking message handling:

```cpp
class Client {
public:
    int fd;
    std::string nick;
    std::string buffer;
    std::queue<std::string> outgoingQueue;  // Queue for outgoing messages
    std::string currentSendBuffer;         // Current message being sent

    Client(int fd) : fd(fd) {
        nick = generateRandomNick(4);
    }

    ~Client() {
        close(fd);
    }

    // Add a message to the outgoing queue
    void queueMessage(const std::string& message) {
        outgoingQueue.push(message);
    }

    // Check if client has pending data to send
    bool hasDataToSend() const {
        return !currentSendBuffer.empty() || !outgoingQueue.empty();
    }

    // Try to send data from queue, returns status code indicating result
    SendResult trySendData() {
        // If currentSendBuffer is empty but queue isn't, move a message to currentSendBuffer
        if (currentSendBuffer.empty() && !outgoingQueue.empty()) {
            currentSendBuffer = outgoingQueue.front();
            outgoingQueue.pop();
        }

        if (currentSendBuffer.empty()) {
            return SendResult::COMPLETE; // No data to send, all done
        }

        ssize_t sent = send(fd, currentSendBuffer.c_str(), currentSendBuffer.size(), 0);
        if (sent < 0) {
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                return SendResult::PENDING; // Socket buffer full, try again later
            }
            // Other error with the TCP connection
            std::cerr << "Socket error on fd " << fd << ": " << strerror(errno) << std::endl;
            return SendResult::ERROR; // Signal that connection should be closed
        }

        // If we sent part of the message, keep the rest for later
        if (static_cast<size_t>(sent) < currentSendBuffer.size()) {
            currentSendBuffer = currentSendBuffer.substr(sent);
            return SendResult::PENDING; // More data pending
        }

        // Message completely sent
        currentSendBuffer.clear();
        
        // Check if there's more in the queue
        return outgoingQueue.empty() ? SendResult::COMPLETE : SendResult::PENDING;
    }

    // ... other methods ...
};
```

The [buffer](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html) field stores incomplete incoming messages until we receive a newline character, while the outgoingQueue and currentSendBuffer work together to handle non-blocking message sending with proper backpressure management.

## **Managing Clients with Non-Blocking Sends**

Our ClientManager class handles client lifecycle and message broadcasting with asynchronous writes:

```cpp
class ClientManager {
private:
    std::unordered_map<int, Client*> clients;
    std::mutex mutex;

public:
    void addClient(Client* client) {
        std::lock_guard<std::mutex> lock(mutex);
        clients[client->fd] = client;
        std::cout << "Client added: " << client->nick << std::endl;
    }

    void sendMessage(int senderFd, const std::string& message) {
        std::lock_guard<std::mutex> lock(mutex);
        std::string senderNick;

        if (clients.find(senderFd) != clients.end()) {
            senderNick = clients[senderFd]->nick;
        } else {
            return;
        }

        std::string fullMessage = senderNick + ": " + message + "\r\n";

        for (auto& pair : clients) {
            if (pair.first != senderFd) {
                send(pair.first, fullMessage.c_str(), fullMessage.size(), 0);
            }
        }
    }

    // Other methods omitted for brevity...
}
```

Unlike the previous Go program that used mutexes, this implementation is designed for a single-threaded event loop, eliminating the need for thread synchronization. Instead, it uses non-blocking I/O with message queuing, an approach better suited to the epoll model.

## **Non-Blocking Writes with EPOLLOUT**

A significant improvement in our implementation is proper handling of non-blocking writes. When a client's socket buffer is full (which happens under load), we:

1. Queue the unsent data
    
2. Register for EPOLLOUT notifications
    
3. Resume sending when the socket becomes writable again
    

This is implemented in th[e handleClientWrite](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html) function:

```cpp
void handleClientWrite(int clientFd, ClientManager& clientManager) {
    Client* client = clientManager.getClient(clientFd);
    if (!client) {
        return;
    }

    // Try to send queued data
    SendResult result = client->trySendData();

    if (result == SendResult::COMPLETE) {
        // All data sent, stop monitoring for EPOLLOUT
        updateClientEpollEvents(clientFd, EPOLLIN | EPOLLET);
    } else if (result == SendResult::ERROR) {
        // Error occurred, close the connection
        handleClientDisconnection(clientFd, g_epollFd, clientManager);
    }
    // If result == SendResult::PENDING, keep monitoring for EPOLLOUT
}
```

This approach prevents blocking on socket writes and allows us to handle back pressure efficiently, making the server robust under high load conditions.

## **The Power of epoll: Understanding Edge vs. Level Triggering**

A key difference from the Go implementation is our use of epoll, which offers two notification modes:

1. **Level-triggered** (LT): Notifies when a file descriptor is ready (default mode[)](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html)
    
2. **Ed**[**g**](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html)**e-triggered** (ET): Notifies only when state changes from not-ready to ready
    

Our server uses a hybrid approach:

```cpp
// For server socket (accepting connections) - Level-triggered for reliability
ev.events = EPOLLIN;  
ev.data.fd = serverFd;

// For client sockets (data reading) - Edge-triggered for efficiency
ev.events = EPOLLIN | EPOLLET;  
ev.data.fd = clientFd;
```

This hybrid approach combines the reliability of level-triggered mode for accepting connections (never missing a connection) with the efficiency of edge-triggered mode for data reading (minimizing wake-ups).

## **The Complete Event Loop with EPOLLOUT Support**

The main event loop manages both read and write operations efficiently:

```cpp
vwhile (true) {
    int numEvents = epoll_wait(epollFd, events, MAX_EVENTS, -1);

    for (int i = 0; i < numEvents; i++) {
        const int currentFd = events[i].data.fd;
        const uint32_t currentEvents = events[i].events;

        if (currentFd == serverFd) {
            // New client connection
            handleNewConnection(serverFd, epollFd, clientManager);
        }
        else {
            // Client socket events
            if (currentEvents & EPOLLIN) {
                handleClientData(currentFd, epollFd, buffer, MAX_BUFFER_SIZE, clientManager);
            }

            if (currentEvents & EPOLLOUT) {
                handleClientWrite(currentFd, clientManager);
            }

            if (currentEvents & (EPOLLERR | EPOLLHUP)) {
                // Error or hang up
                handleClientDisconnection(currentFd, epollFd, clientManager);
            }
        }
    }
}
```

With edge-triggered epoll, we need to ensure we read all available data when notified. Our method correctly buffers incomplete messages, which is crucial for handling TCP streams. By watching for EPOLLOUT events, the server manages write operations efficiently, only when socket buffers have space, avoiding blocks with slow clients.

This single thread handles all network operations efficiently without blocking, enabling our server to scale to thousands of concurrent connections with minimal resource use.

## **Containerization with Docker**

For easy deployment and testing, we use Docker:

```dockerfile
FROM gcc:latest

WORKDIR /app

COPY . .

RUN apt-get update && apt-get install -y cmake
RUN cmake . && make

CMD ["./chat-server-cpp"]
```

## **Testing the Server**

You can connect to the server using telnet or netcat:

```bash
telnet localhost 12345
```

After connecting, try sending messages and using the `/nick` command to change your nickname:

```bash
/nick Alice
Hello everyone!
```

Other connected clients will see:

```bash
Alice: Hello everyone!
```

## **C++ vs. Go: A Performance and Design Comparison**

Having implemented chat servers in both Go and C++, it's interesting to compare the approaches:

**Go Implementation**

* **Concurrency Model**: Uses goroutines (one per client) with mutex synchronization
    
* **Memory Overhead**: Higher due to goroutine stacks (2KB-8KB each)
    
* **Code Complexity**: Lower thanks to goroutines abstraction
    
* **I/O Handling**: Blocking I/O abstracted by goroutines
    
* **Performance Characteristic**: CPU-efficient but more memory usage with many connections
    

**C++ Implementation**

* **Concurrency Model**: Single-threaded event loop with epoll
    
* **Memory Overhead**: Lower, only one thread regardless of connection count
    
* **Code Complexity**: Higher due to explicit event handling and buffer management
    
* **I/O Handling**: Fully non-blocking with explicit buffer management
    
* **Performance Characteristic**: Highly efficient for many connections with minimal activity
    

Both the C++ version and the Go version can handle tens of thousands of connections (the C10K problem). However, the C++ version uses less memory per connection because it doesn't have a goroutine stack.

## **Conclusion**

Building this chat server in C++ with epoll demonstrates a powerful alternative to Go's goroutine-based model. While Go's approach is more developer-friendly, the C++ implementation provides finer control over I/O operations and resource usage.

The most significant improvement in our C++ implementation is the sophisticated handling of non-blocking writes with message queuing. This approach ensures that slow clients don't block the server while maintaining correct message ordering—a critical requirement for any chat application.

Both implementations represent different architectural patterns:

* The Go version exemplifies the "thread-per-client" model (with lightweight goroutines)
    
* The C++ version showcases the "event loop" model popular in high-performance servers
    

For applications requiring maximum connection capacity with minimal resources, the C++ epoll approach has clear advantages. However, for scenarios where development speed and code maintainability are paramount, Go's goroutine model remains compelling.

The full source code for this C++ chat server is available on [GitHub](https://github.com/zhangbiao2009/chat-server-cpp). I encourage you to experiment with both implementations to better understand the tradeoffs between these different approaches to network programming.