## volatile

---

`volatile`修饰符的作用有两个：

+ 禁止指令重排序（也就是说对`volatile`修饰的变量修改都是对之后的代码可见的）
+ 可见性(对`volatile`修饰的变量修改对别的线程立即可见)

`volatile`的实现：

​	用两类来测试`User`和`People`（不要在意类名，瞎鸡儿取的）

`User`类的`cnt`用`volatile`修饰

```Java
package demo;

public class User {
    private volatile int cnt = 0;

    public int getCnt() {
        return cnt;
    }

    public void setCnt(int cnt) {
        this.cnt = cnt;
    }
}

```

`People`类的`cnt`没有用`volatile`修饰

```java
package demo;

public class People {
    private int cnt = 0;

    public int getCnt() {
        return cnt;
    }

    public void setCnt(int cnt) {
        this.cnt = cnt;
    }
}

```

`User`类的`.class`文件

```clike
// class version 52.0 (52)
// access flags 0x21
public class demo/User {

  // compiled from: User.java

  // access flags 0x42
  private volatile I cnt

  // access flags 0x1
  public <init>()V
   L0
    LINENUMBER 3 L0
    ALOAD 0
    INVOKESPECIAL java/lang/Object.<init> ()V
   L1
    LINENUMBER 4 L1
    ALOAD 0
    ICONST_0
    PUTFIELD demo/User.cnt : I
    RETURN
   L2
    LOCALVARIABLE this Ldemo/User; L0 L2 0
    MAXSTACK = 2
    MAXLOCALS = 1

  // access flags 0x1
  public getCnt()I
   L0
    LINENUMBER 7 L0
    ALOAD 0
    GETFIELD demo/User.cnt : I
    IRETURN
   L1
    LOCALVARIABLE this Ldemo/User; L0 L1 0
    MAXSTACK = 1
    MAXLOCALS = 1

  // access flags 0x1
  public setCnt(I)V
   L0
    LINENUMBER 11 L0
    ALOAD 0
    ILOAD 1
    PUTFIELD demo/User.cnt : I
   L1
    LINENUMBER 12 L1
    RETURN
   L2
    LOCALVARIABLE this Ldemo/User; L0 L2 0
    LOCALVARIABLE cnt I L0 L2 1
    MAXSTACK = 2
    MAXLOCALS = 2
}

```

`People`的`.class`

```clike
// class version 52.0 (52)
// access flags 0x21
public class demo/People {

  // compiled from: People.java

  // access flags 0x2
  private I cnt

  // access flags 0x1
  public <init>()V
   L0
    LINENUMBER 3 L0
    ALOAD 0
    INVOKESPECIAL java/lang/Object.<init> ()V
   L1
    LINENUMBER 4 L1
    ALOAD 0
    ICONST_0
    PUTFIELD demo/People.cnt : I
    RETURN
   L2
    LOCALVARIABLE this Ldemo/People; L0 L2 0
    MAXSTACK = 2
    MAXLOCALS = 1

  // access flags 0x1
  public getCnt()I
   L0
    LINENUMBER 7 L0
    ALOAD 0
    GETFIELD demo/People.cnt : I
    IRETURN
   L1
    LOCALVARIABLE this Ldemo/People; L0 L1 0
    MAXSTACK = 1
    MAXLOCALS = 1

  // access flags 0x1
  public setCnt(I)V
   L0
    LINENUMBER 11 L0
    ALOAD 0
    ILOAD 1
    PUTFIELD demo/People.cnt : I
   L1
    LINENUMBER 12 L1
    RETURN
   L2
    LOCALVARIABLE this Ldemo/People; L0 L2 0
    LOCALVARIABLE cnt I L0 L2 1
    MAXSTACK = 2
    MAXLOCALS = 2
}
```

使用`meld`工具发现除了类民不同，变量修饰符不同外没什么别的区别

![1546870827683](/home/lx/.config/Typora/typora-user-images/1546870827683.png)

要看到`volatile`到底做了什么，要看汇编代码，也就是`JIT`编译后的代码。

执行`java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly -Xcomp -XX:CompileCommand=compileonly,*User.setCnt demo.Demo`查看`User.setCnt`方法的汇编，在执行类似指令查看`People.setCnt`方法的汇编发现了区别。

(这条指令需要在`$CLASS_HOME/jre/lib/amd64/`下添加`hsdis-amd64.so`文件）

`user.setCnt`

![1546871268794](/home/lx/.config/Typora/typora-user-images/1546871268794.png)

`people.setCnt`

![1546871330851](/home/lx/.config/Typora/typora-user-images/1546871330851.png)

可以看出`User`比`People`多了条` lock addl $0x0,(%rsp) `指令，这条指令就是关键。

`lock`表示锁，`lock`后的指令是原子的，也就是使用缓存一致性协议来保证可见行。这里给`%rsp+0`，也就是说`%rsp`的值(高速缓存中的值)要立即对别的`cpu`(多核心情况下)可见。

`volatile`只能保证修改立即可见，所以会有`ABA`缺陷。`CAS`中的比较并交换操作前也有`lock`指令(多核情况下)。

