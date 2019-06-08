## 聊聊fetch

### fetch的使用

fetch是一个发起异步请求的新api, 在浏览器(有些不支持)中可以直接使用。

Promise fetch(url, init)

fetch接收两个参数，第一个参数是请求路径，第二个参数是可选的。

fetch函数返回的是一个promise对象。

一个简单使用的例子:

```javascript
let url = "http://localhost:8080/user/list";
fetch(url)
    .then(response => {
    	// 检查是否正常返回
        if(response.status >= 200 && response.status < 300) {
            // 返回的是一个promise对象, 值就是后端返回的数据, 调用then()可以接收
            return response.json();
        }
        const err = new Error(response.statusText);
        err.response = response;
        throw err;
	})
    .then(data => {
    	// 后端返回的JSON数据
    	console.log(data)
     })
    .catch(err => console.log(err));
```

因为fetch返回的是一个promise对象, 因此可以链式调用then、catch等函数。

第一次调用then()拿到的只是一个Response对象，这个对象里面有一系列属性，以及方法。

具体可以参考: [Response API](https://developer.mozilla.org/zh-CN/docs/Web/API/Response)

这里主要说说status、statusText、ok、body这几个属性

status: 就是服务器返回的状态码(200、400、500)

statusText: 与status对应的文本显示

ok: 一个bool类型的值, 如果成功(状态码是200-299之间)返回就是true， 否则就是false

body: 响应的数据, 无法直接读取，需要使用response.json()、response.text()等方法解析。

通常后端返会的都是JSON数据, 一般使用response.json()即可。response.json()返回的还是一个Promise

对象，接着再调用then()就能获取到后端返回的数据了。  

**注: 使用fetch发起异步请求时，只有当网络等原因fetch返回的promise才会变成reject状态，而一般的**

**错误诸如404、500等还是会变成resolve状态, 因此需要再第一个then函数里面检查是否是成功的返回。**

**可以 检查状态码或者ok属性是否为true。**

### fetch的第二个参数init

这个参数是可选的, 如果不传这个参数，那么fetch默认使用get方式发起异步请求。

init是一个对象，格式可以参考: [fetch API](https://developer.mozilla.org/zh-CN/docs/Web/API/WindowOrWorkerGlobalScope/fetch) 下面主要列举几个常用的属性。

method: 请求方式(get、post、put等)

headers: 请求头信息

body: 请求参数，对get方式的请求无效，因为get方式请求的参数在url上 。body的格式与headers设置的

Content-Type参数对应。如Content-Type=application/json，那么body是一个JSON格式。

如Content-Type=application/x-www-form-urlencoded(表单默认的形式就是这个)，那么body就是一个键值

对形式，诸如: username=wangtao&password=123456这样。

Content-Type可以不设置，会根据body格式自动设置Content-Type的值

mode: 请求的模式，与跨域有关系。有cors、no-cors、same-origin 3个值

cors即使用跨域资源共享这种方式实现跨域，需要服务端支持。**这种方式不支持Content-Type=application/json**

no-cors也是可以跨域的，但是拿不到返回的值。

same-origin不支持跨域。

### 与generator函数结合

在redux-saga中使用往往都是通过使用generator函数结合实现异步调用。

```js
function request(url) {
    return fetch(url)
        .then(response => {
            if(response.ok) {
                return response.json();
            }
            const err = new Error(response.statusText);
            err.response = response;
            throw err;
        })
        .then(result => ({result}))
        .catch(err => ({err}));
}
// 非常像resux-saga处理异步请求的写法
function* asyncFetch() {
    const {result, err} = yield request("http://127.0.0.1/user/list")
    if(result) {
        // 执行逻辑, 展示数据
    	console.log(result)
    } else if(err) {
        //处理错误
    }
}

//调用asyncFetch（redux-saga自己调用, 不用手写）
const g = asyncFetch();
let next = g.next();
if(!next.done) {
    next.value.then(data => {
        g.next(data)
    })
}
```

上述代码主要用来理解redux-saga处理异步请求的写法，只是一个雏形。

### 总结

使用fetch需要注意的几个点

- 服务端返回错误的状态码，fetch返回的promise也是resolve的，因此需要加判断。

- 如果请求方式是get，fetch的第二参数的body属性无效。

- 如果跨域需要指定第二个参数的mode=cors，需要后端支持(Access-Control-Allow-Origin = "*")

  此种方式不支持Content-Type=application/json。