# JWT简介
JWT全称为JSON Web Token。它是一种JSON格式的Web Token。<br/>
<br/>
JWT由3部分组成：头部(Header)、有效载荷(Payload)和签名(Signature)。<br/>
其中，签名需要Header和Payload进行加密后形成，类似于数字证书。Header和Payload也需要经过Base64编码后拼接。<br/>
具体公式如下：<br/>
JWTString=Base64(Header).Base64(Payload).HMACSHA256(base64UrlEncode(header)+"."+base64UrlEncode(payload),secret)<br/>
<br/>
其中：
+ header和payload可以直接利用base64解码出原文，从header中获取哈希签名的算法，从payload中获取有效数据
+ signature由于使用了不可逆的加密算法，无法解码出原文，它的作用是校验token有没有被篡改。服务端获取header中的加密算法之后，利用该算法加上secretKey对header、payload进行加密，比对加密后的数据和客户端发送过来的是否一致。因此secretKey只能保存在服务端。

<br/>
那么为什么要使用JWT呢？cookie和session它不香吗？<br/>
相较于session而言
+ 如果用户信息都保存在session服务端，那么随着用户数量的增多，不可避免的会导致服务端开销过大，内存不够等情况发生。
+ 分布式系统中，session会需要同步，增加了复杂性。
+ 可以将JWT放在请求头中，以此代替cookie（个人感觉实现起来挺难）
+ 不要每次验证时都去Redis或数据库中查找凭证。

JWT也有缺点，因为其不依赖于服务端，所以服务端无法实时对其销毁，或是改变有效时间等等。<br/>
<br/>
JWT的使用：<br/>
<br/>
依赖:

```pom
<dependency>
    <groupId>com.auth0</groupId>
    <artifactId>java-jwt</artifactId>
    <version>3.10.3</version>
</dependency>
```

<br/>
生成与解析JWT的token（对称加密）

```Java
public class JWTUtils {
    // 签名密钥
    private static final String SECRET = "!DAR$";

    /**
     * 生成token
     * @param payload token携带的信息
     * @return token字符串
     */
    public static String getToken(Map<String,String> payload){
        // 指定token过期时间为7天
        Calendar calendar = Calendar.getInstance();
        calendar.add(Calendar.DATE, 7);

        JWTCreator.Builder builder = JWT.create();
        // 构建payload
        payload.forEach((k,v) -> builder.withClaim(k,v));
        // 指定过期时间和签名算法
        String token = builder.withExpiresAt(calendar.getTime()).sign(Algorithm.HMAC256(SECRET));
        return token;
    }


    /**
     * 解析token
     * @param token token字符串
     * @return 解析后的token
     */
    public static DecodedJWT decode(String token){
        JWTVerifier jwtVerifier = JWT.require(Algorithm.HMAC256(SECRET)).build();
        DecodedJWT decodedJWT = jwtVerifier.verify(token);
        return decodedJWT;
    }
}
```

注意因为token过期或者不正确无法验证通过时，都会报异常，因此在解析token时应该捕获异常进行处理。<br/>
<br/>
生成与解析JWT的token（非对称加密）

```Java
private static final String RSA_PRIVATE_KEY = "...";
private static final String RSA_PUBLIC_KEY = "...";

/**
     * 生成token
     * @param payload token携带的信息
     * @return token字符串
     */
public static String getTokenRsa(Map<String,String> payload){
    // 指定token过期时间为7天
    Calendar calendar = Calendar.getInstance();
    calendar.add(Calendar.DATE, 7);

    JWTCreator.Builder builder = JWT.create();
    // 构建payload
    payload.forEach((k,v) -> builder.withClaim(k,v));

    // 利用hutool创建RSA
    RSA rsa = new RSA(RSA_PRIVATE_KEY, null);
    // 获取私钥
    RSAPrivateKey privateKey = (RSAPrivateKey) rsa.getPrivateKey();
    // 签名时传入私钥
    String token = builder.withExpiresAt(calendar.getTime()).sign(Algorithm.RSA256(null, privateKey));
    return token;
}

/**
     * 解析token
     * @param token token字符串
     * @return 解析后的token
     */
public static DecodedJWT decodeRsa(String token){
    // 利用hutool创建RSA
    RSA rsa = new RSA(null, RSA_PUBLIC_KEY);
    // 获取RSA公钥
    RSAPublicKey publicKey = (RSAPublicKey) rsa.getPublicKey();
    // 验签时传入公钥
    JWTVerifier jwtVerifier = JWT.require(Algorithm.RSA256(publicKey, null)).build();
    DecodedJWT decodedJWT = jwtVerifier.verify(token);
    return decodedJWT;
}
```

# JWT实现单点登录

![单点登录](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/JWT单点登录.png)



# JWT在博客中的使用
之前在博客中，对于登录成功的用户，我直接将其个人信息存储到session中进行保存。现在用JWT+cookie的方式进行代替。<br/>
<br/>
properties配置

```properties
#JWT对称加密
jwt.secret=ENC(7GLL1otKl6E+9CXA+kQY0WdjR3wOvE0Z)
```

<br/>
JWTUtil

```Java
@Component
public class JWTUtil {
    private Logger logger = LoggerFactory.getLogger(JWTUtil.class);

    @Value("${jwt.secret}")
    private String JWTSCRET;

    public String createToken(String userJson){
        // 指定token过期时间
        Calendar calendar = Calendar.getInstance();
        calendar.add(Calendar.HOUR, 6);
        Map<String, Object> header = new HashMap<>();
        header.put("type", "JWT");
        String token = JWT.create()
                .withHeader(header)
                .withClaim("user", userJson)
                .withExpiresAt(calendar.getTime())
                .sign(Algorithm.HMAC256(JWTSCRET));
        return token;
    }

    public Boolean checkToken(String token) {
        // 创建解析对象，使用的算法和secret要与创建token时保持一致
        JWTVerifier jwtVerifier = JWT.require(Algorithm.HMAC256(JWTSCRET)).build();
        // 解析指定的token
        DecodedJWT decodedToken = null;
        try {
            decodedToken = jwtVerifier.verify(token);
        } catch (JWTVerificationException e) {
            e.printStackTrace();
            return false;
        }
//        // 获取解析后的token中的payload信息
//        Claim userName = decodedToken.getClaim("userName");
//        System.out.println(userName.asString());
//        // 输出超时时间
//        System.out.println(decodedToken.getExpiresAt());
//        System.out.println(decodedToken.getToken());
//        System.out.println(decodedToken.getHeader());
//        System.out.println(decodedToken.getPayload());
//        System.out.println(decodedToken.getSignature());
        return true;
    }

    public User getUser(String token) {
        // 创建解析对象，使用的算法和secret要与创建token时保持一致
        JWTVerifier jwtVerifier = JWT.require(Algorithm.HMAC256(JWTSCRET)).build();
        // 解析指定的token
        DecodedJWT decodedToken = null;
        try {
            decodedToken = jwtVerifier.verify(token);
        } catch (JWTVerificationException e) {
            e.printStackTrace();
            return null;
        }
        token = decodedToken.getClaim("user").asString();
        return JSON.parseObject(token, User.class);
    }
}
```

可以看到，我的JWT接受到的参数是由User对象转换成的JSON格式的字符串。<br/>
所以在验证JWT通过后，获取到其中保存的payload后，就可以再将String转换成User对象，从而获取用户信息。<br/>
<br/>
controller部分：

```Java
//跳转登录页面
    @RequestMapping(method = RequestMethod.GET)
    public String loginPage() {
        return "admin/login";
    }
	
    @RequestMapping(path = "/login", method = RequestMethod.POST)
    public String login(@RequestParam String username, @RequestParam String password, @RequestParam("srcPath") String srcPath, HttpServletResponse response, RedirectAttributes attributes) {
        password = BlogUtil.md5(password);
        User user = userService.findUserByUsernameAndPassword(username, password);
        if (user != null) {
            String token = jwtUtil.createToken(JSON.toJSONString(user));
            Cookie cookie = new Cookie("Authorization", token);
            cookie.setMaxAge(60 * 60 * 6);
            response.addCookie(cookie);
            return "redirect:" + ((srcPath == null || srcPath == "" || srcPath == "/admin") ? "/admin/index" : srcPath);
        }else {
            attributes.addFlashAttribute("message", "用户名和密码错误");
            return "redirect:/admin";
        }
    }
```

可以看到当登录验证成功后，就创造JWT，并将其存入到cookie中。之后重定向到之前保存的srcPath，如果srcPath为空则重定向到首页。<br/>
<br/>
interceptor部分：

```Java
@Component
public class LoginRequiredInterceptor implements HandlerInterceptor {
    private final Logger logger = LoggerFactory.getLogger(this.getClass());

    @Autowired
    private JWTUtil jwtUtil;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        if (handler instanceof HandlerMethod) {
            HandlerMethod handlerMethod = (HandlerMethod) handler;
            Method method = handlerMethod.getMethod();
            LoginRequired loginRequired =  method.getAnnotation(LoginRequired.class);
            if (loginRequired != null) {
                Cookie[] cookies = request.getCookies();
                if (cookies != null) {
                    for (Cookie cookie : cookies) {
                        if (cookie.getName().equals("Authorization")) {
                            User user = null;
                            if (jwtUtil.checkToken(cookie.getValue())
                                    && (user = jwtUtil.getUser(cookie.getValue())) != null
                                    && user.getType() == 1) {
                                request.setAttribute("user", user);
                                return true;
                            }else {
                                break;
                            }
                        }
                    }
                }
                request.setAttribute("srcPath", request.getRequestURI());
                request.getRequestDispatcher("/admin").forward(request, response);
                return false;
            }
        }
        return true;
    }
}
```

对于需要登录验证的访问请求，都会使用拦截器拦截。之后查看cookie，如果为空或者没有指定cookie或者验证错误，则设置srcPath，并重定向到登录页面。如果成功则返回true。<br/>

参考:
+ https://blog.csdn.net/weixin_45070175/article/details/118559272