<a href="https://zhuanlan.zhihu.com/p/86953757?utm_source=wechat_session&utm_medium=social&utm_oi=1052569979358179328&from=singlemessage">如何解决 Keep-Alive 导致 ECONNRESET 的问题</a>

- nodejs 作为客户端发起请求，使用 http.request / http.get 方法，默认使用 globalAgent ，也可以自己指定 agent，详见 node 文档

- 如果使用 rpc 服务，需要建立长链接，即 keep-alive ，可以在创建 agent 实例的时候传入参数，见 nodejs 文档

- 在使用 keep-alive 时，会经常出现错误：ECONNRESET 或者 socket hang up 

#### 解决办法

在 nodejs 13.0.1 版本中，http.ClientRequest 类增加了 reusedSocket 属性，可以判断是否是 ECONNRESET 报错，从而进行重连，用法如下

```

const http = require('http');
const agent = new http.Agent({ keepAlive: true });

function retriableRequest() {
  const req = http
    .get('http://localhost:3000', { agent }, (res) => {
      // ...
    })
    .on('error', (err) => {
      // Check if retry is needed
      if (req.reusedSocket && err.code === 'ECONNRESET') {
        retriableRequest();
      }
    });
}

retriableRequest();
```

