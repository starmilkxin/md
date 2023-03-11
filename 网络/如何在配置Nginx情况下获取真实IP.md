# 未配置Ngnix的情况下
在没有配置Nginx的情况下，获取用户的IP非常的简单，通过request.getRemoteAddr();即可。<br/>
如在日志切片，我们需要获取用户的真实IP。

```Java
public void doBefore(JoinPoint joinPoint) {
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attributes.getRequest();
        String url = request.getRequestURL().toString();
        String ip = request.getRemoteAddr();
        String classMethod = joinPoint.getSignature().getDeclaringTypeName()+ "." + joinPoint.getSignature().getName();
        Object[] args = joinPoint.getArgs();
        RequestLog requestLog = new RequestLog(url, ip, classMethod, args);
        logger.info( "Request: {}", requestLog);
    }
```

# 配置了Ngnix的情况下
在配置了Nginx的情况下，便不能用request.getRemoteAddr();来获取用户的真实IP了，因为用户的请求经过了Nginx的转发后，我们服务端获取的remote_addr是转发给我们服务器的最近的那个Nginx的IP地址。<br/>
此时我们可以利用X-Forwarded-For请求头来存储这期间所有被转发的IP地址，在Nginx中如下配置

```conf
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

此时请求头中的X-Forwarded-For数据便是这样的

```
X-Forwarded-For: client_ip, proxy1_ip, proxy2_ip
```

当client_ip遇到第一个Ngnix时，请求头中便会存放X-Forwarded-For并且添加client_ip（用户IP），此后遇到第二个Ngnix时又会添加proxy1_ip（第一个Ngnix的IP），在最后转发给服务器时会添加proxy2_ip（第二个Ngnix的IP）。<br/>
看起来好像我们只要取第一个IP就是用户的真实IP了是吧，非也。<br/>

## X-Forwarded-For的伪造
既然我们可以设置请求头，那自然其也可以被伪造。
![伪造请求头](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20211231212858.png)
如果你服务端的判断IP的逻辑为选取X-Forwarded-For最左侧第一个IP的话，那么如上图，利用Postman就可以轻松伪造X-Forwarded-For来实现刷票防止被ban掉IP等行为。<br/>
解决的方法也很简单
+ 方法一：在第一层Nginx中配置proxy_set_header X-Forwarded-For $remote_addr;因为$remote_addr是获取的是直接TCP连接的客户端IP（类似于Java中的request.getRemoteAddr()），这个是无法伪造的，所以可以获取到与第一层Nginx连接的IP。
+ 方法二：遍历X-Forwarded-For请求头的IP地址，获取第一层Nginx的左侧的IP地址，其实原理和方法一一样。

缺陷: 如果用户是通过正向代理的方式来进行访问的话，那么如果将代理的IP地址查封后，就会使得所有通过该代理IP进行访问的用户无法进行访问。而对于在服务器进行过注册的代理服务器而言，我们可以取该代理服务器左侧的IP作为用户的真实IP，原理与方法一和方法二相同。
<br/>
<br/>
