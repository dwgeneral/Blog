> 空结构体占用内存空间吗? 我们使用 unsafe.Sizeof 可以计算出一个数据结构实例占用的字节数

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	fmt.Println(unsafe.Sizeof(struct{}{})) // 0
}
```

你会发现输出是0, 也就是说空结构体是不占用任何内存空间的. 基于这个特性, 我们可以在一些场景下使用空结构体来节约内存空间, 并且语义表达更清晰.

### 实现集合(Set)

Go 语言标准库没有提供 Set 的实现，通常使用 map 来代替。事实上，对于集合来说，只需要 map 的键，而不需要值。即使是将值设置为 bool 类型，也会多占据 1 个字节，那假设 map 中有一百万条数据，就会浪费 1MB 的空间。
因此呢，将 map 作为集合(Set)使用时，可以将值类型定义为空结构体，仅作为占位符使用即可。

```go
type Set map[string]struct{}

func (s Set) Has(key string) bool {
	_, ok := s[key]
	return ok
}

func (s Set) Add(key string) {
	s[key] = struct{}{}
}

func (s Set) Delete(key string) {
	delete(s, key)
}

func main() {
	s := make(Set)
	s.Add("beijing")
	s.Add("shanghai")
	fmt.Println(s.Has("beijing"))
	fmt.Println(s.Has("shanghai"))
}
```

### channel 信号

有时候使用 channel 不需要发送任何的数据，只用来通知子协程(goroutine)执行任务，或只用来控制协程并发度。这种情况下，使用空结构体作为占位符就非常合适了。

```
func do(ch chan struct{}) {
	<-ch
	fmt.Println("do something")
	close(ch)
}

func main() {
	ch := make(chan struct{})
	go do(ch)
	ch <- struct{}{}
}

```

### 含有方法的任意结构体

```
type Apple struct{}

func (a Apple) eat() {
	fmt.Println("nom.......")
}

func (a Apple) drop() {
	fmt.Println("xiu.......")
}
```

在部分场景下，结构体只包含方法，不包含任何的字段。例如上面例子中的 Apple，在这种情况下，Apple 事实上可以用任何的数据结构替代。

```
type Apple string
type Apple int
```

无论是 string 还是 int 都会浪费额外的内存，因此呢，这种情况下，声明为空结构体是最合适的。

