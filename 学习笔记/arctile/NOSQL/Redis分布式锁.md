# Redis分布式锁

## 使用场景

当服务器集群需对某一串操作进行原子性操作时,需使用分布式锁来实现互斥性



## 实现方式

#### 常规实现方式

由以下三个命令组成的方案

+ `setnx key val`
+ `  expire key`
+ `del key`

##### 问题

`setnx`与`expire`指令不是原子性操作,当客户端因各种原因而在`setnx`与`expire` 之间宕机或断开链接时,此时锁的维护key将一直存在,其它想竞争锁的客户端将一直无法获取锁.



#### 优化版

Redis在2.8版本推出了`setnx`与`expire`集成命令`set key val ex seconds nx`

如`set lock true ex 5 nx`表示当lock键不存在时,设置lock键值为true并将其过期时间设置为5s.

所以此时的实现方案是由以下两个命令组成的

+ `set key val ex seconds nx`
+ del key

此时解决了锁可能永远获取不了的问题

##### 问题

依然存在超时问题

如果分布式锁锁住的业务执行时间过长,甚至大于expire的超时时间,那么此时分布式锁的key将被删除.而此时另一个客户端发现key不在了,并且成功抢占到了锁,执行了`set`命令.那么此时就有两个客户端在执行同一个任务.原子性保证被破坏.而且当第一个客户端执行完毕任务时,会删除掉锁的key,导致分布式锁的原子性再一次被破坏.



#### 优化版++

针对优化版的分布式锁原子性连续被破坏问题,我们可以将锁的值设置为一个时间戳(随机数),然后删除锁时需值匹配才能删除.这样就能避免上一个超时任务影响下一个正在执行的任务,从而减少任务执行超时带来的影响.



##### 问题

依旧没解决超时问题,而且删除操作从原子操作变为先比较再删除的非原子操作



**最优解**

**加锁**

```
set key val ex seconds nx
```

**解锁**

为解决删除操作的非原子性问题,我们可引入`lua`脚本来实现原子性

```lua
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

##### 问题

依旧没解决超时问题



## 注意事项

+ 尽量控制业务在锁的执行时间来减少expire时间超时引起的问题
+ redis的指令同步是异步的,而expire指令的同步策略是主是在AOF文件增加一条`del`指令,从库通过执行这条指令来实现删除过期key的.而如果主从数据不一致,主库木有的数据在从库还存在.而如果此时主出现故障,那么后面的任务将永远获取不到锁.



## Java的常见实现方式

#### 正确版

```java
public class RedisLockTool {
    public static final DEL_SUCCESS = "1";

    //获取锁
    public static boolean tryGetDistributedLock(Jedis jedis, String lockKey, String val, int expireTime) {
        String result = jedis.set(lockKey, val, "NX", "PX", expireTime);
        if ("OK".equals(result)) {
            return true;
        }
        return false;
    }
    
    //释放锁
    public static boolean releaseDistributedLock(Jedis jedis, String lockKey, String val) {
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        Object result = jedis.eval(script, Collections.singletonList(lockKey), Collections.singletonList(val));
        if (DEL_SUCCESS.equals(result)) {
           return true;
        }
        return false;
    }
}

```

#### 典型的错误版本

```java
/**
 * 1.获取锁的主要问题在于时间戳由客户端生成,无法保证统一性,需定时做时间同步
 * 2.getset操作可能会相互覆盖,虽然最终是第一个成功修改的获取到锁,但过期时间会被覆盖(影响较小)
 * 3.锁不具备标志性,expire超时情况出现时,会出现上一个超时持有锁者删除下一个正在执行任务持有者的情况
*/
public static boolean wrongGetLock(Jedis jedis, String lockKey, int expireTime) {
    long expires = System.currentTimeMillis() + expireTime;
    String expiresStr = String.valueOf(expires);
    // 如果当前锁不存在，返回加锁成功
    if (jedis.setnx(lockKey, expiresStr) == 1) {
        return true;
    }
    // 如果锁存在，获取锁的过期时间
    String currentValueStr = jedis.get(lockKey);
    if (currentValueStr != null && Long.parseLong(currentValueStr) < System.currentTimeMillis()) {
        // 锁已过期，获取上一个锁的过期时间，并设置现在锁的过期时间
        String oldValueStr = jedis.getSet(lockKey, expiresStr);
        if (oldValueStr != null && oldValueStr.equals(currentValueStr)) {
            // 考虑多线程并发的情况，只有一个线程的设置值和当前值相同，它才有权利加锁
            jedis.expire(expireTime);
            return true;
        }
    }
    // 其他情况，一律返回加锁失败
    return false;
}


/**
* 锁不具备拥有者标志,任何客户端都可以解锁
*/
public static void wrongReleaseLock(Jedis jedis, String lockKey) {
    jedis.del(lockKey);
}

/**
 * 错误的释放锁实例
*/
public static void wrongReleaseLock2(Jedis jedis, String lockKey, String val) {
    // 判断加锁与解锁是不是同一个客户端
    if (val.equals(jedis.get(lockKey))) {
        // 若在此时，这把锁突然不是这个客户端的，则会误解锁
        jedis.del(lockKey);
    }
}
```



## 参考文章

<https://juejin.im/book/5afc2e5f6fb9a07a9b362527/section/5afc35fb6fb9a07abf72b477>

https://dwz.cn/FPaLs4cb

