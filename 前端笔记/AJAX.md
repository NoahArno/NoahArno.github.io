AJAX全称为Asynchronous JavaScript And XML，就是异步的JS和XML。通过AJAX可以在浏览器中向服务器发送异步请求，最大的优势：无刷新获取数据。AJAX不是新的编程语言，而是一种将现有的标准组合在一起使用的新方式。

**1、使用jQuery发送get、post、通用的ajax请求**

```js

$.get('http://127.0.0.1:8000/jquery-server', {a:100, b:200}, function(data){
    console.log(data);
},'json'); // 加了json会自动转换为对象

$.post('http://127.0.0.1:8000/jquery-server', {a:100, b:200}, function(data){
    console.log(data);
});

$.ajax({
    //url
    url: 'http://127.0.0.1:8000/jquery-server',
    //参数
    data: {a:100, b:200},
    //请求类型
    type: 'GET',
    //响应体结果
    dataType: 'json',
    //成功的回调
    success: function(data){
        console.log(data);
    },
    //超时时间
    timeout: 2000,
    //失败的回调
    error: function(){
        console.log('出错啦!!');
    },
    //头信息 需要该方法设置自定义请求头Access-Control-Allow-Headers
    headers: {
        c:300,
        d:400
    }
});
```

**2、使用axios发送ajax请求**

```js
//配置 baseURL
axios.defaults.baseURL = 'http://127.0.0.1:8000';

//GET 请求
axios.get('/axios-server', {
    //url 参数
    params: {
        id: 100
    },
    //请求头信息
    headers: {
        name: 'atguigu',
    }
}).then(value => {
    console.log(value);
});

axios.post('/axios-server', {
    username: 'admin',
    password: 'admin'
}, {
    params: {
        id: 200
    },
    //请求头参数
    headers: {
        height: 180
    }
});

axios({
    //请求方法
    method : 'POST',
    //url
    url: '/axios-server',
    //url参数
    params: {
        vip:10,
        level:30
    },
    //头信息
    headers: {
        a:100,
        b:200
    },
    //请求体参数
    data: {
        username: 'admin',
        password: 'admin'
    }
}).then(response=>{
    console.log(response.status); //响应状态码
    console.log(response.statusText);    //响应状态字符串
    console.log(response.headers);    //响应头信息
    console.log(response.data);    //响应体
})
```

**3、fetch发送ajax请求**

 https://developer.mozilla.org/zh-CN/docs/Web/API/WindowOrWorkerGlobalScope/fetch

```js
fetch('http://127.0.0.1:8000/fetch-server?vip=10', {
    //请求方法
    method: 'POST',
    //请求头
    headers: {
        name:'atguigu'
    },
    //请求体
    body: 'username=admin&password=admin'
}).then(response => {
    // return response.text();
    return response.json();
}).then(response=>{
    console.log(response);
});
```

**4、同源策略**

同源：协议、域名、端口号必须完全相同，违背同源策略就是跨域。

**5、CORS**

CORS（Cross-Origin Resource Sharing），跨域资源共享。CORS是官方的跨域解决方案，它的特点是不需要在客户端做任何特殊的操作，完全在服务器中进行处理，支持get和post请求。跨域资源共享标准新增了一组HTTP首部字段，允许服务器声明哪些源站通过浏览器有权限访问哪些资源

CORS通过设置一个响应头来告诉浏览器，以请求允许跨域，浏览器收到该响应以后就会对响应放行。

```java
@Configuration
public class CorsConfig {
    @Bean
    public CorsWebFilter corsFilter() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowCredentials(true); //是否允许携带cookie
        config.addAllowedOrigin("*"); //可接受的域，是一个具体域名或者*（代表任意域名）
        config.addAllowedHeader("*"); //允许携带的头
        config.addAllowedMethod("*"); //允许访问的方式

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);

        return new CorsWebFilter(source);
    }
}
```

