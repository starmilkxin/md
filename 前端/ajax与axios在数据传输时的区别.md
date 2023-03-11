最近在学Vue，为了实现博客的前后端分离。虽然学的很累，但是学到了不少的东西。<br/>
# 数据的传输方式与格式
数据的传输方式
+ get请求，使用 params 的方式，直接拼接在url后，Request URL: http://localhost:81/index?current=2&limit=2
+ post请求，使用data的方式，数据放在请求体body中传输(也可以用params的方式)。

数据的传输格式
+ application/json 传给后端的数据就形如  { ‘name’:’edward’, ‘age’:’25’ }
+ application/x-www-form-urlencoded 传输给后端的数据就形如   ‘name=edward&age=25’

在body中的数据格式又有两种，一种是json数据格式，另一种是字符串。<br/>
前端可以设置好请求头中的数据格式，不同格式的数据，后端接受的方法也不一样。

# axios与ajax发送数据的区别
get请求都是比较明确的了，就算通过params的方式，直接拼接在url后，后端可以通过@RequestParam或者request.getParameter()来获取参数。<br/>
<br/>
post操作则就有些东西了。<br/>
ajax的话，我现在的前后端不分离博客中就算用ajax来实现翻页中的数据传输与接受操作。ajax.post操作，数据格式为，Content-Type: application/x-www-form-urlencoded; charset=UTF-8<br/>
因为$.ajax() post发送到服务器的数据。将自动转换为请求字符串格式。<br/>
如果为对象，如 { name: 'edward', age: '25' }，jQuery 将自动转化为 'name=edward&age=25'<br/>
如果为数组，jQuery 将自动为不同值对应同一个名称。如 {foo:["bar1", "bar2"]} 转换为 '&foo=bar1&foo=bar2'。<br/>
而application/x-www-form-urlencoded格式的数据，就是通过@RequestParam或者request.getParameter()来获取值，和get请求一样，区别就算url中不会有拼接。<br/>
<br/>
axios的请求方法：<br/>

```JS
//get 请求
axios({
    method: 'GET',
    url: 'xxxxx',
    params: param,
  })
//或者
axios({
    method: 'GET',
    url: '/xxx?message=' + msg,
  })

//post 请求
axios({
    method: 'POST',
    url: '/xxxxx',
    data: param,
  })
//或者
 axios({
    method: 'POST',
    url: '/xxxxx',
    params: param,
  })
```

除了第三种方法外，其余三种都是通过param拼接在url后，然后后端通过@RequestParam或者request.getParameter()来获取值。<br/>
而第三种方法，axios是默认将数据转换为json格式的，同时请求头中的Content-Type:也是application/json。此时后端就得用@RequestBody注解，实体类(也可以是Map或者String)接受。<br/>
但你也可以将其强行转换为ajax.post传输的数据格式，首先当然的了我们要设置headers: { 'content-type': 'application/x-www-form-urlencoded' }，然后
+ 通过qs插件中的，qs.stringfy() 将对象序列化成URL的形式
+ 使用URLSearchParams来处理参数来处理参数

![URLSearchParams](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220409145128.png)

参考：
+ https://blog.csdn.net/qq_41499782/article/details/118916901
+ https://www.cnblogs.com/edwardwzw/p/11694903.html?ivk_sa=1024320u