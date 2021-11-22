为了提升微服务的性能，我们可以选择将项目的静态资源都交给nginx进行转发，实现动静分离（不是前后端分离项目）。

同时，nginx还可以搭建域名访问环境，以谷粒商城为例，每种域名对应一种服务。search.gulimall.com、cart.gulimall.com等等，这些也可以通过nginx进行转发。

**nginx + windows搭建域名访问环境**

1、在nginx中的http块里面添加配置

```
upstream gulimall {
	server 192.168.118.128;
}
```

2、在nginx中的server块中配置

```
server_name gulimall.com *.gulimall.com 

location /static/ {
	root /usr/share/nginx/html; ##如果请求路径以static开头，就去该目录下寻找
}
location / {
	proxy_set_header Host $host; # nginx代理给网关的时候，会丢失请求的host信息
	proxy_pass http://gulimall;
} 
```

3、让nginx帮我们进行反向代理，所有来自gulimall的请求，都转到对应的服务

```yaml
- id: gulimail_host_route
  uri: lb://gulimail-product
  predicates:
    - Host=gulimall.com,item.gulimall.com

- id: gulimail_search_route
  uri: lb://gulimail-search
  predicates:
    - Host=search.gulimall.com
```

4、在window的host文件里面进行配置

```
# gulimail
192.168.118.128 gulimall.com
192.168.118.128 search.gulimall.com
192.168.118.128 item.gulimall.com
192.168.118.128 auth.gulimall.com
192.168.118.128 cart.gulimall.com
192.168.118.128 member.gulimall.com
192.168.118.128 seckill.gulimall.com
192.168.118.128 order.gulimall.com
```

