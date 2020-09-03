# Overview

Kubernetes使用该算法的地方较多，例如Client-go里面的request函数，workqueue中的限速队列等。

为什么把该算法拿出来建，是因为限流是非常重要的设计，为了避免服务过载，在没有使用service mesh的情况下我们需要限流来保护服务。

# 概念

令牌桶算法（BucketRateLimiter）: 内部实现一个存放token的桶，开始的时候桶是空的，token会以**固定速率**往桶里面填充token，直到填满为止，多余的会被丢弃，每一个进入桶里面的元素都会拿到一个token，只有得到token的才会被通过，否则就等到。 令牌桶是通过控制发放token来达到**限速**的目的。

令牌桶算法在Kubernetes中是以使用三方库`golang.org/x/time/rate`来实现的。

# 原理

![](C:\Users\EZLIUJA\Desktop\workspace\k8s-arithmetic\images\bucketratelimiter.png)

# 使用

### 构造一个限流器对象

第一个参数是`r Limit`，也就是每秒可以往桶里面填充token数

第二个参数是`b int`，也就是桶的容量（即令牌桶最多存放的 token 数量）

```go
limiter := NewLimiter(10, 100);
```



### Wait/WaitN

```
func (lim *Limiter) Wait(ctx context.Context) (err error)
func (lim *Limiter) WaitN(ctx context.Context, n int) (err error)
```

当使用 Wait 方法消费 Token 时，如果此时桶内 Token 数组不足 (小于 N)，那么 Wait 方法将会阻塞一段时间，直至 Token 满足条件。如果充足则直接返回。

这里可以看到，Wait 方法有一个 context 参数。
我们可以设置 context 的 Deadline 或者 Timeout，来决定此次 Wait 的最长时间。



### Reserve/ReserveN

```go
func (lim *Limiter) Reserve() *Reservation
func (lim *Limiter) ReserveN(now time.Time, n int) *Reservation
```

ReserveN 的用法就相对来说复杂一些，当调用完成后，无论 Token 是否充足，都会返回一个 Reservation * 对象。

你可以调用该对象的 Delay() 方法，该方法返回了需要等待的时间。如果等待时间为 0，则说明不用等待。
必须等到等待时间之后，才能进行接下来的工作。

或者，如果不想等待，可以调用 Cancel() 方法，该方法会将 Token 归还。

# Kubernetes使用令牌桶例子

## Client-go中的request使用

这里使用了令牌桶算法，主要是当下游服务出问题，重试调用下游服务的request的时候限速。

代码块`staging/src/k8s.io/client-go/rest/request.go`

```go
func (r *Request) request(ctx context.Context, fn func(*http.Request, *http.Response)) error {
	// 获取这次请求从开始到结束的latency的metrics 暴露给prometheus
	start := time.Now()
	defer func() {
		metrics.RequestLatency.Observe(r.verb, r.finalURLTemplate(), time.Since(start))
	}()


	if err := r.requestPreflightCheck(); err != nil {
		return err
	}

	client := r.c.Client
	if client == nil {
		client = http.DefaultClient
	}


    // 第一次的时候，request不需要等待
	if err := r.tryThrottle(ctx); err != nil {
		return err
	}

	if r.timeout > 0 {
		var cancel context.CancelFunc
		ctx, cancel = context.WithTimeout(ctx, r.timeout)
		defer cancel()
	}

    // 重试机制
	retries := 0
    // 循环控制
	for {
		url := r.URL().String()
		req, err := http.NewRequest(r.verb, url, r.body)
		req = req.WithContext(ctx)
		req.Header = r.headers
		r.backoff.Sleep(r.backoff.CalculateBackoff(r.URL()))
        // 如果retries大于0，说明之前至少已经尝试过一次request发送给api server了
		if retries > 0 {
			// 交给tryThrottle去判断拿到这次token等待的时间
			if err := r.tryThrottle(ctx); err != nil {
				return err
			}
		}
        // 获取response
		resp, err := client.Do(req)
		updateURLMetrics(r, resp, err)
		...

		done := func() bool {
			defer func() {
				const maxBodySlurpSize = 2 << 10
				if resp.ContentLength <= maxBodySlurpSize {
					io.Copy(ioutil.Discard, &io.LimitedReader{R: resp.Body, N: maxBodySlurpSize})
				}
				resp.Body.Close()
			}()

			retries++
            // 通过checkWait检查 response的返回码，只要不是500以上或者429就重试
			if seconds, wait := checkWait(resp); wait && retries <= r.maxRetries {
				if seeker, ok := r.body.(io.Seeker); ok && r.body != nil {
					_, err := seeker.Seek(0, 0)
					
				}
				// 计算重试的等待时间
				r.backoff.Sleep(time.Duration(seconds) * time.Second)
				return false
			}
			fn(req, resp)
			return true
		}()
		if done {
			return nil
		}
	}
}

```



这里特地把tryThrottle方法，这个方法就是使用了令牌桶来做限流，r.rateLimiter.Wait会阻塞直到获取到令牌

```go
func (r *Request) tryThrottle(ctx context.Context) error {
	if r.rateLimiter == nil {
		return nil
	}

	now := time.Now()

    // 获取这次request的上下文，根据上下文返回拿到这次token需要等待的时间
	err := r.rateLimiter.Wait(ctx)

	latency := time.Since(now)
	if latency > longThrottleLatency {
		klog.V(3).Infof("Throttling request took %v, request: %s:%s", latency, r.verb, r.URL().String())
	}
	if latency > extraLongThrottleLatency {
		// If the rate limiter latency is very high, the log message should be printed at a higher log level,
		// but we use a throttled logger to prevent spamming.
		globalThrottledLogger.Infof("Throttling request took %v, request: %s:%s", latency, r.verb, r.URL().String())
	}
	metrics.RateLimiterLatency.Observe(r.verb, r.finalURLTemplate(), latency)

	return err
}
```

Wait 方法来通过获取上下文计算需要等待的时间

```go
func (t *tokenBucketRateLimiter) Wait(ctx context.Context) error {
	return t.limiter.Wait(ctx)
}
```



## Workqueue中的限速队列

在Workqueue中是延迟把元素插入到FIFO队列中。

代码块`staging/src/k8s.io/client-go/util/workqueue/default_rate_limiters.go`

默认的清空下就实例化令牌桶实现的，以固定速率往桶里面插入元素，被插入的元素都会拿到一个token，以此来达到限制速度的目的。

```go
func DefaultControllerRateLimiter() RateLimiter {
	return NewMaxOfRateLimiter(
		NewItemExponentialFailureRateLimiter(5*time.Millisecond, 1000*time.Second),
		// 10 qps, 100 bucket size.  This is only for retry speed and its only the overall factor (not per item)
		&BucketRateLimiter{Limiter: rate.NewLimiter(rate.Limit(10), 100)},
	)
}

func (r *BucketRateLimiter) When(item interface{}) time.Duration {
	return r.Limiter.Reserve().Delay()
}
```

(rate.Limit(10), 100)

第一个参数表示每秒往“桶”里填充的 token 数量

第二个参数表示令牌桶的大小（即令牌桶最多存放的 token 数量）

这里我们可以看见Workqueue是以固定的速率： 每秒往桶里面10填充10个token，然后调用了Reserve().Delay()来计算需要等待的时间。





# Reference

https://www.cyhone.com/articles/analisys-of-golang-rate/

https://www.cyhone.com/articles/usage-of-golang-rate/

