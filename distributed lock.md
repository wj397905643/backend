# distributed lock

[TOC]

## 问题分析

考虑用户模块下的用户服务实现（`com.seckill.dis.user.service.UserServiceImpl`）：

```java
@Override
public CodeMsg register(RegisterVo userModel) {

    // 检查用户是否注册
    SeckillUser user = this.getSeckillUserByPhone(userModel.getPhone());

    // 用户已经注册
    if (user != null) {
        return CodeMsg.USER_EXIST;
    }

    // 生成skuser对象
	...

    // 写入数据库
    long id = userMapper.insertUser(newUser);

    // 用户注册成功
    if (id > 0)
        return CodeMsg.SUCCESS;

    // 用户注册失败
    return CodeMsg.REGISTER_FAIL;
}
```

显然，这段代码存在线程安全问题，在恶意用户使用注册接口重复写入相同数据的时候，会导致用户数据表中存在重复的数据，这是我们所不希望的。

**实际上，上述问题可以归结为：一次请求和多次请求对资源本身具有同样的结果，也就是说，相同条件下的任意多次执行对资源本身所产生的影响均与一次执行的影响相同，即保证接口的幂等性**。这就是我们需要解决的问题。

## 解决思路

上述问题的解决可以从几个方面入手：

1. 数据库：使用乐观锁、悲观锁或唯一索引；
1. 服务端：加锁。

不管从哪个方面入手，本质的手段都是加锁。

- 使用悲观锁比较简单，使用在`insert`加上`on duplicate key update `即可，这样可以以去重复的方式加锁访问。

- 使用索引也比较简单，对`phone`字段加唯一索引即可。

- 使用乐观锁需要在表上加上一个表示版本的字段。

但是，利用数据库实现锁，会造成大量的请求落在数据库上，并阻塞等待执行，这个过程是需要消耗数据库资源的，使得真正的业务请求得不到处理，在秒杀大并发情形下，可能会导致数据库宕机，数据我们不使用数据库来实现锁，而是在服务端实现锁。

如果只有一个用户服务模块实例，则采用JUC下的`ReentrantLock`重入锁实现即可，或者直接对方法加上`synchronized`，这在单个实例下式没有问题的，也就是单进程的情况。

如果用户服务模块存在多个实例，也就是以集群的方式部署，那么就涉及进程之间的锁问题，`synchronized`和`ReentrantLock`这种单进程的锁只对落到本服务模块的请求有效，而对多进程无效，依旧会有线程安全问题。

这个时候，分布式锁就派上用场了。

## Redis 分布式锁

分布式锁一般有三种实现方式：

1. 数据库乐观锁；
1. 基于Redis的分布式锁；
1. 基于ZooKeeper的分布式锁。

本文将介绍第二种方式，基于Redis实现分布式锁。

首先，为了确保分布式锁可用，我们至少要确保锁的实现同时满足以下四个条件：

1. 互斥性。在任意时刻，只有一个客户端能持有锁。
1. 不会发生死锁。即使有一个客户端在持有锁的期间崩溃而没有主动解锁，也能保证后续其他客户端能加锁。
1. 具有容错性。只要大部分的Redis节点正常运行，客户端就可以加锁和解锁。
1. 解铃还须系铃人。加锁和解锁必须是同一个客户端，客户端自己不能把别人加的锁给解了。

### 加锁

```java
public boolean lock(String lockKey, String uniqueValue, int expireTime) {
    Jedis jedis = null;
    try {
        jedis = jedisPool.getResource();
        // 获取锁
        String result = jedis.set(lockKey, uniqueValue, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, expireTime);
        if (LOCK_SUCCESS.equals(result)) {
            return true;
        }
        return false;
    } finally {
        if (jedis != null)
            jedis.close();
    }
}
```

可以看到，我们加锁就一行代码：jedis.set(String key, String value, String nxxx, String expx, int time)，这个set()方法一共有五个形参：

- 第一个为key，我们使用key来当锁，因为key是唯一的。
- 第二个为value，我们传的是uniqueValue，有key作为锁不就够了吗，为什么还要用到value？原因就是我们在上面讲到可靠性时，分布式锁要满足第四个条件解铃还须系铃人，通过给value赋值为uniqueValue，我们就知道这把锁是哪个请求加的了，在解锁的时候就可以有依据。
- 第三个为nxxx，这个参数我们填的是NX，意思是SET IF NOT EXIST，即当key不存在时，我们进行set操作；若key已经存在，则不做任何操作；
- 第四个为expx，这个参数我们传的是PX，意思是我们要给这个key加一个过期的设置，具体时间由第五个参数决定。
- 第五个为time，与第四个参数相呼应，代表key的过期时间。

总的来说，执行上面的set()方法就只会导致两种结果：

1. 当前没有锁（key不存在），那么就进行加锁操作，并对锁设置个有效期，同时value表示加锁的客户端。
1. 已有锁存在，不做任何操作。

我们的加锁代码满足可靠性里描述的三个条件。

首先，set()加入了NX参数，可以保证如果已有key存在，则函数不会调用成功，也就是只有一个客户端能持有锁，满足互斥性。

其次，由于我们对锁设置了过期时间，即使锁的持有者后续发生崩溃而没有解锁，锁也会因为到了过期时间而自动解锁（即key被删除），不会发生死锁。

最后，因为我们将value赋值为uniqueValue，代表加锁的客户端请求标识，那么在客户端在解锁的时候就可以进行校验是否是同一个客户端。由于我们只考虑Redis单机部署的场景，所以容错性我们暂不考虑。

### 解锁代码

```java
public boolean unlock(String lockKey, String uniqueValue) {
    Jedis jedis = null;
    try {
        jedis = jedisPool.getResource();
        // 使用Lua脚本保证操作的原子性
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                "return redis.call('del', KEYS[1]) " +
                "else return 0 " +
                "end";
        Object result = jedis.eval(script, Collections.singletonList(lockKey), Collections.singletonList(uniqueValue));

        if (RELEASE_SUCCESS.equals(result)) {
            return true;
        }
        return false;
    } finally {
        if (jedis != null)
            jedis.close();
    }
}
```

我们写了一个简单的Lua脚本代码，首先获取锁对应的value值，检查是否与uniqueValue相等，如果相等则删除锁（解锁）。那么为什么要使用Lua语言来实现呢？因为要确保上述操作是原子性的。

如果不用lua，而直接用get和del，则有以下几步：

1. `get lockKey`
1. 判断
1. `del lockKey`

可见，这三步不是原子的，也就会在`del`的时候导致误删。考虑下面的情形：

如果客户端A执行`get lockKey`，执行完后，判断是客户端A，那么接下来就是准备删除操作了`del lockKey`，而如果在`del lockKey`之前`lockKey`失效，客户端B此时获取锁，问题出现了。

客户端继续执行`del lockKey，lockKey`被删除，而此时的`lockKey`却是客户端B的锁，锁以就造成了锁的误删。这个时候如果再出现客户端C获取锁，那就会造成客户端C和客户端B同时访问一个资源，违反了分布式锁的互斥性。因此，分步骤实现分布式锁是不可行的，但是如果这三个步骤是原子的，那就没问题了。而Lua正好解决了这个问题。

## 加锁后的实现

```java
public CodeMsg register(RegisterVo userModel) {

        // 加锁
        String uniqueValue = UUIDUtil.uuid() + "-" + Thread.currentThread().getId();
        String lockKey = "redis-lock" + userModel.getPhone();
        boolean lock = dLock.lock("redis-lock", uniqueValue, 60 * 1000);
        if (!lock)
            return CodeMsg.WAIT_REGISTER_DONE;
        logger.debug("注册接口加锁成功");

        // 检查用户是否注册
        SeckillUser user = this.getSeckillUserByPhone(userModel.getPhone());

        // 用户已经注册
        if (user != null) {
            return CodeMsg.USER_EXIST;
        }

        // 生成skuser对象
		...
            
        // 写入数据库
        long id = userMapper.insertUser(newUser);

        boolean unlock = dLock.unlock(lockKey, uniqueValue);
        if (!unlock)
            return CodeMsg.REGISTER_FAIL;
        logger.debug("注册接口解锁成功");

        // 用户注册成功
        if (id > 0)
            return CodeMsg.SUCCESS;

        // 用户注册失败
        return CodeMsg.REGISTER_FAIL;
    }
```

在访问数据库之前加分布式锁，在完成业务之后释放分布式锁。值得注意的是，分布式锁需要标识用户以防止其他用户无法获取到锁。


## 不靠谱的情况
上面我们说的是redis，是单点的情况。如果是在redis sentinel集群中情况就有所不同了。关于redis sentinel 集群可以看这里。在redis sentinel集群中，我们具有多台redis，他们之间有着主从的关系，例如一主二从。我们的set命令对应的数据写到主库，然后同步到从库。当我们申请一个锁的时候，对应就是一条命令 setnx mykey myvalue ，在redis sentinel集群中，这条命令先是落到了主库。假设这时主库down了，而这条数据还没来得及同步到从库，sentinel将从库中的一台选举为主库了。这时，我们的新主库中并没有mykey这条数据，若此时另外一个client执行 setnx mykey hisvalue , 也会成功，即也能得到锁。这就意味着，此时有两个client获得了锁。这不是我们希望看到的，虽然这个情况发生的记录很小，只会在主从failover的时候才会发生，大多数情况下、大多数系统都可以容忍，但是不是所有的系统都能容忍这种瑕疵。

## redlock
为了解决故障转移情况下的缺陷，Antirez 发明了 Redlock 算法，使用redlock算法，需要多个redis实例，加锁的时候，它会想多半节点发送 setex mykey myvalue 命令，只要过半节点成功了，那么就算加锁成功了。释放锁的时候需要想所有节点发送del命令。这是一种基于【大多数都同意】的一种机制。感兴趣的可以查询相关资料。在实际工作中使用的时候，我们可以选择已有的开源实现，python有redlock-py，java 中有Redisson redlock。

redlock确实解决了上面所说的“不靠谱的情况”。但是，它解决问题的同时，也带来了代价。你需要多个redis实例，你需要引入新的库 代码也得调整，性能上也会有损。所以，果然是不存在“完美的解决方案”，我们更需要的是能够根据实际的情况和条件把问题解决了就好。
