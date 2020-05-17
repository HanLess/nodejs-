### 压测工具：ab

### 性能监控工具：v8-profiler-node8

### 性能报告查看工具：chrome -> 右上角三个点的 -> more tools -> javascript profiler

javascript profiler self time：此方法执行占用的时间

javascript profiler total time：此方法执行 和 它调用的方法执行，共占用的时间

当发现 cpu 使用率过高后，可以使用 v8-profiler-node8 来进行运行报告的生成，示例代码如下

```

const profiler = require('v8-profiler-node8')

profiler.startProfiling('CPU profile')

router.get('/compute', function() {
    // 业务逻辑
})

// 对业务逻辑进行压测后，访问此接口，生成报告
router.get('/profile', function() {
  var profile = profiler.stopProfiling('');
      profile.export().pipe(fs.createWriteStream(`cpuprofile-${Date.now()}.cpuprofile`))
      .on('finish', () => profile.delete())

    ctx.status = 200
  ctx.body = 'profile'
  
})

```

### 最终生成 cpuprofile 报告，可以通过 chrome 进行分析

<a href="https://zhuanlan.zhihu.com/p/60265439">Node.js调试之性能测试篇（二）</a>

（1）这个案例中的问题是使用了同步的方法做 cpu 密集型操作，改成异步即可，改为异步后，libuv 会对此 cpu 密集型运算单开线程处理

（2）在分析报告的过程中，面对 gc 占用时间过长，需要分情况讨论，如果真的因为新生代空间问题导致，可以进行 gc 参数设置，但大部分还是因为业务代码的问题，如上面的这个案例是因为：一个体积比较大的对象被频繁的重新赋值，就会频繁的 gc，导致时间占用过长


（3）除此之外，这个案例还有一个因素导致 qps 过低，即 idle 时间过长，即 cpu空闲时间很长，这是因为 ab 压测的参数设置不对

```
ab -k -c 20 -n 2000 "http://localhost:3000/string"
```

20个并发，共发 2000 次请求，会导致每次发 20 个，分 100 次发，每次都需要等上一批请求成功后再发下一批，这就导致会有大量空闲时间

在做压测的时候，要避免此类问题




