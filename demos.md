Awesome — let’s build on this and give you **several focused demos** to help you understand:

1. **Basic Echo Server/Client (Done)**
2. **Simulated Client Crash (Already Included)**
3. **TCP Keep-Alive in Go**
4. **Manual Disconnect with Netcat**
5. **Client Timeouts and Application-Level Heartbeats**
6. **Detecting Dead Clients via Write Failures**

Let’s walk through these one by one.

---

## ✅ 3. TCP Keep-Alive in Go (Server-Side)

You can configure the server socket to use **TCP keep-alive**, which helps detect dead connections over time.

### 🔧 Updated `handleConnection` with TCP Keep-Alive:

```go
func handleConnection(conn net.Conn) {
	tcpConn, ok := conn.(*net.TCPConn)
	if ok {
		tcpConn.SetKeepAlive(true)
		tcpConn.SetKeepAlivePeriod(30 * time.Second) // OS sends a probe every 30s
	}

	defer func() {
		fmt.Println("Client disconnected:", conn.RemoteAddr())
		conn.Close()
	}()

	reader := bufio.NewReader(conn)
	for {
		message, err := reader.ReadString('\n')
		if err != nil {
			fmt.Println("Read error:", err)
			return
		}
		fmt.Print("Received: ", message)
	}
}
```

⏱️ Now, if the client crashes or disconnects silently (e.g., network is unplugged), the server will eventually get notified when keep-alive probes fail — depending on OS TCP settings.

---

## ✅ 4. Using `netcat` (nc) to Simulate a Client

Instead of a Go client, you can use the shell to simulate clients.

### 🖥️ Command:

```bash
nc localhost 9000
```

* Type messages → server sees them.
* Hit `Ctrl+C` → this is an **abrupt disconnect**, no proper shutdown.

Server sees:

```
Read error: EOF
Client disconnected: 127.0.0.1:xxxxx
```

If you unplug the network or kill `nc` abruptly, server may see:

```
Read error: read tcp ...: connection reset by peer
```

---

## ✅ 5. Heartbeat (Ping-Pong) to Detect Client Timeout

Sometimes TCP keep-alive is too slow. You can build your own **ping/heartbeat system**.

### Server that expects heartbeats every 5 seconds:

```go
func handleConnection(conn net.Conn) {
	defer conn.Close()
	fmt.Println("Client connected:", conn.RemoteAddr())

	timeout := 10 * time.Second
	conn.SetReadDeadline(time.Now().Add(timeout))

	reader := bufio.NewReader(conn)

	for {
		message, err := reader.ReadString('\n')
		if err != nil {
			fmt.Println("Connection timeout or error:", err)
			return
		}
		fmt.Print("Received: ", message)
		conn.SetReadDeadline(time.Now().Add(timeout)) // Reset timer
	}
}
```

### Client sending pings:

```go
func main() {
	conn, err := net.Dial("tcp", "localhost:9000")
	if err != nil {
		panic(err)
	}
	defer conn.Close()

	ticker := time.NewTicker(5 * time.Second)
	defer ticker.Stop()

	for {
		<-ticker.C
		_, err := conn.Write([]byte("ping\n"))
		if err != nil {
			fmt.Println("Failed to send ping:", err)
			return
		}
	}
}
```

⏰ If the client crashes or stops sending pings, the server times out in \~10 seconds.

---

## ✅ 6. Detecting Client Disconnect on Write (not just read)

Let’s say the server only sends data. You can detect disconnects on **write**.

```go
func handleConnection(conn net.Conn) {
	defer conn.Close()
	fmt.Println("Client connected:", conn.RemoteAddr())

	ticker := time.NewTicker(2 * time.Second)
	defer ticker.Stop()

	for range ticker.C {
		_, err := conn.Write([]byte("Hello\n"))
		if err != nil {
			fmt.Println("Write failed:", err)
			return
		}
	}
}
```

Try running this server. Then:

* Kill the client.
* Unplug the client’s network.
* You’ll see a write error like:

```
Write failed: write tcp ...: broken pipe
```

---

## 🧪 BONUS: Combine It All

You can mix:

* TCP keep-alive
* Application-layer heartbeats
* Timeout detection on `read` and `write`

To build **resilient real-world servers** that know when the client is alive or gone.

---

## 🛠️ What Would You Like Next?

* Add logging with timestamps?
* Simulate many clients?
* Build a chat room with multiple clients and broadcasting?
* Test behavior under dropped Wi-Fi or offline mode?

Let me know what you're most curious about and I’ll give you the exact demo.
