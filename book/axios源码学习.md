记录之前学习axios的源码，加深印象
==========================

<br>


axios的核心代码在lib/core/、lib/adapters/、lib/cancel/、lib/axios.js、lib/defaults中。

<br>

```
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

