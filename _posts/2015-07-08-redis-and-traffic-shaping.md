---
    layout: default
    comments: true
    title:  【Redis】Redis 与网络流量整形
---

##<strong>{{ page.title }}</strong>&nbsp;&nbsp;<small>{{ page.date | date_to_string }}</small><br><br>


###摘要


- 我们希望服务器能在请求流量的控制上有一定的自动控制能力；本文通过简介令牌桶算法和讨论算法的 redis 实现给出流量整形(traffic shaping)的示例，来介绍网络流量整形。

###令牌桶算法


- [令牌桶算法](http://baike.baidu.com/link?url=NP_yYC5SnzB2Z9vkfdx-8WRLlAR5I3YO47qzWOpVbamsQdmwd3vwacofBGxK3lpcUvmaV9AMufBS7rBrcHt77a)(token bucket) 并不是网络流量整形中的奇技淫巧，而是非常常用的算法，从百度百科上已经可以对它有一个概括的了解。对此算法的深入读者可自行查阅研究，这里我通俗化的来解释一下这个算法。

- 在令牌桶算法中，每一个访客都拥有一个独立的“令牌桶”，在这个“令牌桶”里放了一些“令牌”，访客每次来访都会消耗“令牌桶”中的“令牌”，如果“令牌桶”空了，将会对访客做特殊处理（如拒绝其继续访问以达到限流的目的）。

- 问题一：访客来访是一个持续的过程，如果最初的“令牌”数目固定，“令牌桶”中的令牌会慢慢被消耗殆尽，这样正常的访客也将无法访问----所以我们需要以一个恒定的速率来向“令牌桶”中添加一定数量的“令牌”， 这样就可以让访客持续的访问。

- 问题二：我们以一个恒定速率向“令牌桶”中添加“令牌”， 那么如果访客一直没来访他的“令牌桶”岂不会累积大量“令牌”么？----所以，我们设定“令牌桶”中“令牌”的最大数量，“令牌桶”满了就不需要再去添加了。这解决了“令牌”累积的问题，也使它更像一个“桶”。

- 如此，“令牌桶算法”中的重要的参数有：1. 给“令牌桶”添加“令牌”的速率（如果访客以这个速率消耗令牌，将一直不会被限流）； 2. “令牌桶”的容量（如果消耗令牌的速率大于添加令牌的速率，将消耗桶中的存货，如果消耗速率过大，令牌会被消耗殆尽，访客将被限流）。注意：一般情况下“令牌桶”最初的状态是满的。

###Redis


- 作为优秀的内存数据库，redis 可以帮助我们在应用层次快速响应。本文不过多赘述 redis 的优劣，你可以用 redis 做很多事情，在网络流量整形方面，它是很好的实现方案， 下面我们来解析这样一个方案。

###Show me the  code


- 说明：这是一段 Python 代码，这段代码来自 GitHub 用户 [justinfay](https://gist.github.com/justinfay/3403846)。为了使逻辑更清楚，我修改了代码的部分内容和注释，以下是修改后的代码，我们用这段代码来看令牌桶算法的 redis 实现。


```

import redis
from redis import WatchError
import time

RATE = 0.1  
# 令牌桶的最大容量       
DEFAULT = 100 
# redis key 的过期时间
TIMEOUT = 60 * 60 

r = redis.Redis()

def token_bucket(tokens, key):

    pipe = r.pipeline()
    while 1:
        try:
            pipe.watch('%s:available' % key) 
            pipe.watch('%s:ts' % key)    

            current_ts = time.time()

            # 获取令牌桶中剩余令牌
            old_tokens = pipe.get('%s:available' % key)
            if old_tokens is None:
                current_tokens = DEFAULT
            else:
                old_ts = pipe.get('%s:ts' % key)
                # 通过时间戳计算这段时间内应该添加多少令牌，如果桶满，令牌数取桶满数。
                current_tokens = float(old_tokens) + min(
                    (current_ts - float(old_ts)) * RATE,
                    DEFAULT - float(old_tokens)  
                )
            # 判断剩余令牌是否足够
            if 0 &lt;= tokens &lt;= current_tokens: 
                current_tokens -= tokens 
                consumes = True 
            else: 
                consumes = False 

            # 以下动作为更新 redis 中key的值，并跳出循环返回结果。
            pipe.multi() 
            pipe.set('%s:available' % key, current_tokens) 
            pipe.expire('%s:available' % key, TIMEOUT) 
            pipe.set('%s:ts' % key, current_ts) 
            pipe.expire('%s:ts' % key, TIMEOUT) 
            pipe.execute() 
            break 
        except WatchError: 
            continue 
        finally: 
            pipe.reset() 
    return consumes 

if __name__ == "__main__": 
    tokens = 5 
    key = '192.168.1.1' 
    if token_bucket(tokens, key): 
        print 'haz tokens' 
    else: 
        print 'cant haz tokens'</pre>

```

###需要说的几点


####1. 这段代码在网络流量整形策略中起到什么作用？</span>


- 对访客的一次访问，我们通过以上代码可以来判断此次访问是否超过了我们的限制，通过返回的判断结果，我们将对此次访问选择正确的处理策略，比如你可以拒绝消耗完令牌的访客进行访问，从而控制他的访问速率，从而达到网络流量整形的目的。

####2.  redis 在其中如何工作？</span>


- 对于每个独立的访客，redis 会为他建立两个 key，一个 key 保存了剩余令牌的数量，另外一个 key 保存了最近一次访问的时间戳。其中，最近一次访问时间戳在新访问到来时候用于计算时间间隔，从而计算在此时间间隔内应该向令牌桶中添加多少令牌，进而获得当前令牌桶的剩余令牌数。

####3. redis pipe 起到什么作用？


- 我们看到代码中 while 循环，执行了 redis pipe 中的 watch 动作，这是对 [redis 事务](http://redisbook.readthedocs.org/en/latest/feature/transaction.html) 的使用。 这使这里的算法能处理并发的来访。在 redis 中，事务执行是对 redis key 的一个加锁的操作，一个事务没有执行完，别的动作将无法操作这个 key ，代码中循环执行 watch 动作，就是去检查当前 key 是否有未执行完毕的事务，只有所有事务都执行的时候才可能进入执行体，完成令牌判断或者消耗。 ------ 这样避免了并发的访问在 set redis key 时候的混乱。

####4. 如何调参


- 代码中 RATE 和 DEFAULT 为主要参数，分别代表每秒钟消耗令牌的速率，和令牌桶的容量。通过调整这两个参数来控制你想要的访问速率。

###总结


- 这是一个实用的方式来完成网络流量整形，可以有效控制一些爆发式的流量访问，使访问更加平滑容易控制。
