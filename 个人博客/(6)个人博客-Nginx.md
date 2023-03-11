关于http升级成https，我太懒了。。。其实早就可以做的。。。

# 动态分离
为了能够分摊服务器的访问流量，提升服务访问性能，将博客静态资源分开存放。并且通过Nginx拦截静态资源uri转而访问服务器事先存放好的静态资源。

```conf
                #拦截静态资源
                location ~ .*\.(html|htm|gif|jpg|jpeg|bmp|png|ico|js|css)$ {
                        root /usr/local/myblog/static/;
                        expires 14d;
                }

```

# 支持https访问
为了实现支持https访问,Nginx需要配置如下

```conf
        upstream myserver {
                server ***** max_fails=3 fail_timeout=30s;
        }

        proxy_set_header X-Forwarded-For $remote_addr;

        server {
                listen 443;
                server_name starmilk.top;
                ssl on;
                ssl_certificate "******";
                ssl_certificate_key "******";
                ssl_session_cache shared:SSL:1m;
                ssl_session_timeout  10m;
                ssl_ciphers PROFILE=SYSTEM;
                ssl_prefer_server_ciphers on;

                location / {
                        proxy_pass http://myserver;
                }

                #拦截静态资源
                location ~ .*\.(html|htm|gif|jpg|jpeg|bmp|png|ico|js|css)$ {
                        root /usr/local/myblog/static/;
                        expires 14d;
                }
        }

        server {
                listen 80;
                server_name starmilk.top;
                return 301 https://starmilk.top$request_uri;
        }

```

除了要监听443端口实现https访问，还要监听80端口，将http访问重定向到https访问。

# 反向代理时如何获取ip
在Nginx配置中可以看到，我将所有符合条件的url都转发到了真正的项目地址中，这样反向代理可以隐藏真实的服务器ip地址，并且可以实现负载均衡。

```conf
                location / {
                        proxy_pass http://myserver;
                }
```

<br/>
但是因为使用了Nginx所以我们无法直接在后端通过request.getRemoteAddr()获取客户端的ip。<br/>
具体可以看这篇博客

[如何在配置Nginx情况下获取真实IP](https://starmilk.top/blog/52)

因此我们在Nginx中配置了proxy_set_header X-Forwarded-For $remote_addr;来获取访问Nginx时的客户端ip，并将其存放在请求头中。<br/>
后端获取客户端ip时，就直接获取请求头中的参数即可，如下

```Java
String ip = request.getHeader("X-Forwarded-For");
```

<br/>
最终日志切片的实现效果和以前一样

![地址](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220319083300.png)