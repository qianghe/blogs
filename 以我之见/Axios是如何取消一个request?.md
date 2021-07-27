## Axios是如何取消一个request?

前端同学对axios都非常熟悉，它是活跃在大家项目里负责网络请求的热门库。在业务项目中，我们通常会配置化axios的使用实例（如指定baseUrl、timeout等），方便好用的request和response拦截器也提供了通用逻辑处理的方式。

今天想谈论的一个问题是，axios是如何取消一个已发送的request的呢？

##### 问题场景

那为什么需要处理这个问题呢？我们平时遇到的哪些业务场景需要这个方法呢？

* 用户input输入频繁请求；

* 用户选择组合式表单触发实时频繁的请求；

* 用户频繁切换tab获取activeTab下的数据；

  ...

这些问题统一为：用户行为频繁触发 -> 引发数据store变更 -> 触发store变更的监听 -> 触发网络请求

##### 问题分析

当我们分析一个问题的解决思路时，我们必须清楚我们面临的问题，其触发的流程是怎么样的（步骤）、问题可以在哪些点被规避（关键点）、思考规避的方法并实践（方案比较）。

那在这个问题中，我找到的两个点是：

1. 减少用户高频率触发请求，使用debounce（比如在100ms内只能触发3次）；
2. 对于相当store的触发的情况下，新request发送前取消之前的request；

对于第二点，对于不需要考虑网络资源的场景来说，可以不考虑cancel的方式，可以思考如果获取最新一次的网络请求结果。

##### Axios的cancel是如何实现的？

那如何取消一个网络请求呢？我们知道axios其内部实现的adapter（brower端）使用的是xhr（node使用的是http模块）。而原生xhr是有提供取消网络请求事件的方法，其名曰：abort。看下MDN上的描述：

> 如果该请求已被发出，**XMLHttpRequest.abort()** 方法将终止该请求。当一个请求被终止，它的  [`readyState`](https://developer.mozilla.org/zh-CN/docs/Web/API/XMLHttpRequest/readyState) 将被置为 `XMLHttpRequest.UNSENT`，并且请求的 [`status`](https://developer.mozilla.org/zh-CN/docs/Web/API/XMLHttpRequest/status) 置为 0。

再看下其兼容性：Full support，因此我们可以暂时不考虑polyfill的方法，通过调用axios提供的cancel方法即可解决问题。

我们可以思考一下，如果让我们来实现一个这样的场景，我们会怎么做？

```javascript
// 创建一个xhr
const xhr = new XMLHttpRequest()
xhr.open('post', data)
// ...参数配置
xhr.onreadystatechange = () => {
  if(xhr.readyState === XMLHttpRequest.DONE && xhr.status === 200) {
    console.log(xhr.responseText)
  }
}
xhr.send()

// 使用者打算取消，直接调用abort就行了
xhr.abort()
```

如果是使用原生xhr就是这么的简单，那使用Axios呢？它的内部是如何处理的呢？也很简单。

看一个官网的demo：

```javascript
const CancelToken = axios.CancelToken;
let cancel;

axios.get('/user/12345', {
  cancelToken: new CancelToken(function executor(c) {
    // An executor function receives a cancel function as a parameter
    cancel = c;
  })
});

// cancel the request
cancel();
```

(两种方式，或者通过cancelToken的source方法获取token + cancel，本质都是相同的)。

这个CancelToken通过实例化的方法提供了一个token，axios通过配置信息获取这个token后，当前发起的request就和token创建了关联；而当用户调用cancel的方法时，这个request就会被取消。那token到底是什么呢？我们先看下CancelToken的源码：

```javascript
function CancelToken(executor) {
  if (typeof executor !== 'function') {
    throw new TypeError('executor must be a function.');
  }

  var resolvePromise;
  this.promise = new Promise(function promiseExecutor(resolve) {
    resolvePromise = resolve;
  });

  var token = this;
  executor(function cancel(message) {
    if (token.reason) {
      // Cancellation has already been requested
      return;
    }

    token.reason = new Cancel(message);
    resolvePromise(token.reason);
  });
}
```

实例化一个CancelToken，相当于得到了一个promise（即配置中的cancelToken），而这个promise的resolve其实是交给了cancel函数，即当用户调用cancel函数时，这个promise才被resolved。那这个逻辑就很容易理解了，用户调用cancel，promise被resolved，可以通过promise.then(() =>{ xhr.abort() })来实现请求的取消。

##### 业务中如何处理

我们需要一个service帮助我们收集某个url发起的request请求，如果该请求时需要进行debouceCancel处理的，那我们就

* 在配置中为其自动添加cancelToken；
* 并且有一个栈去存储当前url对应的cancelToken（promise）；
* 为了满足多个url的处理，我们需要一个map结构存储信息<url: string, token: Pomise[]>;
* 当有一个新的请求发送时，我们将其拦截，查看map结构获取之前的请求token序列，调用所有token的cancel方法，取消xhr。

首先，我们要拦截request，通过axios提供的interceptors.request.use：

```javascript
axios.interceptors.request.use((config) => {
  const { cancel, url } = config
  // 自定义配置，是否需要debounceCancel
  if (!cancel) return config

  requestCancelService.register(url)
	
  return requestCancelService.proxyRequst(config)
})
```

requestCancelService的实现：

```javascript
import Axios from 'axios'
import { isEmpty } from 'lodash-es'

class RequestCancelService {
  constructor() {
    this.map = new Map()
  }
  // 注册url
  register(url) {
    if (this.map.has(url)) return
    this.map.set(url, [])
  }

  clearCancelResource(url) {
    this.map.set(url, [])
  }
  // 添加
  addCancelResource(url, source) {
    this.map.get(url)?.push(source)
  }

  proxyRequst(axiosConfig) {
    const { url } = axiosConfig
    const cancelResources = this.map.get(url)

    if (!isEmpty(cancelResources)) {
      const copyCancelResources = [...cancelResources]

      this.clearCancelResource()
			// 取消之前的request
      copyCancelResources.forEach((cancelInvoker) => {
        cancelInvoker?.cancel()
      })
    }
    const CancelToken = Axios.CancelToken
    const source = CancelToken.source()
    // create new cancelToken
    axiosConfig.cancelToken = source.token

    this.addCancelResource(url, source)

    return axiosConfig
  }
}

const requestCancelServiceInstance = new RequestCancelService()

export default requestCancelServiceInstance
```

##### 总结

综上所述，描述了：

* 在什么场景下我们需要取消一个request；
* axios如何使用xhr.abort来取消一个request的；
* 我们在业务中如果通用化axios配置帮助解决这个问题的。







