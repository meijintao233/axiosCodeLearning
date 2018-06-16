记录之前学习axios的源码，加深印象
==========================

#### axios的核心代码在lib/core/、lib/adapters/、lib/cancel/、lib/axios.js、lib/defaults中。

```javascript
    //lib/core/Axios.js
    
    //axios的构造函数，
    //把defaults、interceptors挂载到axios实例中
    //其中的request和response就是请求和响应的拦截器，通过interceptorManager进行管理
    function Axios(instanceConfig) {
      this.defaults = instanceConfig;
      this.interceptors = {
        request: new InterceptorManager(),
        response: new InterceptorManager()
      };
    }
```

```javascript
//Axios原型中的request函数，这是axios发起请求时使用的函数
//config就是Axios内部传递信息的载体
Axios.prototype.request = function request(config) {
  
  if (typeof config === 'string') {
    //发起请求时用的axios('url'[, config])，就是在这里处理的
    config = arguments[1] || {};
    config.url = arguments[0];
  } else {
    
    config = config || {};
  }

  //合并defaults和config的值，config的优先级高于defaults
  config = mergeConfig(this.defaults, config);
  //传入的config对象中如果设置了method属性就把值转为小写，否则就设置为get请求
  //也就是axios默认发起get请求  
  config.method = config.method ? config.method.toLowerCase() : 'get';

  //这里是拦截器，整个过程就像是通过一条水管，
  //请求拦截-》发起请求-》响应拦截-》响应
  
  //这里dispatchRequest是发起请求，而undefined的作用是占位
  var chain = [dispatchRequest, undefined];
  //返回一个promise对象，方便控制异步的过程
  var promise = Promise.resolve(config);
    
  //这里是InterceptorManager中的foreach方法，遍历InterceptorManager的handlers数组，
  //对每个元素执行传入的函数，
  //将handlers数组中的拦截器对象从chain数组开头压入
  this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {
    chain.unshift(interceptor.fulfilled, interceptor.rejected);
  });
  //将handlers数组中的拦截器对象从chain数组末尾压入
  this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {
    chain.push(interceptor.fulfilled, interceptor.rejected);
  });
    
  //chain数组实现了拦截器的执行,压入拦截器后数组变成
  //[
  // request.fulfilled,request.rejected,request.fulfilled,request.rejected...,
  // dispatchRequest, undefined,
  // responce.fulfilled,responce.rejected,responce.fulfilled,responce.rejected...,
  //]
  //chain中有元素时，就成对的从数组开头弹出并执行
  //实现  请求拦截-》发起请求-》响应拦截-》响应
  while (chain.length) {
    promise = promise.then(chain.shift(), chain.shift());
  }
  //最后返回promise对象，便于链式调用
  return promise;
};
```

#### 接下来就是拦截器管理的模块，主要有三个功能，use方法注册拦截器，eject方法移除拦截器，forEach方法遍历执行handlers中的元素

```javascript
//lib/core/InterceptorManager.js

//把handlers挂载在InterceptorManager的实例中
//handlers存放的是拦截器执行函数
function InterceptorManager() {
  this.handlers = [];
}

//将拦截器执行函数推入handlers中，返回的是handlers的长度
InterceptorManager.prototype.use = function use(fulfilled, rejected) {
  this.handlers.push({
    fulfilled: fulfilled,
    rejected: rejected
  });
  return this.handlers.length - 1;
};

//移除拦截器执行函数，通过将数组相应位置的元素置空实现
InterceptorManager.prototype.eject = function eject(id) {
  if (this.handlers[id]) {
    this.handlers[id] = null;
  }
};

//遍历handlers数组，传入一个函数作为参数，将每一个存在的元素作为传入的函数的参数执行
InterceptorManager.prototype.forEach = function forEach(fn) {
  utils.forEach(this.handlers, function forEachHandler(h) {
    if (h !== null) {
      fn(h);
    }
  });
};
```



```javascript
//lib/core/dispatchRequest.js

function throwIfCancellationRequested(config) {
  if (config.cancelToken) {
    config.cancelToken.throwIfRequested();
  }
}

//请求前对config进行最后一次处理
module.exports = function dispatchRequest(config) {
  throwIfCancellationRequested(config);

  //将config中的baseURL和url组合成完整的url
  if (config.baseURL && !isAbsoluteURL(config.url)) {
    config.url = combineURLs(config.baseURL, config.url);
  }

  
  config.headers = config.headers || {};

  //调用config对象的transformRequest中的函数对data和headers进行处理
  config.data = transformData(
    config.data,
    config.headers,
    config.transformRequest
  );

  
  config.headers = utils.merge(
    config.headers.common || {},
    config.headers[config.method] || {},
    config.headers || {}
  );

  utils.forEach(
    ['delete', 'get', 'head', 'post', 'put', 'patch', 'common'],
    function cleanHeaderConfig(method) {
      delete config.headers[method];
    }
  );

  //根据执行环境进行请求，nodejs中用http，浏览器中用xhr
  var adapter = config.adapter || defaults.adapter;

  return adapter(config).then(function onAdapterResolution(response) {
    throwIfCancellationRequested(config);

    //调用config对象的transformResponce中的函数对data和headers进行处理
    response.data = transformData(
      response.data,
      response.headers,
      config.transformResponse
    );

    return response;
  }, function onAdapterRejection(reason) {
    if (!isCancel(reason)) {
      throwIfCancellationRequested(config);
        
 	  //调用config对象的transformResponce中的函数对data和headers进行处理
      if (reason && reason.response) {
        reason.response.data = transformData(
          reason.response.data,
          reason.response.headers,
          config.transformResponse
        );
      }
    }

    return Promise.reject(reason);
  });
};

```
