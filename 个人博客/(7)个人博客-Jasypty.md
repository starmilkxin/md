jasypt是国外开发者(@author Daniel Fernández)写的一个对PropertySource资源进行加密保护的依赖工具。我们可以使用其来对一些敏感信息(如：配置文件中的各种账号密码)进行加密保护。<br/>
<br/>
在Spring Boot中使用jasypt非常的便捷。<br/>
<br/>
首先导入依赖：

```pom
<dependency>
	<groupId>com.github.ulisesbocchio</groupId>
	<artifactId>jasypt-spring-boot-starter</artifactId>
	<version>3.0.1</version>
</dependency>
```

<br/>
指定加密算法<br/>
jasypt.encryptor.algorithm=PBEWithMD5AndDES<br/>
<br/>
指定密钥<br/>
jasypt.encryptor.password=123<br/>
<br/>
得到加密后的字符串

```Java
@SpringBootTest
class CommunityForumApplicationTests {
    @Resource
    private StringEncryptor stringEncryptor;

    @Test
    void contextLoads() {
        String secret = stringEncryptor.encrypt("root");
        System.out.println(secret);
        String inital = stringEncryptor.decrypt(secret);
        System.out.println(inital);
    }
}
```

<br/>
得到加密后的字符串后，将其原密码改为ENC(加密后字符串)<br/>
此时程序已经可以运行了<br/>
<br/>
但是密钥不能泄露，所以删除密钥之后，需要通过参数的方法传给程序<br/>
在IDEA中则是

![IDEA](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220405083317.png)

<br/>
bash则是:
java -Djasypt.encryptor.password="password" -jar my-application.jar<br/>
或不挂起后台运行<br/>
nohup java -Djasypt.encryptor.password="password" -jar my-application.jar >> output.log 2>&1 &

<br/>
<br/>
当然最简单还是使用配置中心来进行配置信息了。