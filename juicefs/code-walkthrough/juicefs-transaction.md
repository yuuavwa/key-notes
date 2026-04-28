# 元数据事务流程

- redis watch键值之后，这些键值被修改会导致接下来的事务(MULTI ... EXEC，无论事务内容是否涉及这些键值)中断返回失败，此时需要重试事务
- juicefs中执行事务函数txn的redis实现

```go
func (m *redisMeta) txn(ctx Context, txf func(tx *redis.Tx) error, keys ...string) error {
    if m.conf.ReadOnly {
        return syscall.EROFS
    }
    for _, k := range keys {
        if !strings.HasPrefix(k, m.prefix) {
            panic(fmt.Sprintf("Invalid key %s not starts with prefix %s", k, m.prefix))
        }
    }
    var khash = fnv.New32()
    _, _ = khash.Write([]byte(keys[0]))
    h := int(khash.Sum32())
    start := time.Now()
    defer func() { txDist.Observe(time.Since(start).Seconds()) }()
    // 这里是1024个悲观锁，也就是通过加锁的方式控制并发最多在1024
    // 这里应该是为了减少redis下watch key之后事务失败重试的概率
    m.txLock(h)
    defer m.txUnlock(h)
    // TODO: enable retry for some of idempodent transactions
    var retryOnFailture = false
    var lastErr error
    for i := 0; i < 50; i++ {
        if ctx.Canceled() {
            return syscall.EINTR
        }
        err := m.rdb.Watch(ctx, txf, keys...)
        if eno, ok := err.(syscall.Errno); ok && eno == 0 {
            err = nil
        }
        // 这里错误说明是watch的key被其它修改了，redis的事务机制失败返回，这里直接重试
        if err != nil && m.shouldRetry(err, retryOnFailture) {
            txRestart.Add(1)
            logger.Debugf("Transaction failed, restart it (tried %d): %s", i+1, err)
            lastErr = err
            time.Sleep(time.Millisecond * time.Duration(rand.Int()%((i+1)*(i+1))))
            continue
        } else if err == nil && i > 1 {
            logger.Warnf("Transaction succeeded after %d tries (%s), keys: %v, last error: %s", i+1, time.Since(start), keys, lastErr)
        }
        return err
    }
    logger.Warnf("Already tried 50 times, returning: %s", lastErr)
    return lastErr
}
```
