原文地址：[使用枚举简单封装一个优雅的 Spring Boot 全局异常处理！](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247486379&idx=2&sn=48c29ae65b3ed874749f0803f0e4d90e&chksm=cea24460f9d5cd769ed53ad7e17c97a7963a89f5350e370be633db0ae8d783c3a3dbd58c70f8&scene=178&cur_album_id=1322577180722872320#rd)

# 使用 @ControllerAdvice 和 @ExceptionHandler 处理全局异常
这是目前很常用的一种方式，非常推荐。测试代码中用到了 Junit 5，如果你新建项目验证下面的代码的话，记得添加上相关依赖。
1. 新建异常信息实体类
   非必要的类，主要用于包装异常信息。

```Java
public class ErrorResponse {
    private String message;
    private String errorTypeName;

    public ErrorResponse(Exception e) {
        this(e.getClass().getName(), e.getMessage());
    }

    public ErrorResponse(String errorTypeName, String message) {
        this.errorTypeName = errorTypeName;
        this.message = message;
    }
    ......省略getter/setter方法
}
```

2. 自定义异常类型
   一般我们处理的都是 RuntimeException ，所以如果你需要自定义异常类型的话直接集成这个类就可以了。

```Java
public class ResourceNotFoundException extends RuntimeException {
    private String message;

    public ResourceNotFoundException() {
        super();
    }

    public ResourceNotFoundException(String message) {
        super(message);
        this.message = message;
    }

    @Override
    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }
}
```

3. 新建异常处理类
   我们只需要在类上加上@ControllerAdvice注解这个类就成为了全局异常处理类，当然你也可以通过 assignableTypes指定特定的 Controller 类，让异常处理类只处理特定类抛出的异常。

```Java
@ControllerAdvice(assignableTypes = {ExceptionController.class})
@ResponseBody
public class GlobalExceptionHandler {

    ErrorResponse illegalArgumentResponse = new ErrorResponse(new IllegalArgumentException("参数错误!"));
    ErrorResponse resourseNotFoundResponse = new ErrorResponse(new ResourceNotFoundException("Sorry, the resourse not found!"));

    @ExceptionHandler(value = Exception.class)// 拦截所有异常, 这里只是为了演示，一般情况下一个方法特定处理一种异常
    public ResponseEntity<ErrorResponse> exceptionHandler(Exception e) {

        if (e instanceof IllegalArgumentException) {
            return ResponseEntity.status(400).body(illegalArgumentResponse);
        } else if (e instanceof ResourceNotFoundException) {
            return ResponseEntity.status(404).body(resourseNotFoundResponse);
        }
        return null;
    }
}
```

4. controller模拟抛出异常

```Java
@RestController
@RequestMapping("/api")
public class ExceptionController {

    @GetMapping("/illegalArgumentException")
    public void throwException() {
        throw new IllegalArgumentException();
    }

    @GetMapping("/resourceNotFoundException")
    public void throwException2() {
        throw new ResourceNotFoundException();
    }
}
```

# @ExceptionHandler 处理 Controller 级别的异常
我们刚刚也说了使用@ControllerAdvice注解 可以通过 assignableTypes指定特定的类，让异常处理类只处理特定类抛出的异常。所以这种处理异常的方式，实际上现在使用的比较少了。

```Java

    @ExceptionHandler(value = Exception.class)// 拦截所有异常
    public ResponseEntity<ErrorResponse> exceptionHandler(Exception e) {

        if (e instanceof IllegalArgumentException) {
            return ResponseEntity.status(400).body(illegalArgumentResponse);
        } else if (e instanceof ResourceNotFoundException) {
            return ResponseEntity.status(404).body(resourseNotFoundResponse);
        }
        return null;
```

# ResponseStatus 和 ResponseStatusException
## ResponseStatus
研究 ResponseStatusException 我们先来看看，通过 ResponseStatus注解简单处理异常的方法（将异常映射为状态码）。

```Java
@ResponseStatus(code = HttpStatus.NOT_FOUND)
public class ResourseNotFoundException2 extends RuntimeException {

    public ResourseNotFoundException2() {
    }

    public ResourseNotFoundException2(String message) {
        super(message);
    }
}
```

```Java
@RestController
@RequestMapping("/api")
public class ResponseStatusExceptionController {
    @GetMapping("/resourceNotFoundException2")
    public void throwException3() {
        throw new ResourseNotFoundException2("Sorry, the resourse not found!");
    }
}
```

使用 Get 请求 localhost:8080/api/resourceNotFoundException2[2] ，服务端返回的 JSON 数据如下：

```Json
{
    "timestamp": "2019-08-21T07:11:43.744+0000",
    "status": 404,
    "error": "Not Found",
    "message": "Sorry, the resourse not found!",
    "path": "/api/resourceNotFoundException2"
}
```

## ResponseStatusException
这种通过 ResponseStatus注解简单处理异常的方法是的好处是比较简单，但是一般我们不会这样做，通过ResponseStatusException会更加方便,可以避免我们额外的异常类。

```Java
    @GetMapping("/resourceNotFoundException2")
    public void throwException3() {
        throw new ResponseStatusException(HttpStatus.NOT_FOUND, "Sorry, the resourse not found!", new ResourceNotFoundException());
    }
```

使用 Get 请求 localhost:8080/api/resourceNotFoundException2[3] ，服务端返回的 JSON 数据如下,和使用 ResponseStatus 实现的效果一样：

```Json
{
    "timestamp": "2019-08-21T07:28:12.017+0000",
    "status": 404,
    "error": "Not Found",
    "message": "Sorry, the resourse not found!",
    "path": "/api/resourceNotFoundException3"
}
```

ResponseStatusException 提供了三个构造方法：

```Java

    public ResponseStatusException(HttpStatus status) {
        this(status, null, null);
    }

    public ResponseStatusException(HttpStatus status, @Nullable String reason) {
        this(status, reason, null);
    }

    public ResponseStatusException(HttpStatus status, @Nullable String reason, @Nullable Throwable cause) {
        super(null, cause);
        Assert.notNull(status, "HttpStatus is required");
        this.status = status;
        this.reason = reason;
    }
```

构造函数中的参数解释如下：
+ status ：http status
+ reason ：response 的消息内容
+ cause ：抛出的异常

# 使用枚举简单封装一个优雅的 Spring Boot 全局异常处理
我们定义的异常被系统捕捉后返回给客户端的信息是这样的：
![枚举异常处理](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220223072614.png)

返回的信息包含了异常下面 5 部分内容：
1. 唯一标示异常的 code
2. HTTP 状态码
3. 错误路径
4. 发生错误的时间戳
5. 错误的具体信息

**ErrorCode.java (此枚举类中包含了异常的唯一标识、HTTP 状态码以及错误信息)**<br/>
这个类的主要作用就是统一管理系统中可能出现的异常，比较清晰明了。

```Java
import org.springframework.http.HttpStatus;

public enum ErrorCode {

    RESOURCE_NOT_FOUND(1001, HttpStatus.NOT_FOUND, "未找到该资源"),
    REQUEST_VALIDATION_FAILED(1002, HttpStatus.BAD_REQUEST, "请求数据格式验证失败");

    private final int code;

    private final HttpStatus status;

    private final String message;

    ErrorCode(int code, HttpStatus status, String message) {
        this.code = code;
        this.status = status;
        this.message = message;
    }

    public int getCode() {
        return code;
    }

    public HttpStatus getStatus() {
        return status;
    }

    public String getMessage() {
        return message;
    }

    @Override
    public String toString() {
        return"ErrorCode{" +
                "code=" + code +
                ", status=" + status +
                ", message='" + message + '\'' +
                '}';
    }
}
```

**ErrorReponse.java（返回给客户端具体的异常对象）**<br/>
这个类作为异常信息返回给客户端，里面包括了当出现异常时我们想要返回给客户端的所有信息。

```Java
@Data
public class ErrorResponse {
    private int code;
    private int status;
    private String message;
    private String path;
    private Instant timestamp;
    private Map<String, Object> data = new HashMap<String, Object>();

    public ErrorResponse() {
    }

    public ErrorResponse(BaseException ex, String path) {
        this(ex.getError().getCode(), ex.getError().getStatus().value(), ex.getError().getMessage(), path, ex.getData());
    }

    public ErrorResponse(int code, int status, String message, String path, Map<String, Object> data) {
        this.code = code;
        this.status = status;
        this.message = message;
        this.path = path;
        this.timestamp = Instant.now();
        if (!data.isEmpty()) {
            this.data.putAll(data);
        }
    }

    @Override
    public String toString() {
        return"ErrorResponse{" +
        "code=" + code +
        ", status=" + status +
        ", message='" + message + '\'' +
        ", path='" + path + '\'' +
        ", timestamp=" + timestamp +
        ", data=" + data +
        '}';
    }
}
```

**BaseException.java（继承自 RuntimeException 的抽象类，可以看做系统中其他异常类的父类）**<br/>
系统中的异常类都要继承自这个类。

```Java
public class BaseException extends RuntimeException {
    private final ErrorCode error;
    private final Map<String, Object> data = new HashMap<>();

    public BaseException(ErrorCode error, Map<String, Object> data) {
        super(error.getMessage());
        this.error = error;
        if (!data.isEmpty()) {
            this.data.putAll(data);
        }
    }

    protected BaseException(ErrorCode error, Map<String, Object> data, Throwable cause) {
        super(error.getMessage(), cause);
        this.error = error;
        if (!data.isEmpty()) {
            this.data.putAll(data);
        }
    }

    public ErrorCode getError() {
        return error;
    }

    public Map<String, Object> getData() {
        return data;
    }

}
```

**ResourceNotFoundException.java （自定义异常）**<br/>
可以看出通过继承 BaseException 类我们自定义异常会变的非常简单！

```Java
public class ResourceNotFoundException extends BaseException {

    public ResourceNotFoundException(Map<String, Object> data) {
        super(ErrorCode.RESOURCE_NOT_FOUND, data);
    }
}
```

**GlobalExceptionHandler.java（全局异常捕获）**<br/>
我们定义了两个异常捕获方法。<br/>
当我们抛出了 ResourceNotFoundException异常会被下面哪一个方法捕获呢？<br/>
答案：会被handleResourceNotFoundException()方法捕获。因为 @ExceptionHandler 捕获异常的过程中，会优先找到最匹配的。

```Java
@ControllerAdvice
@ResponseBody
public class GlobalExceptionHandler {

    @ExceptionHandler(value = BaseException.class)// 拦截所有异常（注意函数式参数的类型）
    public ResponseEntity<ErrorResponse> exceptionHandler(BaseException e, HttpServletRequest request) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(new ErrorResponse(e, request.getRequestURI()));
    }

    @ExceptionHandler(value = ResourceNotFoundException.class)// 只拦截ResourceNotFoundException异常，ResourceNotFoundException优先匹配（注意函数式参数的类型）
    public ResponseEntity<ErrorResponse> handleResourceNotFoundException(ResourceNotFoundException e, HttpServletRequest request) {
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(new ErrorResponse(e, request.getRequestURI()));
    }
}
```