---
title: "Build a Tiny Chat Server in Go"
datePublished: Sat Apr 12 2025 04:40:49 GMT+0000 (Coordinated Universal Time)
cuid: cm9dqa19n000h09kydwqnbis0
slug: go-chat-server
tags: go, network-programming

---

After reading antirez’s elegant [*Smallchat*](https://github.com/antirez/smallchat) program, I really appreciated it. It's a simple C program that shows how clean and minimal a chat server can be, and it stuck with me. That inspired me to write something similar in Go—keeping it small and easy to understand, but using Go’s built-in concurrency and networking features to make it even more approachable.

In this article, I'll walk you through the entire process step by step, sharing both the code and my thought process along the way.

By the end, we'll have a working chat server (just ~100 LOC) where you can use the `nc` (Netcat) command to join from different terminal windows, and any message sent by one client will be broadcast to all connected clients.

---

## What We're Building

The goal is to create a simple TCP-based chat server with the following features:

* **TCP Server:** Listens on a port and accepts client connections.
    
* **Multi-Client Support:** Handles multiple clients concurrently.
    
* **Broadcast Messaging:** When one client sends a message, it appears for all connected clients.
    
* **Nicknames:** Each client gets a random nickname when they join, and they can change it using the `/nick` command.
    

---

## Step 1: Representing a Connected Client

We need a way to represent each user that connects to our server. For this, we'll define a `Client` struct that contains:

* The TCP connection itself.
    
* A nickname for the user.
    

Each time a new connection is made, we'll create a `Client` object. To give each client a unique identity, we'll generate a random 4-character nickname.

```go
type Client struct {
	conn net.Conn
	nick string
}

func NewClient(conn net.Conn) *Client {
	client := &Client{conn: conn}
	client.nick = RandString(4)
	return client
}
```

We'll use a helper function to generate the random nickname:

```go
func RandString(n int) string {
	var letters = []rune("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")
	s := make([]rune, n)
	for i := range s {
		s[i] = letters[rand.Intn(len(letters))]
	}
	return string(s)
}
```

### How It Works

* **Client struct:** Holds the connection and nickname.
    
* **NewClient:** Instantiates a new client and assigns it a random nickname (e.g., "aB3k").
    
* **RandString:** Generates a string by randomly selecting characters from a predefined set.
    

---

## Step 2: Managing Multiple Clients

We need to keep track of all connected clients so that we can:

* Add a client when they connect.
    
* Remove a client when they disconnect.
    
* Broadcast messages from one client to all others.
    

We'll create a `ClientMgr` (Client Manager) struct that holds a map of clients. To allow safe concurrent access, we'll use a read-write mutex (`sync.RWMutex`). Note: think about why we use `sync.RWMutex` here?

```go
type ClientMgr struct {
	clientMap map[*Client]struct{}
	sync.RWMutex
}

func NewClientMgr() *ClientMgr {
	return &ClientMgr{clientMap: make(map[*Client]struct{})}
}
```

Methods to add, remove, and send messages:

```go
func (c *ClientMgr) AddClient(client *Client) {
	c.Lock()
	defer c.Unlock()
	c.clientMap[client] = struct{}{}
	fmt.Println("Client added:", client.nick)
}

func (c *ClientMgr) RemoveClient(client *Client) {
	c.Lock()
	defer c.Unlock()
	delete(c.clientMap, client)
	fmt.Println("Client removed:", client.nick)
}

func (c *ClientMgr) SendMessage(sender *Client, message string) {
	c.RLock()
	defer c.RUnlock()
	for client := range c.clientMap {
		if client == sender {
			continue
		}
		client.conn.Write([]byte(sender.nick + ": " + message))
	}
}
```

### How It Works

* **AddClient/RemoveClient:** Manage the map of active clients using locks to prevent race conditions.
    
* **SendMessage:** Loops through all clients (except the sender) to write the incoming message to each of their connections.
    

---

## Step 3: Starting the Server

The server must:

* Listen on a specific TCP port.
    
* Accept incoming client connections.
    
* Create a new `Client` for each connection.
    
* Spawn a separate goroutine to handle each client’s communication.
    

We'll use Go's `net.Listen` to start the server and then accept connections in a loop. For every new connection, we'll create a client, add it to our manager, and launch a goroutine to process incoming messages.

### The Code

```go
func main() {
	listenAddr := "127.0.0.1:12345"

	listener, err := net.Listen("tcp", listenAddr)
	if err != nil {
		fmt.Println("Error listening:", err)
		os.Exit(1)
	}
	defer listener.Close()

	fmt.Println("Server is listening on", listenAddr)

	for {
		conn, err := listener.Accept()
		if err != nil {
			fmt.Println("Error accepting connection:", err)
			continue
		}

		client := NewClient(conn)
		clientMgr.AddClient(client)

		go handleClient(client)
	}
}
```

### How It Works

* **net.Listen:** Starts the TCP server.
    
* **Infinite loop:** Continuously accepts new connections.
    
* **Goroutines:** Each client is handled concurrently so multiple users can chat simultaneously.
    

---

## Step 4: Handling Incoming Client Messages

Each connected client should be able to send messages to the server. The server must:

* Read messages from the client.
    
* Detect commands (e.g., `/nick` to change the nickname).
    
* Broadcast non-command messages to all other clients.
    

We'll use a buffered reader (`bufio.Reader`) to read input from the client line by line. A `bufio.Reader` can reduce unnecessary read system calls to improve performance. Note if a message starts with a slash `/`, we treat it as a command. Otherwise, we broadcast it as a message.

### The Code

```go
func handleClient(c *Client) {
	defer c.conn.Close()
	defer clientMgr.RemoveClient(c)

	reader := bufio.NewReader(c.conn)

	for {
		message, err := reader.ReadString('\n')
		if err != nil {
			fmt.Println("Connection closed:", err)
			break
		}

		if message[0] == '/' {
			// Handle commands (e.g., changing nickname)
			parts := strings.SplitN(message, " ", 2)
			cmd := parts[0]
			if cmd == "/nick" && len(parts) > 1 {
				oldNick := c.nick
				c.nick = strings.Trim(parts[1], "\n")
				fmt.Println("Client renamed from", oldNick, "to", c.nick)
			}
			continue
		}

		fmt.Print("Received:", message)
		clientMgr.SendMessage(c, message)
	}
}
```

### How It Works

* **Reading messages:** Uses `bufio.Reader` to read each line.
    
* **Command processing:** Checks if the message starts with a slash (`/`) and handles `/nick` to change the nickname.
    
* **Broadcasting:** Any non-command message is passed to the `SendMessage` method to be relayed to all other connected clients.
    

---

## Step 5: Trying It Out!

Now that our code is ready, it’s time to test our chat server.

### Run the Server

Open your terminal and run:

```bash
go run chat_server.go
```

You should see an output like:

```bash
Server is listening on 127.0.0.1:12345
```

### Connect with Netcat

Open another terminal window and type:

```bash
nc 127.0.0.1 12345
```

Repeat this step in additional terminal windows to simulate multiple clients.

### Chat Away

Now, type messages in one terminal. The messages will appear in the other connected clients. You can also change your nickname by typing:

```bash
/nick YourNewName
```

Your new nickname will be reflected in all future messages.

### Demo

As you can see, when one user sends a message, the server broadcasts it to all other users.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1744430179577/842d63fd-6510-452f-9d14-84a760e573d2.png align="center")

---

## Wrapping Up

In this project, we built a simple but effective chat server in Go that covers:

* **TCP Networking:** Listening and accepting client connections.
    
* **Concurrency:** Using goroutines to handle multiple clients at the same time.
    
* **Synchronization:** Safely managing shared resources with mutexes.
    
* **Simple Command Parsing:** Handling commands like `/nick`.
    

Check out the code on [GitHub](https://github.com/zhangbiao2009/chat-server-go). Happy coding and happy chatting!