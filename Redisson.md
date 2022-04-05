### 可重入锁（Reentrant Lock）

基于Redis的Redisson分布式可重入锁RLock java对象实现了`java.util.concurrent.locks.Lock`接口。

```java
RLock lock = redisson.getLock("qhyu");
// 最常见的使用方法
lock.lock();
lock.unlock();
```

大家都知道，如果负责存储这个分布式锁的Redisson节点宕机以后，而且这个锁正好处于锁住的状态时，这个锁会出现锁死的状态，为了避免这种情况发送，Redisson内部提供了一个监控锁的看门狗，它的作用是在Redisson实例关闭前，不断的延长锁的有效期，默认情况下，看门狗的检查锁的超时时间是30s，也可以通过修改Config.lockWatchdogTimeOut来另行指定。

另外Redisson还通过加锁的方法提供了`leaseTime`的参数来指定加锁的时间。超过这个时间后锁便自动解开了。

```java
// 加锁以后10秒钟自动解锁
// 无需调用unlock方法手动解锁
lock.lock(10, TimeUnit.SECONDS);

// 尝试加锁，最多等待100秒，上锁以后10秒自动解锁
boolean res = lock.tryLock(100, 10, TimeUnit.SECONDS);
if (res) {
   try {
     ...
   } finally {
       lock.unlock();
   }
}
```

Redisson同时还为分布式锁提供了异步执行的相关方法：

```java
RLock lock = redisson.getLock("anyLock");
lock.lockAsync();
lock.lockAsync(10, TimeUnit.SECONDS);
Future<Boolean> res = lock.tryLockAsync(100, 10, TimeUnit.SECONDS);
```

查看RedissonLock中的lock方法

```java
@Override
public void lock() {
    try {
        lock(-1, null, false);
    } catch (InterruptedException e) {
        throw new IllegalStateException();
    }
}
```

leaseTime参数用来指定加锁的时间，TimeUnit时间单位，interruptibly是否中断

```java
 private void lock(long leaseTime, TimeUnit unit, boolean interruptibly) throws InterruptedException {
     	// 获取当前线程的线程Id
        long threadId = Thread.currentThread().getId();
        // 尝试获取锁
        Long ttl = tryAcquire(leaseTime, unit, threadId);
        // .......先省略后续代码查看tryAcquire(leaseTime, unit, threadId);
    }
```

跟进tryAcquire代码逻辑最后会进入tryAcquireAsync方法。**注意：当leaseTime被我们手动设置了值且不是-1的时候，将不会执行看门狗的一些逻辑。**

```java
private <T> RFuture<Long> tryAcquireAsync(long leaseTime, TimeUnit unit, long threadId) {
        // 当前传入的就是-1
    	if (leaseTime != -1) {
            // 如果当前锁空闲，返回null，如果是重入也返回null，否则返回锁的过期时间
            return tryLockInnerAsync(leaseTime, unit, threadId, RedisCommands.EVAL_LONG);
        }
        // 如果leaseTime=-1，那么获取看门狗的时间，默认是30*1000毫秒
        RFuture<Long> ttlRemainingFuture = tryLockInnerAsync(commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout(), TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);
        // 脚本返回的值ttlRemaining，获取到锁的返回的是null，否则返回的锁过期时间
        ttlRemainingFuture.onComplete((ttlRemaining, e) -> {
            // 异常了 直接return
            if (e != null) {
                return;
            }

            // lock acquired
            if (ttlRemaining == null) {
                // 到期续订的方法
                scheduleExpirationRenewal(threadId);
            }
        });
        return ttlRemainingFuture;
    }
```

tryLockInnerAsync方法中主要就是查看这个luna脚本。

KEYS[1]:表示redis-key，即传入的key(qhyu)

ARGV[1]：表示获取时间

ARGV[2]：表示加锁的客户端id

```lua
--首先查看key是不是存在，存在返回1，否则返回0
if (redis.call('exists', KEYS[1]) == 0) then 
-- 不存在则加锁，将hash表key中的客户端id字段的值+1
redis.call('hset', KEYS[1], ARGV[2], 1);
-- 设置key过期时间，以毫秒计
redis.call('pexpire', KEYS[1], ARGV[1]); 
return nil; 
end; 
-- 查看hash表key中是否存在客户端id字段，存在返回1，不存在返回0
if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then 
-- 为hash表key中指定的客户端id字段的整数值加上增量1
redis.call('hincrby', KEYS[1], ARGV[2], 1); 
-- 设置key过期时间，以毫秒计    
redis.call('pexpire', KEYS[1], ARGV[1]); 
return nil; 
end; 
-- 返回key的剩余过期时间
return redis.call('pttl', KEYS[1]);
```

第一个if判断是判断存不存在key值，如果存在再进入第二个if判断，如果客户端id相同则是可重入的操作方式，如果客户端id不同，直接返回key的剩余过期时间。

scheduleExpirationRenewal(threadId);方法源码解析

```java
private static final ConcurrentMap<String, ExpirationEntry> EXPIRATION_RENEWAL_MAP = new ConcurrentHashMap<>();

private void scheduleExpirationRenewal(long threadId) {
        ExpirationEntry entry = new ExpirationEntry();
        // putIfAbsent 如果不存在，put一个新得并且返回null
        ExpirationEntry oldEntry = EXPIRATION_RENEWAL_MAP.putIfAbsent(getEntryName(), entry);
        // 如果不为空的时候，说明此时是重入锁
        if (oldEntry != null) {
            // 直接锁数量+1,放入oldEntry中
            oldEntry.addThreadId(threadId);
        } else {
            // 如果是新获得的锁，那么就是entry中将锁数量设置为1，方如entry中
            entry.addThreadId(threadId);
            // 更新到期方法
            renewExpiration();
        }
    }
```

renewExpiration()方法有三个关键点：

1、Timeout类

2、renewExpirationAsync执行luna脚本的核心代码

3、internalLockLeaseTime / 3，该任务10s执行一次。

```java
// 更新到期方法
private void renewExpiration() {
        // 先从map中获取到ee对象
        ExpirationEntry ee = EXPIRATION_RENEWAL_MAP.get(getEntryName());
		// 如果为空直接返回
        if (ee == null) {
            return;
        }
        // 设置ee中timeout的值
        Timeout task = commandExecutor.getConnectionManager().newTimeout(new TimerTask() {
            @Override
            public void run(Timeout timeout) throws Exception {
                ExpirationEntry ent = EXPIRATION_RENEWAL_MAP.get(getEntryName());
                if (ent == null) {
                    return;
                }
                // 获取到第一个线程Id，而且这里只会有一个，scheduleExpirationRenewal方法中put进去的
                Long threadId = ent.getFirstThreadId();
                if (threadId == null) {
                    return;
                }
                // 异步更新过期方法，里面也是个luna脚本
                RFuture<Boolean> future = renewExpirationAsync(threadId);
                // 如果更新过期时间成功返回1，否则返回0
                future.onComplete((res, e) -> {
                    if (e != null) {
                        log.error("Can't update lock " + getName() + " expiration", e);
                        return;
                    }
                    // luna延期30s之后，返回成功，又调用自己
                    if (res) {
                        // reschedule itself
                        renewExpiration();
                    }
                });
            }
        }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);
        ee.setTimeout(task);
    }
```

解锁的核心方法是unlockInnerAsync，核心的逻辑就是中间的luna脚本。

```java
protected RFuture<Boolean> unlockInnerAsync(long threadId) {
        return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
                // 如果释放锁的线程ID不存在当前的hash表中，返回null                             
                "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
                    "return nil;" +
                "end; " +
                // 存在的化，释放一次锁，锁数量-1                              
                "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
                // 剩余的锁数量大于0，刷新过期时间                              
                "if (counter > 0) then " +
                    "redis.call('pexpire', KEYS[1], ARGV[2]); " +
                    "return 0; " +
                // 否则释放锁，删除hash表key                              
                "else " +
                    "redis.call('del', KEYS[1]); " +
                    "redis.call('publish', KEYS[2], ARGV[1]); " +
                    "return 1; "+
                "end; " +
                "return nil;",
                Arrays.<Object>asList(getName(), getChannelName()), LockPubSub.UNLOCK_MESSAGE, internalLockLeaseTime, getLockName(threadId));
    }
```

#### 总结

1、加锁的时候自己设定过期时间，就没有看门狗机制了。所以在使用的时候尽量可以不设定过期时间，因为其实没办法预估代码在锁中执行的时间，有看门狗机制就可以不考虑这个问题。

2、看门狗默认的续命时间是10s左右，internalLockLeaseTime / 3。并且是通过luna脚本进行续命的。

3、看门狗延时时间 可以由 lockWatchdogTimeout指定默认延时时间，但是不要设置太小。

4、如果程序释放锁操作时因为异常没有被执行，那么锁无法被释放，所以释放锁操作一定要放到 finally {} 中。

