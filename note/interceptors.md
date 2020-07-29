## ajax防重复点击  
<a href="/" style="float:right">回首页</a>
### 先放图，献丑了  
![图1](/img/runaway.gif)  
![图2](/img/shenteng.gif)  

### Demo  
1. get方法，重复请求时，保留最后一次，取消之前的请求。取消之前的请求，使用的是cancelToken  
2. post方法，重复请求时，保留第一次，取消后面的请求。取消后面的请求，使用的是直接调用Promise.reject()，直接打到instance.interceptors.response.use的error下  
3. 通过 md5.hex(url + JSON.stringify(params) + JSON.stringify(data))来判断是否是相同请求。计算后的key值放入request.header中，请求完成后可以直接获取到  
4. 一定要重复请求的接口可以在参数中添加随机数，或者用全局白名单  

```javascript  
import axios from 'axios'
import md5 from 'js-md5'

const DUPLICATE_CANCELED_MESSAGE = 'canceled because pending'
const duplicateCanceledWhiteList = [
  '/v1/admin/user/myself'
]
const duplicateCanceledMessageWhiteList = [
  '/v1/admin/user/myself'
] // 这个白名单是用来控制提示信息显示的，不是请求的白名单

const instance = axios.create()
const cancelToken = axios.CancelToken;
const pendingGetQueue = {}  // get请求的pending队列
const pendingPostQueue = {}  // post请求的pending队列

const calReqKey = (url, params, data) => {
  return md5.hex(url + JSON.stringify(params) + JSON.stringify(data));
}

/**
 * 如果没有同一个请求在pending队列里，则添加进去
 * 如果已有同一个请求，则根据get/post有不同的做法
 *  get: 取消前一次请求，进行新的请求
 *  post: 取消本次请求
 * @param {*} config 
 */
const addPendingIfNot = (config) => {
  if (duplicateCanceledWhiteList.includes(config.url)) {
    return config
  }
  let myKey
  switch (config.method.toLowerCase()) {
    case 'get':
        myKey = calReqKey(config.url, config.params, config.data)
        config.headers._rid = myKey
        if (typeof pendingGetQueue[myKey] === 'function') {
          pendingGetQueue[myKey]({reason: DUPLICATE_CANCELED_MESSAGE, type: 'get', url: config.url})
          delete pendingGetQueue[myKey]
        }
        config.cancelToken = new cancelToken((c) => {
          pendingGetQueue[myKey] = c
        })
      break
    case 'post':
      myKey = calReqKey(config.url, config.params, config.data)
      config.headers._rid = myKey       
      if (pendingPostQueue[myKey]) {
        return Promise.reject({message: {reason: DUPLICATE_CANCELED_MESSAGE, type: 'post', url: config.url}})
      } else {
        pendingPostQueue[myKey] = myKey
      }
      break
    default:
      break
  }
  return config
}

const removePending = (config) => {
  let myKey
  switch (config.method.toLowerCase()) {
    case 'get':
      if (Object.keys(pendingGetQueue).length) {
        myKey = config.headers._rid
        delete pendingGetQueue[myKey]
      }
      break
    case 'post':
      if (Object.keys(pendingPostQueue).length) {
        myKey = config.headers._rid
        delete pendingPostQueue[myKey]
      }
      break
    default:
      break
  }
}

instance.interceptors.request.use(
  config => {
    return addPendingIfNot(config)
  },
  err => {
    return Promise.reject(err)
  }
)

instance.interceptors.response.use(
  response => {
    //拦截响应，做统一处理 
    removePending(response.config)
    return response
  },
  //接口错误状态处理，也就是说无响应时的处理
  err => {
    if (err.message.reason === DUPLICATE_CANCELED_MESSAGE) {
      //可以通过设置白名单
      if (!duplicateCanceledMessageWhiteList.includes(err.message.url)) {
        Vue.prototype.$Message.warning('等待服务器响应中')
      }
    }
    return Promise.reject(err) // 返回接口返回的错误信息
  }
)

export default instance   
```  

其他问题：
1. 因网络等问题导致请求失败，会导致该已完成请求不能从队列中删除。未解决
2. 用在已有项目中，项目原先可能存在重复请求。未解决（有个思路：重复的请求暂存，当已发出去的请求响应后，使用其响应结果。或者干脆只拦post）