
### 🔍  **Go (Golang)**

a **basic TCP server and client demo** in Go that:

1. Connects from client to server.
2. Client sends data.
3. Client drops the connection.
4. Server reacts to the disconnection.

---

## ✅ 1. Go TCP Server

```go
// server.go
package main

import (
	"bufio"
	"fmt"
	"net"
)

func main() {
	listener, err := net.Listen("tcp", ":9000")
	if err != nil {
		panic(err)
	}
	defer listener.Close()
	fmt.Println("Server is listening on port 9000...")

	for {
		conn, err := listener.Accept()
		if err != nil {
			fmt.Println("Error accepting:", err)
			continue
		}
		fmt.Println("Client connected:", conn.RemoteAddr())

		go handleConnection(conn)
	}
}

func handleConnection(conn net.Conn) {
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

---

## ✅ 2. Go TCP Client

```go
// client.go
package main

import (
	"bufio"
	"fmt"
	"net"
	"os"
	"time"
)

func main() {
	conn, err := net.Dial("tcp", "localhost:9000")
	if err != nil {
		panic(err)
	}
	defer conn.Close()

	fmt.Println("Connected to server. Type messages or 'exit' to disconnect.")

	go func() {
		for {
			// This simulates the client dying suddenly (no FIN)
			time.Sleep(10 * time.Second)
			fmt.Println("Simulating crash: exiting without closing connection.")
			os.Exit(1)
		}
	}()

	reader := bufio.NewReader(os.Stdin)
	for {
		fmt.Print("> ")
		text, _ := reader.ReadString('\n')
		if text == "exit\n" {
			fmt.Println("Gracefully closing connection.")
			return
		}
		_, err := conn.Write([]byte(text))
		if err != nil {
			fmt.Println("Write error:", err)
			return
		}
	}
}
```

---

## 🧪 What You'll See

### Case 1: Normal Disconnect

* Run server: `go run server.go`
* Run client: `go run client.go`
* Type some messages.
* Type `exit` and hit Enter.
* Server will print:

  ```
  Client disconnected: [client IP]
  ```

### Case 2: Simulated Crash

* Run client again.
* Wait 10 seconds.
* It exits with `os.Exit(1)` (no graceful shutdown).
* Server prints:

  ```
  Read error: EOF
  Client disconnected: [client IP]
  ```

OR you might see:

```
Read error: read tcp ...: connection reset by peer
```

That depends on your OS’s behavior.

---

## 🧠 Notes

* `ReadString('\n')` blocks until data or error.
* If client closes normally → `err == EOF`.
* If client crashes or loses network → error like `"connection reset by peer"` or timeout.
* You can add TCP keep-alive if you want to detect dead connections more quickly.

