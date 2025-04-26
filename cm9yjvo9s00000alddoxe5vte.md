---
title: "High-Performance Chat Server in C++ using Coroutines and epoll"
datePublished: Sat Apr 26 2025 18:24:51 GMT+0000 (Coordinated Universal Time)
cuid: cm9yjvo9s00000alddoxe5vte
slug: high-performance-chat-server-in-c-using-coroutines-and-epoll
tags: cpp, coroutines, tcp, backend-developments, epoll, network-programming

---

Building on my previous exploration of [a traditional event-driven chat server in C++ with epoll](https://davidzhang.hashnode.dev/cpp-chat-server), I've now implemented a modern solution using C++20 coroutines. While both approaches leverage epoll's efficiency, coroutines provide a more natural, synchronous-like coding style while maintaining all the benefits of non-blocking I/O. In this article, I'll walk through this implementation and compare it with the traditional event-loop approach.

## **The Promise of Coroutines: Combining Efficiency with Readability**

C++20 coroutines offer a revolutionary way to write asynchronous code that looks synchronous. This seemingly contradictory benefit gives us the best of both worlds:

1. The efficiency and scalability of non-blocking I/O
    
2. The readability and maintainability of synchronous code
    

First, Let’s see how to choose between these approaches:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1745690848546/843dd222-6ad0-496c-a4e4-17867e8c937c.png align="center")

Comparing to the traditional non-blocking epoll loop approach, the fundamental shift in our architecture is how we represent and handle client connections:

Instead of writing handlers for different events, we write coroutines that look like sequential code but can suspend execution, waiting for I/O operations to complete.

## **The Magic of EpollAwaitable: Bridging Coroutines and epoll**

The core innovation in our implementation is the [EpollAwaitable](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-sandbox/workbench/workbench.html) class, which serves as the bridge between the epoll event loop and C++ coroutines:

```cpp
struct EpollAwaitable {
    Client* client;
    bool is_write;

    EpollAwaitable(Client* c, bool write = false) : client(c), is_write(write) {}

    bool await_ready() noexcept { return false; }

    void await_suspend(std::coroutine_handle<> h) noexcept {
        client->set_coroutine_handle(h, is_write);
        if (is_write) client->update_epoll(true);
    }

    uint32_t await_resume() noexcept { return client->event; }
};
```

This class enables us to write code like:

```cpp
// Wait for read events in a blocking style
uint32_t events = co_await EpollAwaitable(client);
```

When the coroutine reaches this line, several things happen:

1. The coroutine's state is captured and suspended
    
2. The coroutine handle is stored in the client object
    
3. When epoll signals an event for this client, the coroutine resumes
    
4. Execution continues with the events returned by `await_resume()`
    

## **The Elegant Client Handler Coroutines**

The read and write operations are encapsulated in separate coroutines, providing a clean separation of concerns:

```cpp
Task handle_client_read(Client* client) {
    client->send("Welcome to the chat server!\r\n");
    std::string buffer;

    while (!client->is_closed()) {
        // Wait for read events in a blocking style
        uint32_t events = co_await EpollAwaitable(client);

        // Handle disconnection
        if (events & (EPOLLHUP | EPOLLERR)) {
            process_complete_lines(client, buffer);
            break;
        }

        // Handle read events
        if (events & EPOLLIN) {
            while (true) { // Read until buffer is empty
                char tmp[1024];
                ssize_t bytes = ::read(client->fd(), tmp, sizeof(tmp) - 1);

                if (bytes > 0) {
                    tmp[bytes] = '\0';
                    buffer += tmp;
                    process_complete_lines(client, buffer);
                } 
                else if (bytes == 0 || (bytes < 0 && errno != EAGAIN && errno != EWOULDBLOCK)) {
                    process_complete_lines(client, buffer);
                    co_return; // Connection closed or error
                }
                else {
                    break; // EAGAIN/EWOULDBLOCK - no more data available
                }
            }
        }
    }
}
```

This code handles reads in what appears to be a blocking manner but is actually fully non-blocking. When a client connects, we create a read coroutine that:

1. Waits for read events from epoll
    
2. Processes data when it's available
    
3. Suspends when no data is available
    
4. Continues where it left off when more data comes in
    

### Write Coroutine

The write coroutine is created on-demand when data can't be written immediately:

```cpp
Task handle_client_write(Client* client) {
    while (!client->is_closed() && client->has_write_data()) {
        // Wait for write events
        uint32_t events = co_await EpollAwaitable(client, true);

        if (events & (EPOLLHUP | EPOLLERR)) break;

        if (events & EPOLLOUT) {
            const std::string& data = client->front_message();
            ssize_t bytes = ::send(client->fd(), data.data(), data.size(), 0);

            // Handle write result
            if (bytes <= 0) {
                if (errno != EAGAIN && errno != EWOULDBLOCK) break; // Real error
            } else if (static_cast<size_t>(bytes) == data.size()) {
                client->pop_message();
            } else {
                client->update_front_message(data.substr(bytes));
            }
        }
    }

    // Disable write events when done
    if (!client->is_closed()) client->update_epoll(false);
}
```

## **Optimization: The "Write First" Pattern**

One notable optimization is that we attempt to write data immediately before creating a write coroutine:

```cpp
void send(const std::string& msg) {
    if (closed_) return;

    write_queue_.push(msg);

    // Try immediate write and create coroutine if needed
    while (!write_queue_.empty()) {
        const std::string& data = write_queue_.front();
        ssize_t bytes = ::send(fd_, data.data(), data.size(), 0);

        if (bytes <= 0) {
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                // Would block - start write coroutine if not already running
                if (!write_task_ || !write_handle_ || write_handle_.done()) {
                    start_coroutine(handle_client_write(this), true);
                }
                return;
            } else {
                // Real error
                close();
                return;
            }
        }

        if (static_cast<size_t>(bytes) == data.size()) {
            write_queue_.pop();
        } else {
            write_queue_.front() = data.substr(bytes);
            // Need to wait for more space in socket buffer
            if (!write_task_ || !write_handle_ || write_handle_.done()) {
                start_coroutine(handle_client_write(this), true);
            }
            return;
        }
    }
}
```

Since sockets are writable most of the time, this approach avoids creating unnecessary coroutines for the common case where data can be written immediately.

## **The Event Loop: Resuming Coroutines**

Our event loop is remarkably simple. Instead of complex handlers, we just:

1. Wait for epoll events
    
2. Store the event flags in the client object
    
3. Resume the appropriate coroutine
    

```cpp
void run() {
    if (!running_) return;

    const int MAX_EVENTS = 64;
    struct epoll_event events[MAX_EVENTS];

    while (running_) {
        int n = epoll_wait(epoll_fd_, events, MAX_EVENTS, 100);

        // ... error handling ...

        for (int i = 0; i < n; i++) {
            // Handle server socket (new connections)
            if (events[i].data.fd == server_fd_) {
                accept_connections();
                continue;
            }

            // Handle client events
            Client* client = static_cast<Client*>(events[i].data.ptr);
            client->event = events[i].events;

            // Resume read coroutine if active
            auto read_handle = client->read_handle();
            if (read_handle && !read_handle.done()) {
                read_handle.resume();
            }

            // Resume write coroutine if active
            auto write_handle = client->write_handle();
            if (write_handle && !write_handle.done()) {
                write_handle.resume();
            }

            // Clean up completed clients
            if (read_handle && read_handle.done()) {
                clients_.erase(client->fd());
            }
        }
    }
}
```

## **Performance Characteristics**

In terms of performance, the coroutine implementation can efficiently handle tens of thousands of connections because it also uses epoll for effective I/O multiplexing. While it might use a bit more memory due to coroutine frames, this is usually minor compared to other memory usage in a typical server.

## **Advanced Pattern: On-Demand Coroutines**

A notable optimization in our coroutine implementation is that we create the write coroutine only when needed:

1. When sending data, we first try to write immediately
    
2. Only if the socket buffer is full do we create a write coroutine
    
3. The coroutine automatically destroys itself once all data is written
    

This "on-demand coroutine" pattern results in very efficient resource usage, creating coroutines only when their special capabilities are needed.

## **Code Size and Complexity Comparison**

While the coroutine implementation is slightly larger (~450 LOC) in terms of line count, it has less boilerplate code and more semantic code. The additional lines are primarily due to the coroutine infrastructure that simplifies the actual business logic.

## **Conclusion: Which Approach Should You Choose?**

The choice between traditional event-driven and coroutine-based approaches depends on several factors:

1. **Language version constraints**: If you're limited to pre-C++20, the traditional approach is your only option.
    
2. **Development team familiarity**: Coroutines introduce new concepts that may have a learning curve for some developers.
    
3. **Code maintainability priority**: For complex logic flows, coroutines offer significant maintainability benefits because they help avoid "callback hell".
    
4. **Error handling requirements**: Coroutines allow for more natural error handling patterns.
    

Both implementations demonstrate how to build efficient, scalable chat servers in C++. The traditional approach has served us well for years, but coroutines represent the future of asynchronous programming in C++, offering a compelling combination of efficiency and readability.

The complete source code for the coroutine-based chat server is available on [GitHub](https://github.com/zhangbiao2009/chat-server-cpp/blob/main/chat_server_coroutine.cpp), alongside the traditional implementation. I encourage readers to compare both implementations and try running them to experience the differences firsthand.

For most new projects where C++20 is available, I would recommend the coroutine-based approach due to its improved readability and maintainability. The traditional event-driven approach remains a solid choice when working with older codebases or when compiler support for C++20 coroutines is limited.