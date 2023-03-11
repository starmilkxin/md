平常的博客与项目中免不了需要插入大量图片，关于图片的存储就成了难题，市面上付费的与免费的图床有许多，不过我选择使用PicGo+Github+jsdelivr搭建高效的个人免费图床
# 一、PicGo
[PicGo的Github链接](https://github.com/Molunerfinn/PicGo)<br/>
PicGo: 一个用于快速上传图片并获取图片 URL 链接的工具

支持多种图床：
+ 七牛图床 v1.0
+ 腾讯云 COS v4\v5 版本 v1.1 & v1.5.0
+ 又拍云 v1.2.0
+ GitHub v1.5.0
+ SM.MS V2 v2.3.0-beta.0
+ 阿里云 OSS v1.6.0
+ Imgur v1.6.0

官网直接下载安装，它的[指导手册](https://picgo.github.io/PicGo-Doc/zh/guide/getting-started.html)已经把如何配置各类图床说的非常详细了。这次我们要搭建的就是Github图床。

# 二、Github图床
![1](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/PicGo_Github设置.PNG)

首先在PicGo中选定GitHub图床，根据指导手册我们来一一设置仓库名、分支名(一般默认就是main)、设定Token、指定存储路径、自定义域名。

这里需要注意的就是Token和自定义域名这两点。
+ Token： 生成一个token用于PicGo操作你的仓库，访问：https://github.com/settings/tokens 然后点击Generate new token。把repo的勾打上即可。然后翻到页面最底部，点击Generate token的绿色按钮生成token。<br/>
  注意1：这个token生成后只会显示一次！你要把这个token复制一下存到其他地方以备以后要用。<br/>
  注意2：仓库请尽量创建公共仓库，私人仓库好像会导致一些图片失效的情况出现。<br/>
+ 自定义域名：因为Github图床在未使用梯子的情况下，图片大概率会加载不出来，所以需要用到CDN加速。

# 三、CDN
CDN的全称是Content Delivery Network，即内容分发网络。CDN是构建在现有网络基础之上的智能虚拟网络，依靠部署在各地的边缘服务器，通过中心平台的负载均衡、内容分发、调度等功能模块，使用户就近获取所需内容，降低网络拥塞，提高用户访问响应速度和命中率。CDN的关键技术主要有内容存储和分发技术。

**jsDelivr 为开发者提供免费公共 CDN 加速服务**
jsDelivr提供Github的镜像，使用起来非常的便捷，只需要在Github仓库前添加https://cdn.jsdelivr.net/gh/ 即可，因此我们的**自定义域名**就设置为  https://cdn.jsdelivr.net/gh/ + 仓库名
- - -
最后我们就可以在**上传区**上传我们的图片到Github图床，并且得到对应的URL。
![上传区](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20211121104706.png)

我们还可以在**相册**管理我们的图片（我当时未使用CDN加速时图片都是看不见的，因为没梯子，CDN加速后就都可以看见了）。
![相册](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20211121104840.png)

