Go常见的坑以及高性能Go编程
代码的稳健性、高性能、可读性是我们每一位coder必须去追求的目标，也是coding的基本功，本文结合Go语言的特性，以及自己在写Go项目中做的总结，从Go常见的数据结构、内存管理、并发等方面做了相关总结

说明：本文相关代码的验证环境：GOARCH="arm64" GOOS="darwin" GOVERSION="go1.16.15"
一. 常见的坑
1.1  数据结构
1. 在函数调用过程中，数组是值传递pass-by-value，必要时必须使用slice。因为slice是引用类型，在slice传参时，只会复制slice的Data指针和len、cap，形参和实参的slice使用的是同一个底层数组。
2. map是一种hash表实现，每次遍历的顺序都可能不一样。
3. 切片会导致整个数组被锁定，导致底层数组无法及时释放内存，如果底层数组过大，会对内存产生极大的压力。
错误示例：
func test() {
    fileHeaderMap := make(map[string][]byte)

    for i := 0; i < 5; i++ {
        name := "/data/test/file_"+strconv.Itoa(i)
        data, err := ioutil.ReadFile(name)
        if err != nil {
            fmt.Println(err)
            continue
        }
        fileHeaderMap[name] = data[:1]
   }
   // do some thing
}
   正确示例：解决办法是将结果克隆一份，这样可以释放底层数组
func test() {
    fileHeaderMap := make(map[string][]byte)

    for i := 0; i < 5; i++ {
        name := "/data/test/file_"+strconv.Itoa(i)
        data, err := ioutil.ReadFile(name)
        if err != nil {
            fmt.Println(err)
            continue
        }
        fileHeaderMap[name] = append([]byte{}, data[:1]...)
    }

    // do some thing
}

4. 当函数的可变参数是空接口时，传入空接口的切片时需要注意参数展开的问题
func main() {
    var a = []interface{}{111, 222, 333}

    fmt.Println(a)
    fmt.Println(a...)
}
// 不管是否展开，编译器都会编译通过，但是输出是不同的：
//print：
[111 222 333]
111 222 333
5. 对于切片的操作，会操作同一个底层数组，因此对于一个切片的修改操作，会影响到整个数组。另外由于string 在go中是immutable，对于同一个字符串的两个变量，Go做了内存优化，会使用的相同的底层数组

func main() {
    slice := []int{1, 2, 3, 4, 5}
    slice1 := slice[:2]
    slice2 := slice[:4]
    fmt.Println("slice: ", slice, ",slice1: ", slice1, ",slice2: ", slice2)
    slice2[0] = 6
    fmt.Println("slice: ", slice, ",slice1: ", slice1, ",slice2: ", slice2)
    
    string1 := "go hello word"
    string2 := "go hello word"
    fmt.Printf("string1 data addr: %d \n", (*reflect.StringHeader)(unsafe.Pointer(&string1)).Data)
    fmt.Printf("string2 data addr: %d \n", (*reflect.StringHeader)(unsafe.Pointer(&string2)).Data)
}

//print:
slice:  [1 2 3 4 5] ,slice1:  [1 2] ,slice2:  [1 2 3 4]
slice:  [6 2 3 4 5] ,slice1:  [6 2] ,slice2:  [6 2 3 4]
string1 data addr: 4333475958 
string2 data addr: 4333475958

1.2 Go语言特性相关
1. Go中的recover捕获的是祖父级调用时候的panic，直接调用时不会有任何效果，必须在defer函数中调用才有效果。
func test() err error  {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("[ERROR] Process exception. Panic: %v", r)
        }
    }()
    // do some thing
    panic("some error happend")
}
2. 不同goroutine之间不满足顺序一致性的内存模型，需要使用显示的同步：如 channel 
var testMsg string
var done = make(chan struct{})

func initMsg() {
    testMsg = "go hello world"
    done <- struct{}{}
}

func main() {
    go initMsg()
    <-done
    println(testMsg)
}
3. 闭包的错误使用，导致使用同一个变量
错误示例：
func main() {
    for i := 0; i < 5; i++ {
        defer func() {
            println(i)
        }()
    }
}
print:
5
5
5
5
5
正确示例：
func main() {
    for i := 0; i < 5; i++ {
        defer func(i int) {
            println(i)
        }(i)
        // or
        //i := i
        //defer func(){
        //    println(i)
        //}()
    }
}

//print:
4
3
2
1
0
4. 因为defer在函数退出时才会执行，因此不能在循环内部执行defer，否则会导致相关defer操作调用延迟，比如fd会延迟关闭，应将defer包装在func中
错误示例：
func test() {
    for i := 0; i < 5; i++ {
        f, err := os.Open("/data/test/file_"+strconv.Itoa(i))
        if err != nil {
            fmt.Println(err)
            continue
        }
        defer f.Close()
        // do some thing
    }
    // do some thing
}
正确示例：
func test() {
    for i := 0; i < 5; i++ {
        func() {
            f, err := os.Open("/data/test/file_"+strconv.Itoa(i))
            if err != nil {
                fmt.Println(err)
                return
            }
            defer f.Close()
            // do some thing
        }()
    }
    // do some thing
}
5.  Goroutine 泄漏：
Go语言是带内存自动回收的特性，因此内存一般不会泄漏。但是Goroutine却存在泄漏的情况，同时泄漏的Goroutine引用的内存也同样无法被回收。因此可通过context包或者退出的channel来避免Goroutine的泄漏。
错误示例：
func main() {
    ch := func() <-chan int {
        ch := make(chan int)
        go func() {
            for i := 0; ; i++ {
                ch <- i
            }
        } ()
        return ch
    }()

    for value := range ch {
        fmt.Println(value)
        if value == 10 {
            break
        }
    }
    // do some thing
}
上面的程序中后台Goroutine向channel输入整数，main函数中输出该整数。但是当值为10时， break跳出for循环，后台Goroutine就处于无法被回收的状态了
正确示例：
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    // stopChan := make(chan struct{})
    //ch := func(stopChan chan struct{}) <-chan int {
    ch := func(ctx context.Context) <-chan int {
        ch := make(chan int)
        go func() {
            for i := 0; ; i++ {
                select {
                //case <-stopChan:
                //   return
                case <- ctx.Done():
                    return
                case ch <- i:
                }
            }
        }()
        return ch
    }(ctx)
    //}(stopChan)

    for value := range ch {
        fmt.Println(value)
        if value == 10 {
            cancel()  //通过调用cancel()来通知后台Goroutine退出
            // stopChan <- struct{}{}
            break
        }
    }
    
    // do some thing
}
6. go channel注意事项
  1. 如果是一个buffer channel，即使被 close， 也可以读到之前存入的值，读取完毕后开始读零值，已经关闭的channel写入则会触发 panic
  2.  nil channel 读取和存入都会阻塞，close 会 panic
  3. 已经close过的channel再次close会触发panic
  4. 关闭channel的原则：不要在消费端进行关闭、不要在有多个并行的生产端关闭
 二. 高性能Go编程：
2.1 数据结构
1. 尽可能指定容器容量，以便为容器预先分配内存。这将在后续添加元素时减少通过复制来调整容器大小
2. 遍历 []struct{} 使用下标而不是 range
  如果切片是[]int，切片的Item为int差别不大，但是如果切片Item是一个结构体类型struct{}时，且Item中包含一些比较大内存的成员类型，比如 [2048]byte，如果每次遍历[]struct{} ，都会进行一次值拷贝，所以会带来性能消耗。此外，因为 range 遍历事获取值拷贝的副本，所以对副本的修改，是不会影响到原切片。 如果切片的Item是指针类型，即[]*struct{} 则两者遍历方法都一样
3. string转[]byte的零拷贝操作：
我们一般在做string转[]byte时，会直接进行[]byte(string)，这样的话会发生一次拷贝操作，对于一些长尾的字符串，会产生性能问题。这是因为在Go的设计中string是immutable，而[]byte是mutable，如果不希望进行这次拷贝的消耗，这时候就需要用到unsafe
可参见：https://pkg.go.dev/reflect#SliceHeader
 func string2bytes(s string) []byte {
     stringHeader := (*reflect.StringHeader)(unsafe.Pointer(&s))

     var b []byte
     rs := (*reflect.SliceHeader)(unsafe.Pointer(&b))
     rs.Data = stringHeader.Data
     rs.Len = stringHeader.Len
     rs.Cap = stringHeader.Len
     return b
}
不过需要特别注意这样生成的[]byte不可修改，否则会出现未定义的行为

4. Go 中空结构体 struct{} 是不占用内存空间，因此fmt.Println(unsafe.Sizeof(struct{}{})) 为0，不像 C/C++ 中空结构体仍占用 1 字节大小，因此可用于以下几个场景来节省内存：
  1. 在实现集合时，可以将map的value的设置为struct{}来节省内存；
  2. 在使用不发送数据的channel时，只是用于通知其他Goroutine，也可以使用空结构体；
  3. 仅包含方法的结构体，即结构体没有任何成员类型字段
  
5. 尽量少使用反射，因为反射里边牵扯到类型判断和内存分配，会对性能产生影响
2.2 内存管理
1. struct结构体内的成员布局需要考虑内存对齐，一般建议字段宽度从小到大由上到下排列，减少内存占用，提高内存读写性能

2. 值传递会拷贝整个对象，而指针传递只会拷贝对象的地址，指针指向的对象是同一个。返回指针可以减少值的拷贝，但是会导致内存分配逃逸到堆中，即变量逃逸，增加垃圾回收(GC)的负担。在对象频繁创建和删除的场景下，传递指针会导致GC的开销增大，影响性能。
一般情况下，对于需要修改原对象值，或占用内存比较大的结构体，选择返回指针。对于只读的或者占用内存较小的结构体，直接返回值性能要优于指针

3. 对于需要重复分配、回收内存的地方，优先使用sync.Pool进行池化，用来保存和复用对象，减少内存分配，降低GC压力，并且sync.Pool是并发安全的
2.3 并发编程
1. 锁的使用
  1. 优先考虑无锁的数据结构使用，即lock-free 比如atomic包，但其底层也是memory barrier，如果并发太高，性能也未必高
  2. Goroutine局部，即线程的TLS，最后再进行每个Goroutine合并处理
  3. 进行数据切片，减少锁的竞争，或者使用读写锁，替换互斥锁
  4. 对于map数据结构的并发访问，在读多写少的情况下，建议使用官方的sync.Map，sync.Map是空间换时间的实现内部有两个map，有一个专门读的read map，另一个是提供读写的dirty map，优先读read map，未读到会穿透到dirty map，但是不适用于大量写的场景，dirty map会存在频繁刷新为read map，整体性能会降低。
2. Goroutine 池化 https://github.com/panjf2000/ants
