本文来自Android菜鸟：[https://blog.csdn.net/android_cai_niao/article/details/106027313 ](https://blog.csdn.net/android_cai_niao/article/details/106027313 )

# i++ 和 ++i原理
i++ 即后加加，原理是：先自增，然后返回自增之前的值<br/>
++i 即前加加，原理是：先自增，然后返回自增之后的值<br/>
重点：这是一般人所不知道的，记住：不论是前++还是后++，都有个共同点是先自增。
对于++i 就不说了，大多数人都懂，而对于 i++ 的原理，我用代码模拟其原理，如下：

```Java
int temp = i;
i = i + 1;
return temp;
```

# i++字节码分析
有很多的人写的文章上都是说i++是先返回i的值，然后再自增，这是错误，是先自增，然后再返回自增之前的值，你可能会问，这有区别吗？答案：有的。只要这个没理解对，则你在计算i++的相关问题时就有可能算错。<br/>
<br/>
有的人可能又会问了，我凭什么相信你，你有什么证据证明i++是先自增，然后再返回自增之前的值吗？我还真去找过证据，我们把class的字节码搞出来，分析一下就知道了，证明如下：

```Java
public class Test {
    void fun() {
        int i = 0;
        i = i++;
    }
}
```

如上，我们写了一个超级简单的Test类。在cmd中输入这个命令（javap -c Test.class）以查看其生成的字节码
![字节码](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220227153910.png)

# 表达式原则
表达式有一个原则：一个变量也是表达式，多个表达式的加减法运算都是从左到右进行的<br/>
<br/>
记住这个重点：一个变量也是表达式，多个表达式的加减法运算都是从左到右进行的

# 示例
示例1

```Java
int i = 0;
i = i++;
System.out.println("i = " + i);  // 结果：0
```

示例2

```Java
int a = 2;
int b = (3 * a++) + a;
System.out.println(b);   // 结果：9
```

示例3

```Java
int a = 2;
int b = a + (3 * a++);
System.out.println(b); // 结果：8
```

示例4

```Java
int i = 1;
int j = 1;
int k = i++ + ++i + ++j + j++;
System.out.println(k);  // 结果：8
```

示例5

```Java
int a = 0;
int b = 0;
a = a++;
b = a++;
System.out.println("a = " + a + ", b = " + b); // a = 1, b = 0
```

# 总结
+ i++ 即后加加，原理是：先自增，然后返回自增之前的值
+ ++i 即前加加，原理是：先自增，然后返回自增之后的值
+ 一个变量也是表达式，多个表达式的加减法运算都是从左到右进行的