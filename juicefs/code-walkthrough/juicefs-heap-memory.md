# 内存管理

juicefs使用sync.Pool 当作堆内存，用于减缓对象的创建释放压力和gc带来的影响

```
// Alloc returns size bytes memory from Go heap.
func Alloc(size int) []byte {
    // 不同索引的pool的大小不一样（见init），计算出针对2的幂次数获取对应大小的pool
    zeros := powerOf2(size)
    b := *pools[zeros].Get().(*[]byte)
    if cap(b) < size {
        panic(fmt.Sprintf("%d < %d", cap(b), size))
    }
    atomic.AddInt64(&used, int64(cap(b)))
    return b[:size]
}

// Free returns memory to Go heap.
func Free(b []byte) {
    // buf could be zero length
    atomic.AddInt64(&used, -int64(cap(b)))
    pools[powerOf2(cap(b))].Put(&b)
}

// AllocMemory returns the allocated memory
func AllocMemory() int64 {
    return atomic.LoadInt64(&used)
}

var pools []*sync.Pool

func powerOf2(s int) int {
    var bits int
    var p int = 1
    for p < s {
        bits++
        p *= 2
    }
    return bits
}

func init() {
    pools = make([]*sync.Pool, 30) // 1 -- 1G
    // 申请30个不同大小的pool作为堆内存
    // 分别为1byte/2bytes/4bytes/8bytes...2^29=1G
    for i := 0; i < 30; i++ {
        func(bits int) {
            pools[i] = &sync.Pool{
                New: func() interface{} {
                    b := make([]byte, 1<<bits)
                    return &b
                },
            }
        }(i)
    }
    go func() {
        for {
            // 10min gc一次
            time.Sleep(time.Minute * 10)
            runtime.GC()
        }
    }()
}
```
