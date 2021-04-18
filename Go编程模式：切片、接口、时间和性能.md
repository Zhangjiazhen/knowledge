### Slice
Slice的中文名叫切片，底层是一个结构体，而不是数组，结构体定义如下：
```
type slice struct {
    array unsafe.Pointer //指向存放数据的数组指针
    len   int            //长度有多大
    cap   int            //容量有多大
}
```
先来看一段代码

![image](image/WX20210418-112107.png)
```
foo = make([]int, 5)
foo[3] = 42
foo[4] = 100

bar  := foo[1:4]
bar[1] = 99
```
代码的意思是foo创建了一个长度为5的数组，分别对索引 为3和4的元素赋值，对foo做切片后赋值给bar，对bar[1]赋值。

![image](https://note.youdao.com/yws/public/resource/15129833b3c621081a70cb80262fec54/xmlnote/5B52978B97194FCAA4CFF9B7F6343858/11104)

结合以上图片，因为切片是共享内存的，所以两个切片修改数值会互相影响。

再来看一段代码
```
a := make([]int, 32)
b := a[1:16]
a = append(a, 1)
a[2] = 42
```

在这段代码中，把 a[1:16] 的切片赋给 b ，此时，a 和 b 的内存空间是共享的，然后，对 a 做了一个 append()的操作，这个操作会让 a 重新分配内存，这就会导致 a 和 b 不再共享，如下图所示：

![image](https://note.youdao.com/yws/public/resource/15129833b3c621081a70cb80262fec54/xmlnote/EE9F783BBAAD4F3696D6952DC35A9953/11090)

append()这个函数在 cap 不够用的时候，就会重新分配内存以扩大容量，如果够用，就不会重新分配内存了！

看以下代码：
```
func main() {
    path := []byte("AAAA/BBBBBBBBB")
    sepIndex := bytes.IndexByte(path,'/')

    dir1 := path[:sepIndex]
    dir2 := path[sepIndex+1:]

    fmt.Println("dir1 =>",string(dir1)) //prints: dir1 => AAAA
    fmt.Println("dir2 =>",string(dir2)) //prints: dir2 => BBBBBBBBB

    dir1 = append(dir1,"suffix"...)

    fmt.Println("dir1 =>",string(dir1)) //prints: dir1 => AAAAsuffix
    fmt.Println("dir2 =>",string(dir2)) //prints: dir2 => uffixBBBB
}
```
切片dir1和dir2存在共享内存的片段，dir1对元素的修改涉及到共享内存区域，同时影响了dir2的输出内容。

如何避免以上情况呢？只需要修改一行代码，把
```
dir1 := path[:sepIndex]
```
修改为
```
dir1 := path[:sepIndex:sepIndex]
```
新的代码使用了Full Slice Expression，最后一个参数叫“Limited Capacity”，于是，后续的 append() 操作会导致重新分配内存。

### 性能提示
如果需要把数字转换成字符串，使用 strconv.Itoa() 比 fmt.Sprintf() 要快一倍左右。

尽可能避免把String转成[]Byte ，这个转换会导致性能下降。

如果在 for-loop 里对某个 Slice 使用 append()，请先把 Slice 的容量扩充到位，这样可以避免内存重新分配以及系统自动按 2 的 N 次方幂进行扩展但又用不到的情况，从而避免浪费内存。

使用StringBuffer 或是StringBuild 来拼接字符串，性能会比使用 + 或 +=高三到四个数量级。

尽可能使用并发的 goroutine，然后使用 sync.WaitGroup 来同步分片操作
。
避免在热代码中进行内存分配，这样会导致 gc 很忙。尽可能使用  sync.Pool 来重用对象。

使用 lock-free 的操作，避免使用 mutex，尽可能使用 sync/Atomic包（关于无锁编程的相关话题，可参看《无锁队列实现》或《无锁 Hashmap 实现》）。

使用 I/O 缓冲，I/O 是个非常非常慢的操作，使用 bufio.NewWrite() 和 bufio.NewReader() 可以带来更高的性能。

对于在 for-loop 里的固定的正则表达式，一定要使用 regexp.Compile() 编译正则表达式。性能会提升两个数量级。

如果你需要更高性能的协议，就要考虑使用 protobuf 或 msgp 而不是 JSON，因为 JSON 的序列化和反序列化里使用了反射。

你在使用 Map 的时候，使用整型的 key 会比字符串的要快，因为整型比较比字符串比较要快。
