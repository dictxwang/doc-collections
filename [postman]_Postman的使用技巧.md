# Postman的使用技巧

本文主要记录postman使用中的一些技巧性方法。

Postman可以通过定义环境变量，非常便捷的区分不同的环境（开发环境和测试环境等）的请求地址。还可以通过编写Pre-request Script实现请求的预处理，比如动态设置header值或者参数值。

#### 技巧一、在collection节点定义全局方法

##### 1、在collection的Pre-request Script中定义方法，并设置到全局变量中

```javascript
// 添加依赖包
let Header = require('postman-collection').Header;
let setOcmSgwParam = (apiKey) =>  {
    // 设置计算header参数
    let ts = Math.round(new Date().getTime() / 1000);
    let token = CryptoJS.MD5(apiKey + "#_#" + ts);;
    // 设置header
    pm.request.headers.add(new Header("ocmSgwApiKey:" + apiKey));
    pm.request.addHeader(new Header("ocmSgwTs:" + ts));
    pm.request.headers.append(new Header("ocmSgwToken:" + token));
}

// 设置为全局方法，便于内部的request调用
postman.setGlobalVariable("setOcmSgwParam", "(function() {return " + setOcmSgwParam + ";})()")
```

##### 2、在request的Pre-request Script中调用方法

```javascript
// 添加依赖包
let Header = require('postman-collection').Header;
// 调用全局方法设置请求头部
eval(globals.setOcmSgwParam)("interfaceSampleApi.methodSample");
```

#### 技巧二、通过Pre-request Script设置请求参数

##### 1、在collection的Pre-request Script中定义方法，并设置成全局变量

```javascript
let setOssAgentToken = () =>  {
    // 设置计算header参数
    let ts = Math.round(new Date().getTime());
    let digest = CryptoJS.MD5(ts + "_xxxyyy");
    let token = ts + "_" + digest;
    // 设置request参数
    pm.request.addQueryParams("token=" + token);
}

// 设置为全局方法，便于内部的request调用
postman.setGlobalVariable("setOssAgentToken", "(function() {return " + setOssAgentToken + ";})()")
```

##### 2、在request的Pre-request Script中调用全局方法

```javascript
eval(globals.setOssAgentToken)();
```

