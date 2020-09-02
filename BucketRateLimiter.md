# 概念

令牌桶算法（BucketRateLimiter）: 内部实现一个存放token的桶，开始的时候桶是空的，token会以固定速率往桶里面填充token，直到填满为止，多余的会被丢弃，每一个进入桶里面的元素都会拿到一个token，只有得到token的才会被通过，否则就等到。 令牌桶是通过控制发放token来达到限速的目的。

Kubernetes使用该算法的地方较多，例如Client-go里面的request函数，workqueue中的限速队列等

令牌桶算法在Kubernetes中是以使用三方库`golang.org/x/time/rate`来实现的。

# 原理

![1599050241104](C:\Users\EZLIUJA\AppData\Roaming\Typora\typora-user-images\1599050241104.png)

# 使用

## client-go中的request使用





## workqueue中的限速队列





