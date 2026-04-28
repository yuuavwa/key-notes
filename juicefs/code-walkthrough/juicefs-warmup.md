# 缓存预热

- 1、通过将命令行写入 .control文件，由后端协程 handleInternalMsg 来处理
- 2、写入control文件的内容格式：

```go
func sendCommand(cf *os.File, batch []string, threads uint, background bool, dspin *utils.DoubleSpinner) {
    paths := strings.Join(batch, "\n")
    var back uint8
    if background {
        back = 1
    }
    // NewBuffer的长度由如下的Put总计算得出
    wb := utils.NewBuffer(8 + 4 + 3 + uint32(len(paths)))
    wb.Put32(meta.FillCache)  // Put32位4个字节
    wb.Put32(4 + 3 + uint32(len(paths)))  // 4个字节, 这里边的值+8bytes即NewBuffer的size
    wb.Put32(uint32(len(paths)))  // 4
    wb.Put([]byte(paths))  // len(paths) 个字节
    wb.Put16(uint16(threads))  // 2 个字节
    wb.Put8(back)  // 1个字节
    if _, err := cf.Write(wb.Bytes()); err != nil {
        logger.Fatalf("Write message: %s", err)
    }
    if background {
        logger.Infof("Warm-up cache for %d paths in background", len(batch))
        return
    }
    if errno := readProgress(cf, func(count, bytes uint64) {
        dspin.SetCurrent(int64(count), int64(bytes))
    }); errno != 0 {
        logger.Fatalf("Warm up failed: %s", errno)
    }
}
```
