- [Java与Golang中的内存逃逸](#java与golang中的内存逃逸)
  - [Java](#java)
    - [JIT](#jit)
    - [HotSpot虚拟机的两个不同的即时编译器](#hotspot虚拟机的两个不同的即时编译器)
    - [JVM三种模式](#jvm三种模式)
    - [热点检测](#热点检测)
    - [内存逃逸](#内存逃逸)
      - [逃逸分析](#逃逸分析)
      - [具体优化](#具体优化)
  - [Golang](#golang)
    - [内存逃逸](#内存逃逸-1)
    - [逃逸分析](#逃逸分析-1)
    - [具体优化](#具体优化-1)

# Java与Golang中的内存逃逸
## Java
### JIT 
java程序变成计算机可执行的机器指令拆分为两个步骤：首先是把.java文件转换成.class文件。然后是把.class转化成机器指令的过程。第一段编译就是javac命令。
在第二编译阶段，JVM 通过解释字节码将其翻译成对应的机器指令，逐条读入，逐条解释翻译。这就是传统JVM解释器的功能，速度非常慢，为了解决这种效率问题，在JDK1.6中引入了JIT技术。
Java程序最初是通过解释器进行解释执行的，当虚拟机发现某个方法或代码块运行的特别频繁时，会把这些代码认定为“热点代码”（Hot Spot Code）。为了提高热点代码的执行效率，在运行时，虚拟机会把这些代码编译成本地平台相关的机器码，并进行各种层次的优化，完成这个任务的编译器称为即时编译器。

### HotSpot虚拟机的两个不同的即时编译器
从业务的角度而言，服务端和用户端对代码的执行速度和启动速度的要求是不一样的。

有两种编译模式可以选择，并且其会在运行时决定使用哪一种以达到最优性能。这两种编译模式的命名源自于命令行参数（eg: -client 或者 -server）。JVM Server 模式与 client 模式启动，最主要的差别在于：-server 模式启动时，速度较慢，但是一旦运行起来后，性能将会有很大的提升。原因是：当虚拟机运行在-client 模式的时候，使用的是一个代号为 C1 的轻量级编译器，而-server 模式启动的虚拟机采用相对重量级代号为 C2 的编译器。C2 比 C1 编译器编译的相对彻底，服务起来之后，性能更高。

目前主流的HotSpot虚拟机中默认是采用解释器与其中一个编译器直接配合的方式工作。

### JVM三种模式
int模式： 用 -Xint 开启，即 解释模式， 在这种模式下全部采取解释模式运行。
comp模式： 用 -Xcomp开启，这种模式下通知JVM关闭 解释模式 ， 采用 编译模式 运行。但往往导致无法得到良好的自动优化。
mixed模式： 用 -Xmixed 开启，即混合运行模式，也是Hotspot的默认模式。

### 热点检测
要想触发JIT，首先需要识别出热点代码。目前主要的热点代码识别方式是热点探测（Hot Spot Detection），有以下两种：
+ 基于采样的方式探测（Sample Based Hot Spot Detection) ：周期性检测各个线程的栈顶，发现某个方法经常出险在栈顶，就认为是热点方法。好处就是简单，缺点就是无法精确确认一个方法的热度。容易受线程阻塞或别的原因干扰热点探测。
+ 基于计数器的热点探测（Counter Based Hot Spot Detection)。采用这种方法的虚拟机会为每个方法，甚至是代码块建立计数器，统计方法的执行次数，某个方法超过阀值就认为是热点方法，触发JIT编译。

 在HotSpot虚拟机中使用的是第二种——基于计数器的热点探测方法，因此它为每个方法准备了两个计数器：方法调用计数器和回边计数器。
 + 方法调用计数器：记录一个方法被调用次数的计数器。
 + 回边计数器：记录方法中的for或者while的运行次数的计数器。

### 内存逃逸
#### 逃逸分析
通过逃逸分析，Java Hotspot编译器能够分析出一个新的对象的引用的使用范围，从而决定是否要将这个对象分配到堆上。

JVM判断新创建的对象是否逃逸的依据有：
+ 对象被赋值给堆中对象的字段和类的静态变量。
+ 对象被传进了不确定的代码中去运行。

对于第一种情况，因为对象被放进堆中，则其它线程就可以对其进行访问，所以对象的使用情况，编译器就无法再进行追踪。第二种情况相当于JVM在解析普通的字节码的时候，如果没有发生JIT即时编译，编译器是不能事先完整知道这段代码会对对象做什么操作。保守一点，这个时候也只能把对象是当作是逃逸来处理。

```Java
public class EscapeTest {

    public static Object globalVariableObject;

    public Object instanceObject;

    public void globalVariableEscape(){
        globalVariableObject = new Object(); //静态变量,外部线程可见,发生逃逸
    }

    public void instanceObjectEscape(){
        instanceObject = new Object(); //赋值给堆中实例字段,外部线程可见,发生逃逸
    }
    
    public Object returnObjectEscape(){
        return new Object();  //返回实例,外部线程可见，发生逃逸
    }

    public void noEscape(){
        synchronized (new Object()){
            //仅创建线程可见,对象无逃逸
        }
        Object noEscape = new Object();  //仅创建线程可见,对象无逃逸
    }

}
```

#### 具体优化
1.同步省略：
如果同步块所使用的锁对象通过这种分析被证实只能够被一个线程访问，那么JIT编译器在编译这个同步块的时候就会取消对这部分代码的同步。这个取消同步的过程就叫同步省略，也叫锁消除。

```Java
public void f() {
    Object hollis = new Object();
    synchronized(hollis) {
        System.out.println(hollis);
    }
}
```

代码中对hollis这个对象进行加锁，但是hollis对象的生命周期只在f()方法中，并不会被其他线程所访问到，所以在JIT编译阶段就会被优化掉

```Java
public void f() {
    Object hollis = new Object();
    System.out.println(hollis);
}
```

所以，在使用synchronized的时候，如果JIT经过逃逸分析之后发现并无线程安全问题的话，就会做锁消除。

2.标量替换:
在JIT阶段，如果经过逃逸分析，发现一个对象不会被外界访问的话，那么经过JIT优化，就会把这个对象拆解成若干个其中包含的若干个成员变量来代替。这个过程就是标量替换。

```Java
public static void main(String[] args) {
   alloc();
}

private static void alloc() {
   Point point = new Point（1,2）;
   System.out.println("point.x="+point.x+"; point.y="+point.y);
}
class Point{
    private int x;
    private int y;
}
```

以上代码中，point对象并没有逃逸出alloc方法，并且point对象是可以拆解成标量的。那么，JIT就会不会直接创建Point对象，而是直接使用两个标量int x ，int y来替代Point对象。以上代码，经过标量替换后，就会变成：

```Java
private static void alloc() {
   int x = 1;
   int y = 2;
   System.out.println("point.x="+x+"; point.y="+y);
}
```

这样做的好处就是可以大大减少堆内存的占用。

3.栈上分配:
在Java虚拟机中，对象是在Java堆中分配内存的，这是一个普遍的常识。但是，有一种特殊情况，那就是如果经过逃逸分析后发现，一个对象并没有逃逸出方法的话，那么就可能被优化成栈上分配。这样就无需在堆上分配内存，也无须进行垃圾回收了。

## Golang
### 内存逃逸
在一段程序中，每一个函数都会有自己的内存区域分配自己的局部变量，返回值，这些内存会由编译器在栈中进行分配，每一个函数会分配一个栈帧，在函数运行结束后销毁，但是有些变量我们想在函数运行结束后仍然使用，就需要把这个变量分配在堆上，这种从“栈”上逃逸到“堆”上的现象叫做内存逃逸

### 逃逸分析
虽然Go语言引入的Gc，GC机制会对堆上的对象进行管理，当某个对象不可达(没有其他对象引用他)，他将会被回收。虽然GC可以降低工作人员负担，但是GC也会给程序带来性能损耗，当堆内存上有大量的堆内存对象，就会给GC很大的压力，虽然Go语言使用的是标记清除算法，并且在此基础上使用了三色标记法和写屏障技术，但是我们在堆上分配大量内存，仍然会对GC造成很大压力，Go引入了逃逸分析，就是想减少堆内存的分配，可以在栈分配的内存尽量分配在栈上。
逃逸分析就是在程序编译阶段根据代码中的数据流，对代码中哪些变量需要在栈上分配，哪些需要在对象分配的静态分析方法，堆和栈相比，堆适合分配不可预知大小的内存，但是付出代价是分配速度慢，容易产生碎片，栈分配十分快，栈分配只需要两个指令“Push”和"Release"分配和释放，而且堆分配需要先找一块适合大小的内存块分配，需要垃圾回收释放，所以逃逸分析可以更好的做内存分配

src/cmd/compile/internal/gc/escape.go
+ pointers to stack objects cannot be stored in the heap: 指向栈对象的指针不能存储在堆中
+ pointers to a stack object cannot outlive that object:指向栈对象的指针不能超过该对象的存活期，指针不能在栈对象销毁之后依然存活（例子：声明的函数返回并销毁了对象的栈帧，或者它在循环迭代中被重复用于逻辑上不同的变量）

既然逃逸分析是在编译阶段进行的，那我们就可以通过go build -gcflga '-m -m l'查看逃逸分析结果


### 具体优化
1.函数返回局部指针变量

```go
func Add(x,y int) *int {
 res := 0
 res = x + y
 return &res
}
func main()  {
 Add(1,2)
}
```

函数返回局部变量是一个指针变量，函数Add执行结束，对应栈帧就会销毁，但是引用返回到函数外部，如果我们外部解析地址，就会导致程序访问非法内存，所以经过编辑器分析过后将其在堆上分配

2.interface类型逃逸
```go
func main() {
	a := 1
	fmt.Println("a逃逸，a:", a)
}
```

因为fmt.Println函数参数类型是interface{}, 在 interface 类型上调用方法都是动态调度的 —— 方法的真正实现只能在运行时知道。

3.闭包产生逃逸
```go
func f() func() int{
	a := 1
	return func() int {
		return a
	}
}

func main() {
	f()
}
```

4.对象太大, 超过栈帧大小
```go
func main() {
	_ = make([]int, 0, 1000)
	_ = make([]int, 0, 10000)
}
```

参考：
https://blog.csdn.net/u011972171/article/details/80905564
https://blog.csdn.net/qq_47102228/article/details/121632112
https://blog.csdn.net/shuux666/article/details/123953234
https://www.cnblogs.com/dawnlight/p/15552517.html
https://blog.csdn.net/weixin_45802793/article/details/125633869
https://blog.csdn.net/weixin_45626550/article/details/123740505