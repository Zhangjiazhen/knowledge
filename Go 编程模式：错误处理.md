### JAVA的错误处理
- 正常逻辑的代码和错误处理、资源回收是分开的，可读性较强
- 错误不能忽略，只是显示忽略（在catch中不做处理）

### GO的错误处理

- go支持多个返回值，一般都是把错误放在第二个返回值中，语义上比较清晰，如忽略可用_显示忽略。
- go的异常返回的error是个函数，可以自定义异常类型。

### 资源清理

go的资源清理一般放在defer中处理，另外，一个方法中多个defer的处理顺序和栈是一样的，即从下往上处理。下面是一个示例：

```go
func Close(c io.Closer) {
  err := c.Close()
  if err != nil {
    log.Fatal(err)
  }
}

func main() {
  r, err := Open("a")
  if err != nil {
    log.Fatalf("error opening 'a'\n")
  }
  defer Close(r) // 使用defer关键字在函数退出时关闭文件。

  r, err = Open("b")
  if err != nil {
    log.Fatalf("error opening 'b'\n")
  }
  defer Close(r) // 使用defer关键字在函数退出时关闭文件。
}
```

### Error Check Hell

go的错误处理不像java，如果有多个错误判断，if语句可以写到吐，示例如下：

```go
func parse(r io.Reader) (*Point, error) {

    var p Point

    if err := binary.Read(r, binary.BigEndian, &p.Longitude); err != nil {
        return nil, err
    }
    if err := binary.Read(r, binary.BigEndian, &p.Latitude); err != nil {
        return nil, err
    }
    if err := binary.Read(r, binary.BigEndian, &p.Distance); err != nil {
        return nil, err
    }
    if err := binary.Read(r, binary.BigEndian, &p.ElevationGain); err != nil {
        return nil, err
    }
    if err := binary.Read(r, binary.BigEndian, &p.ElevationLoss); err != nil {
        return nil, err
    }
}
```

我们可以用函数式编程解决这个问题，示例如下：

```GO
func parse(r io.Reader) (*Point, error) {
    var p Point
    var err error
    read := func(data interface{}) {
        if err != nil {
            return
        }
        err = binary.Read(r, binary.BigEndian, data)
    }

    read(&p.Longitude)
    read(&p.Latitude)
    read(&p.Distance)
    read(&p.ElevationGain)
    read(&p.ElevationLoss)

    if err != nil {
        return &p, err
    }
    return &p, nil
}
```

但是上面的写法有个内部方法，还有个error变量，看起来还不够干净。我们可以从以下代码中获得一些灵感：

```go
scanner := bufio.NewScanner(input)

for scanner.Scan() {
    token := scanner.Text()
    // process token
}

if err := scanner.Err(); err != nil {
    // process the error
}
```

首先定义一个结构体和成员方法：

```go
type Reader struct {
    r   io.Reader
    err error
}

func (r *Reader) read(data interface{}) {
    if r.err == nil {
        r.err = binary.Read(r.r, binary.BigEndian, data)
    }
}
```

代码就可以变成以下这样：

```go
func parse(input io.Reader) (*Point, error) {
    var p Point
    r := Reader{r: input}

    r.read(&p.Longitude)
    r.read(&p.Latitude)
    r.read(&p.Distance)
    r.read(&p.ElevationGain)
    r.read(&p.ElevationLoss)

    if r.err != nil {
        return nil, r.err
    }

    return &p, nil
}
```

可以看到，整个代码就变得很清晰。

流式接口也可以变成以下这样了：

```go
package main

import (
  "bytes"
  "encoding/binary"
  "fmt"
)

// 长度不够，少一个Weight
var b = []byte {0x48, 0x61, 0x6f, 0x20, 0x43, 0x68, 0x65, 0x6e, 0x00, 0x00, 0x2c} 
var r = bytes.NewReader(b)

type Person struct {
  Name [10]byte
  Age uint8
  Weight uint8
  err error
}
func (p *Person) read(data interface{}) {
  if p.err == nil {
    p.err = binary.Read(r, binary.BigEndian, data)
  }
}

func (p *Person) ReadName() *Person {
  p.read(&p.Name) 
  return p
}
func (p *Person) ReadAge() *Person {
  p.read(&p.Age) 
  return p
}
func (p *Person) ReadWeight() *Person {
  p.read(&p.Weight) 
  return p
}
func (p *Person) Print() *Person {
  if p.err == nil {
    fmt.Printf("Name=%s, Age=%d, Weight=%d\n",p.Name, p.Age, p.Weight)
  }
  return p
}

func main() {   
  p := Person{}
  p.ReadName().ReadAge().ReadWeight().Print()
  fmt.Println(p.err)  // EOF 错误
}
```

不过这种方法也有局限，仅仅针对于同一个对象的多次处理可以使用这个套路，如果是多个对象，还得使用多个if处理异常。

### 包装错误

这里有个第三方库，基本上是事实标准，有了它，就可以不用只是把错误返回给上层了，而是做统一处理。示例如下：

```go
import "github.com/pkg/errors"

//错误包装
if err != nil {
    return errors.Wrap(err, "read failed")
}

// Cause接口
switch err := errors.Cause(err).(type) {
case *MyError:
    // handle specifically
default:
    // unknown error
}
```

参考[《Go 编程模式：错误处理》](https://time.geekbang.org/column/article/332602)