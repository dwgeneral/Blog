#### 什么是 TCP 粘包问题

> 让我们先来看一个TCP粘包的例子

我们先使用 Go语言的 net 包实现一个TCP通信服务
```go
// tcp/server/main.go
package main

import (
	"io"
	"bufio"
	"fmt"
	"net"
)

func main() {
	listen, err := net.Listen("tcp", "127.0.0.1:3000")
	if err != nil {
		fmt.Println("listen failed, err: ", err)
		return
	}
	fmt.Println("Server started, Listening 3000....")
	defer listen.Close()
	for {
		conn, err := listen.Accept()
		if err != nil {
			fmt.Println("accept failed, err: ", err)
			continue
		}
		go process(conn)
	}
}

func process(conn net.Conn) {
	defer conn.Close()
	reader := bufio.NewReader(conn)
	var buf [1024]byte
	for {
		n, err := reader.Read(buf[:])
		if err == io.EOF { break }
		if err != nil {
			fmt.Println("read from client failed, err: ", err)
			break
		}
		recvStr := string(buf[:n])
		fmt.Println("收到client端发来的数据：", recvStr)
	}
}
```
然后启动 Server `$ go run server/main.go`

```go
// tcp/client/main.go
package main

import (
	"fmt"
	"net"
)

func main() {
	conn, err := net.Dial("tcp", "127.0.0.1:3000")
	if err != nil {
		fmt.Println("err: ", err)
		return
	}
	defer conn.Close()
	for i := 0; i < 20; i++ {
		msg := `Welcome to My Space!`
		conn.Write([]byte(msg))
	}
}
```
然后启动 Client `$ go run client/main.go`

你的 server 端控制台应该会出现类似下面的输出
```shell
Server started, Listening 3000....
收到client端发来的数据： Welcome to My Space!Welcome to My Space!Welcome to My Space!Welcome to My Space!Welcome to My Space!Welcome to My Space!Welcome to My Space!
收到client端发来的数据： Welcome to My Space!Welcome to My Space!Welcome to My Space!Welcome to My Space!Welcome to My Space!Welcome to My Space!Welcome to My Space!Welcome to My Space!Welcome to My Space!Welcome to My Space!Welcome to My Space!Welcome to My Space!
收到client端发来的数据： Welcome to My Space!
```

客户端分20次发送的数据，在服务端并没有成功输出20次，而是多条数据“粘”到了一起。这种现象就是TCP粘包

#### 为什么出现TCP粘包

- TCP 传输的是字节流，在保持长连接时可以进行多次收发
- 为了尽量减少I/O的消耗，可能会发生如下情况
- 由 Nagle 算法造成的发送端的粘包：Nagle算法是一种改善网络传输效率的算法。简单来说就是当我们提交一段数据给TCP发送时，TCP并不立刻发送此段数据，而是等待一小段时间看看在等待期间是否还有要发送的数据，若有则会一次把这两段数据发送出去。
- 接收端接收不及时造成的接收端粘包：TCP会把接收到的数据存在自己的缓冲区中，然后通知应用层取数据。当应用层由于某些原因不能及时的把TCP的数据取出来，就会造成TCP缓冲区中存放了几段数据。

#### 如何解决

出现”粘包”的关键在于接收方不确定将要传输的数据包的大小，因此我们可以对数据包进行封包和拆包的操作。

封包：封包就是给一段数据加上包头，这样一来数据包就分为包头和包体两部分内容了。包头部分的长度是固定的，并且它存储了包体的长度，根据包头长度固定以及包头中含有包体长度的变量就能正确的拆分出一个完整的数据包。

通信双方约定好一个协议，比如数据包的前4个字节为包头，包头中存储包体数据长度。

```go
// TCP 数据包协议

package proto

import (
	"bufio"
	"encoding/binary"
	"bytes"
)

func Encode(message string) ([]byte, error) {
	// 读取消息的长度，转换成int32类型(占4个字节)
	var length = int32(len(message))
	var pkg = new(bytes.Buffer)
	// 写入消息头
	err := binary.Write(pkg, binary.LittleEndian, length)
	if err != nil {
		return nil, err
	}
	// 写入消息实体
	err = binary.Write(pkg, binary.LittleEndian, []byte(message))
	if err != nil {
		return nil, err
	}
	return pkg.Bytes(), nil
}

func Decode(reader *bufio.Reader) (string, error) {
	// 读取消息长度
	lengthByte, _ := reader.Peek(4) // 读取前4个字节的数据
	lengthBuff := bytes.NewBuffer(lengthByte)
	var length int32
	err := binary.Read(lengthBuff, binary.LittleEndian, &length)
	if err != nil {
		return "", err
	}
	// Buffered 返回缓冲中现有的可读取的字节数
	if int32(reader.Buffered()) < length+4 {
		return "", err
	}
	// 读取真正的消息数据
	pack := make([]byte, int(4+length))
	_, err = reader.Read(pack)
	if err != nil {
		return "", err
	}
	return string(pack[4:]), nil
}
```

接下来在服务端和客户端分别使用上面定义的proto包的Decode和Encode函数处理数据。

```go
// tcp/server/main.go
func process(conn net.Conn) {
	defer conn.Close()
	reader := bufio.NewReader(conn)
	for {
		msg, err := proto.Decode(reader)
		if err == io.EOF { return }
		if err != nil {
			fmt.Println("decode msg failed, err: ", err)
			return
		}
		fmt.Println("收到client端发来的数据：", msg)
	}
}
```

```go
// tcp/client/main.go
func main() {
	conn, err := net.Dial("tcp", "127.0.0.1:3000")
	if err != nil {
		fmt.Println("dial err: ", err)
		return
	}
	defer conn.Close()
	for i := 0; i < 20; i++ {
		msg := `Welcome to My Space!`
		data, err := proto.Encode(msg)
		if err != nil {
			fmt.Println("encode msg err: ", err)
			return
		}
		conn.Write(data)
	}
}
```

依次启动 Server 和 Client，粘包问题解决！

```shell
Server started, Listening 3000....
收到client端发来的数据： Welcome to My Space!
收到client端发来的数据： Welcome to My Space!
收到client端发来的数据： Welcome to My Space!
收到client端发来的数据： Welcome to My Space!
收到client端发来的数据： Welcome to My Space!
收到client端发来的数据： Welcome to My Space!
收到client端发来的数据： Welcome to My Space!
收到client端发来的数据： Welcome to My Space!
收到client端发来的数据： Welcome to My Space!
收到client端发来的数据： Welcome to My Space!
收到client端发来的数据： Welcome to My Space!
收到client端发来的数据： Welcome to My Space!
收到client端发来的数据： Welcome to My Space!
收到client端发来的数据： Welcome to My Space!
收到client端发来的数据： Welcome to My Space!
收到client端发来的数据： Welcome to My Space!
收到client端发来的数据： Welcome to My Space!
收到client端发来的数据： Welcome to My Space!
收到client端发来的数据： Welcome to My Space!
收到client端发来的数据： Welcome to My Space!
```

