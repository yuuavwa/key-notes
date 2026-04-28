# 读取流程

一般read操作会先open
open逻辑中，针对该inode new一个handler（一个inode可能有一系列的handler），该handler判断参数来绑定reader/writer，返回该handler对应的fh

```go
func (v *VFS) newFileHandle(inode Ino, length uint64, flags uint32) uint64 {
    h := v.newHandle(inode)
    h.Lock()
    defer h.Unlock()
    switch flags & O_ACCMODE {
    case syscall.O_RDONLY:
        h.reader = v.reader.Open(inode, length)
    case syscall.O_WRONLY: // FUSE writeback_cache mode need reader even for WRONLY
        fallthrough
    case syscall.O_RDWR:
        h.reader = v.reader.Open(inode, length)
        h.writer = v.writer.Open(inode, length)
    }
    return h.fh
}
```
```go
func (f *fileReader) Read(ctx meta.Context, offset uint64, buf []byte) (int, syscall.Errno) {
    f.Lock()
    defer f.Unlock()
    f.acquire()
    defer f.release()

    if f.err != 0 || f.closing {
        return 0, f.err
    }

    size := uint64(len(buf))
    if offset >= f.length || size == 0 {
        return 0, 0
    }
    // offset：读取位置  size：读取大小
    block := &frange{offset, size}
    // block.off + block.size 不会超出文件f的长度
    if block.end() > f.length {
        block.len = f.length - block.off
    }

    f.cleanupRequests(block)
    var lastBS uint64 = 32 << 10
    if block.off+lastBS > f.length {
        // 读取位置+32k后超过文件长度的情况下，才需要readAhead
        // 这样做的目的在于读取block快结束时才启动readAhead预读下一个block，减少读放大
        lastblock := frange{f.length - lastBS, lastBS}
        if f.length < lastBS {
            lastblock = frange{0, f.length}
        }
        f.readAhead(&lastblock)
    }
    ranges := f.splitRange(block)
    reqs := f.prepareRequests(ranges)
    defer func() {
        for _, req := range reqs {
            s := req.s
            s.refs--
            if s.refs == 0 && s.state == INVALID {
                s.delete()
            }
        }
    }()
    f.checkReadahead(block)
    return f.waitForIO(ctx, reqs, buf)
}
```

page有的是new出来的新page用于暂存数据，有的是slice中的page
slice中的page跟block有对应关系
```go
func (s *rSlice) ReadAt(ctx context.Context, page *Page, off int) (n int, err error) {
    // 从位置off开始读取page的所有内容
    p := page.Data
    if len(p) == 0 {
        return 0, nil
    }
    if off >= s.length {
        return 0, io.EOF
    }

    indx := s.index(off)
    boff := off % s.store.conf.BlockSize
    // 偏移位置off所在block的block size
    blockSize := s.blockSize(indx)
    // 当需要读取的block偏移位置 + 待读取的数据大小 > block size的情况下
    // 也就是需要读取的数据内容跨了一个block的情况下，以实际block数据为界分配page读取内容
    // 因为后端实际存储的是一个个block，通过递归调用ReadAt函数读取，逻辑大概为：
    // 先申请page，读取boff~block end
    // 申请page=blocksize，读取中间block内容
    // 最后申请page，读取got~总page end
    if boff+len(p) > blockSize {
        // read beyond currend page
        var got int
        // got为page.data指标，循环读取p
        for got < len(p) {
            // aligned to current page
            // l为需要读取的剩余长度
            //p为应读取的page总大小 - got为已经读取的数据偏移量 = page剩余未读取量
            //s.blockSize(s.index(off)) 为对应block数据总size - boff 为block内偏移量 = block剩余应读取量
            l := utils.Min(len(p)-got, s.blockSize(s.index(off))-off%s.store.conf.BlockSize)
            // 申请一个新的page pp，数据索引为 got ~ l
            pp := page.Slice(got, l)
            n, err = s.ReadAt(ctx, pp, off)
            // 该page内容读取完成后，release释放回堆内存
            pp.Release()
            if err != nil {
                return got + n, err
            }
            if n == 0 {
                return got, io.EOF
            }
            got += n
            off += n
        }
        return got, nil
    }

    // 以下为每次递归调用的实际执行内容
    key := s.key(indx)
    if s.store.conf.CacheSize > 0 {
        start := time.Now()
        r, err := s.store.bcache.load(key)
        if err == nil {
            // 这里是os.File的ReadAt，直接从文件特定位置读取到p中
            n, err = r.ReadAt(p, int64(boff))
            _ = r.Close()
            if err == nil {
                s.store.cacheHits.Add(1)
                s.store.cacheHitBytes.Add(float64(n))
                s.store.cacheReadHist.Observe(time.Since(start).Seconds())
                return n, nil
            }
            if f, ok := r.(*os.File); ok {
                logger.Warnf("remove partial cached block %s: %d %s", f.Name(), n, err)
                _ = os.Remove(f.Name())
            }
        }
    }

    s.store.cacheMiss.Add(1)
    s.store.cacheMissBytes.Add(float64(len(p)))

    if s.store.seekable && boff > 0 && len(p) <= blockSize/4 {
        // 远端存储支持seek，读取数据大小<1/4 blocksize的情况下，直接调用Get
        if s.store.downLimit != nil {
            // 下载限速
            s.store.downLimit.Wait(int64(len(p)))
        }
        // partial read
        st := time.Now()
        // 这里get下来并没有缓存起来
        in, err := s.store.storage.Get(key, int64(boff), int64(len(p)))
        if err == nil {
            n, err = io.ReadFull(in, p)
            _ = in.Close()
        }
        used := time.Since(st)
        logger.Debugf("GET %s RANGE(%d,%d) (%s, %.3fs)", key, boff, len(p), err, used.Seconds())
        if used > SlowRequest {
            logger.Infof("slow request: GET %s (%v, %.3fs)", key, err, used.Seconds())
        }
        s.store.objectDataBytes.WithLabelValues("GET").Add(float64(n))
        s.store.objectReqsHistogram.WithLabelValues("GET").Observe(used.Seconds())
        // 通过prefetch 异步缓存对应的key
        s.store.fetcher.fetch(key)
        if err == nil {
            return n, nil
        } else {
            s.store.objectReqErrors.Add(1)
        }
    }

    // 通过类似singleflight的方式从后端load整个block，避免高并发导致惊群/击穿
    // Execute的实现逻辑类似一写多读锁，并发下对于相同的request只返回一个结果，其余请求需要等待
    block, err := s.store.group.Execute(key, func() (*Page, error) {
        tmp := page
        if boff > 0 || len(p) < blockSize {
            // boff > 0：第一个分配的page
            // len(p) < blockSize：最后一个分配的page
            // 此时重新申请page，因为要把整个block load下来，大小使用对应位置所在block的实际size
            tmp = NewOffPage(blockSize)
        } else {
            tmp.Acquire()
        }
        tmp.Acquire()
        err := utils.WithTimeout(func() error {
            defer tmp.Release()
            return s.store.load(key, tmp, s.store.shouldCache(blockSize), false)
        }, s.store.conf.GetTimeout)
        return tmp, err
    })
    defer block.Release()
    if err != nil {
        return 0, err
    }
    if block != page {
        // 没有复用原来的page，说明是头/尾的page，此时需要根据位置重新copy
        // 复用原来的page，说明是中间page，就是整块的block
        copy(p, block.Data[boff:])
    }
    return len(p), nil
}
```

预读 readAhead

```go
func (f *fileReader) readAhead(block *frange) {
    f.visit(func(r *sliceReader) {
        if r.state.valid() && r.block.off <= block.off && r.block.end() > block.off {
            if r.state == READY && block.len > f.r.blockSize && r.block.off == block.off && r.block.off%f.r.blockSize == 0 {
                // next block is ready, reduce readahead by a block
                block.len -= f.r.blockSize / 2
            }
            if r.block.end() <= block.end() {
                block.len = block.end() - r.block.end()
            } else {
                // 否则说明readslice已经包含需要读取的block的所有区间，不需要再读取了
                block.len = 0
            }
            // 调整block读取起始位置off为找到的block的最后位置，即block代表下一块
            block.off = r.block.end()
        }
    })
    if block.len > 0 && block.off < f.length && uint64(atomic.LoadInt64(&readBufferUsed)) < f.r.GetReadAheadTotal() {
        if block.len < f.r.blockSize {
            // 对齐一个block，原来基础上拉伸至4M block end
            block.len += f.r.blockSize - block.end()%f.r.blockSize // align to end of a block
        }
        // block.len <= 0 不会newSlice
        f.newSlice(block)
        if block.len > 0 {
            f.readAhead(block)
        }
    }
}
```
