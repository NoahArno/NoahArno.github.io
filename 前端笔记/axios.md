**1、axios的特点**

1. 基于xhr + promise的异步ajax请求库
2. 浏览器端/node端都可以使用
3. 支持请求/响应拦截器
4. 支持请求取消
5. 请求/响应数据转换
6. 批量发送多个请求

**2、axios常用语法**

1. axios(config)：通用/最本质的发任意类型请求的方式
2. axios(url [, config])：可以只指定url发get请求
3. axios.request(config)：等同于axios(config)
4. axios.get(url [, config])：发get请求
5. axios.defaults.xxx: 请求的默认全局配置
6. axios.interceptors.request.use(): 添加请求拦截器
7. axios.interceptors.response.use(): 添加响应拦截器
8. axios.create([config]): 创建一个新的 axios(它没有下面的功能 )
9. axios.Cancel(): 用于创建取消请求的错误对象
10. axios.CancelToken(): 用于创建取消请求的 token对象
11. axios.isCancel(): 是否是一个取消请求的错误
12. axios.all(promises): 用于批量执行多个异步请求
13. axios.spread(): 用来指定接收所有成功数据的回调函数的方法

**3、基本使用**

```js
axios({
    method: 'delete',
    url: 'http://localhost:3000/posts/3',
}).then(response => {

})

=============================
// 还可以添加默认配置
//默认配置
axios.defaults.method = 'GET';//设置默认的请求类型为 GET
axios.defaults.baseURL = 'http://localhost:3000';//设置基础 URL
axios.defaults.params = {id:100};
axios.defaults.timeout = 3000;//

btns[0].onclick = function(){
    axios({
        url: '/posts'
    }).then(response => {
        console.log(response);
    })
}
```

**4、创建实例对象**

```js
const duanzi = axios.create({
    baseURL: 'https://api.apiopen.top',
    timeout: 2000
});

// 这个duanzi和axios对象的功能几乎是一样的
duanzi.get('/getJoke').then(response => {
    
});
```

**5、拦截器**

```js
axios.interceptors.request.use(function (config) {
    console.log('请求拦截器 成功');
    //修改 config 中的参数
    config.params = {a:100};

    return config;
}, function (error) {
    console.log('请求拦截器 失败');
    return Promise.reject(error);
});

// 设置响应拦截器
axios.interceptors.response.use(function (response) {
    console.log('响应拦截器 成功 1号');
    return response.data;
    // return response;
}, function (error) {
    console.log('响应拦截器 失败 1号')
    return Promise.reject(error);
});
```

**6、请求取消**

```js
//获取按钮
const btns = document.querySelectorAll('button');
//2.声明全局变量
let cancel = null;
//发送请求
btns[0].onclick = function(){
    //检测上一次的请求是否已经完成
    if(cancel !== null){
        //取消上一次的请求
        cancel();
    }
    axios({
        method: 'GET',
        url: 'http://localhost:3000/posts',
        //1. 添加配置对象的属性
        cancelToken: new axios.CancelToken(function(c){
            //3. 将 c 的值赋值给 cancel
            cancel = c;
        })
    }).then(response => {
        console.log(response);
        //将 cancel 的值初始化
        cancel = null;
    })
}

//绑定第二个事件取消请求
btns[1].onclick = function(){
    cancel();
}
```

