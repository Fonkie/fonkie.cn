---
title: 网站服务迁移引起的nginx转发错误
---
由于机器裁撤，网站服务`www.a.com`需要由机器x迁移到机器y，在迁移的过程中，发现访问`www.a.com`，得到始终是网站`www.b.com`的内容，why?

**问题排查**
追踪nginx错误日志显示，请求`http://www.a.com/`被`www.b.com`的nginx配置命中
```
root@...:/usr/local/nginx/logs
# tail -f error.log
2018/11/01 22:22:27 [error] 3022#0: *1 connect() failed (111: Connection refused) while connecting to upstream, client: 100.xxx.xxx.53 , server: www.b.com, request: "GET / HTTP/1.0", upstream: "http://10.yyy.yyy.yyy:9070/", host: "www.a.com"
......
```
`www.b.com`的nginx配置
```
root@10.yyy.yyy.230:/usr/local/nginx/conf/vhosts
# cat www.b.com.conf
server {
        listen 10.yyy.yyy.230:80;
        server_name www.b.com;
        ......
}
```
`www.a.com`的nginx配置
```
root@10.yyy.yyy.230:/usr/local/nginx/conf/vhosts
# cat www.a.com.conf
server {
        listen 80;
        server_name www.a.com;
        ......
}
```
**解决方法**
将`www.a.com`的nginx配置`listen`修改为``IP:port`
```
root@10.yyy.yyy.230:/usr/local/nginx/conf/vhosts
# cat www.a.com.conf
server {
        listen 10.yyy.yyy.230:80;
        server_name www.a.com;
        ......
}
```

**原因分析**

网站配置文件`www.a.com.conf`从机器x，搬迁到机器y，不同的机器nginx其他网站服务配置风格不一致，机器10.xxx.xxx.15：
```
server {
        listen 80;
        ......
}
```
而机器10.yyy.yyy.230：
```
server {
        listen 10.yyy.yyy.230:80;
        ......
}
```

nginx使用block(如 `server block`, `location block`)来组织配置文件，在接收客户端请求之后会根据请求的domain，port，ip判断处理该请求的`server block`，然后根据请求的资源和URI决定处理该请求的`location block`。
一般nginx会有多个`server block`，每个`server block`中的`listen`定义了这个`server block`能处理的ip和port。nginx首先检查请求的ip和port，并按照优先级找出与之匹配的`server block`。
当端口相同时，listen的定义优先级从高到低排列如下：
```
listen 10.yyy.yyy.230:80;
listen 127.0.0.1:80;
listen 80;
```
如果匹配到多个`ip:port`，nginx将会继续检查`server_name`。

**测试案例**
`ip`,`port`,`server_name`三者优先级顺序分析
```
server {
    listen       192.168.8.81:80;
    server_name  vae.test1.com *.test1.com;
    index index.html;
    location /test1 {
       root /usr/local/nginx/html/;
    }
}
server {
    listen       127.0.0.1:80;
    server_name  vae.test1.com *.test1.com;<!--hosts文件配置的地址-->
    index index.html;
    location /test1 {
       root /usr/local/nginx/html/;
    }
}
server {
    listen       80;
    server_name  vae.test1.com;<!--hosts文件配置的地址-->
    index index.html;
    location /test1 {
       root /usr/local/nginx/html/;
    }
}
server {
    listen       80;
    server_name  vae.test2.com;
    index index.html;
    location /test2 {
        root /usr/local/nginx/html/;
    }
}
server {
    listen       8081;
    server_name  vae.test3.com;
    index index.html;
    location /test3 {
        root /usr/local/nginx/html/;
    }
}
```

host配置：
```
192.168.8.81 vae.test1.com
192.168.8.81 vae.test2.com
192.168.8.81 vae.test3.com
```
ngx_http_add_listen 函数将解析http中的server的地址和端口，以port为粒度，添加addr（即1.1.1.1:80、2.2.2.2:80和80，会放在相同的cmcf->port[i]中）。
时序图和数据结构关系图：
![](/images/nginx-http-server-listen.png)

那么在cmcf中的存储例子
```
listen 192.168.8.81:80;
listen 127.0.0.1:80;
listen 80;
listen 80;
listen 8081;
```
![](/images/nginx-http-listen-2.png)

相关阅读：
[nginx 代码分析listen 和request请求的流程](https://blog.csdn.net/linux_vae/article/details/70911641"nginx 代码分析listen 和request请求的流程")
server_name [docs](http://nginx.org/en/docs/http/server_names.html "docs")
listen [docs](http://nginx.org/en/docs/http/ngx_http_core_module.html#listen "docs")