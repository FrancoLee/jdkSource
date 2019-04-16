## CAS源码

---

[转](	https://blog.csdn.net/isea533/article/details/80301535) 

整个 <font color="dd0000">java.util.concurrent</font> 中的无阻塞共享内存和锁的实现都是用CAS实现。对于锁来说那就是期望是没有其他线程占有该锁，如果没占有，就设置自己占有该锁，当占有成功时，返回值 `true`，此时其他线程就不能再获取这个锁，但是他们会一直调用 CAS 尝试占有，这情况下所有线程在自己的CPU时间片执行，不需要线程切换。

以下是unsafe.cpp(hotspot/src/share/vm/prims/unsafe.cpp)中整形CAS的源码：

```c++
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x))
  UnsafeWrapper("Unsafe_CompareAndSwapInt");
  oop p = JNIHandles::resolve(obj);
  jint* addr = (jint *) index_oop_from_field_offset_long(p, offset);
  return (jint)(Atomic::cmpxchg(x, addr, e)) == e;
UNSAFE_END
```

`e`的值一般是`Volatile`类型，这样才能保证`e`是当前内存中最新的值.

宏展开后:

```c
  "C" {
            jboolean Unsafe_CompareAndSwapInt (JNIEnv * env, jobject unsafe, jobject obj, jlong offset, jint e, jint x ) {
                JavaThread * thread = JavaThread::thread_from_jni_environment (env);
                ThreadInVMfromNative __tiv (thread);
                HandleMarkCleaner __hm (thread);
                Thread * __the_thread__ = thread;
                os::verify_stack_alignment (); 
                ;
                oop p = JNIHandles::resolve (obj);
                jint * addr = (jint *) index_oop_from_field_offset_long(p, offset);
                return (jint) (Atomic::cmpxchg (x, addr, e)) ==e;
            }
        }
```

重点代码:

```c++
  oop p = JNIHandles::resolve (obj);
  jint * addr = (jint *) index_oop_from_field_offset_long(p, offset);
  return (jint) (Atomic::cmpxchg (x, addr, e)) ==e;
```

*(oop*)handle 分解开就是 (oop*)handle 转换为 oop 类型的指针，最后 *指针 就是取该指针的值。

index_oop_from_field_offset_long 方法就是用 p 的地址加上 offset 得到这个值的具体内存地址。

最后执行 `Atomic::cmpxchg(x, addr, e)` 方法，并用返回值和 `e` 进行比较。如果返回值和期望值<font color="dd0000">`这里预期指的内存中预期的值，不是操作后预期的值`</font>相同就会返回 `true`。

` Atomic::cmpxchg(x,addr,e)`源码：(hotspot/src/os_cpu/)	

```c
#define LOCK_IF_MP(mp) "cmp $0, " #mp "; je 1f; lock; 1: "
inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
  int mp = os::is_MP();
  __asm__ volatile (LOCK_IF_MP(%4) "cmpxchgl %1,(%3)"
                    : "=a" (exchange_value)
                    : "r" (exchange_value), "a" (compare_value), "r" (dest), "r" (mp)
                    : "cc", "memory");
  return exchange_value;
}
```

` os::is_MP()` 源码:（hostpot/src/share/vm/runtime/os.hpp）

```c
static inline bool is_MP() {
    // During bootstrap if _processor_count is not yet initialized
    // we claim to be MP as that is safest. If any platform has a
    // stub generator that might be triggered in this phase and for
    // which being declared MP when in fact not, is a problem - then
    // the bootstrap routine for the stub generator needs to check
    // the processor count directly and leave the bootstrap routine
    // in place until called after initialization has ocurred.
    return (_processor_count != 1) || AssumeMP;
  }
```

总的来说就是判断是否是多核CPU，如果是就用`lock`汇编指令锁定总线(优化后使用缓存一致性协议，但竞争大了还是会锁总线？没深入了解)来使得之后的操作成为一个原子操作。之后便是执行`cmpxchgl %1,(%3)`

`compare_value`是预期的值

`exchange_value`源操作数

`dest`目的操作数地址，(%3)表示(dest)，即取`dest`内存的值

`"a" (compare_value)` 表示将`compare_value`值读入`eax`寄存器

`cmpxchgl %1,(%3)` 比较`eax`寄存器的值和`(dest)`的值是否相等，相等则将`exchange_value`的值赋值给`dest`并把`ZF=1`,不等则将`exchange_value(%1)`的值赋值给`eax`寄存器，并设置`ZF=0`。

`cmpxchgl %1,(%3)`的大致语义为: `eax==%3`则 `%3=%1`,`eax!=%3`则`eax=%1`.

`=a`表示最后将`exchange_value`的值写入`eax`

最后`return exchange_value`

若`compare_value==dest`则`exchange_value=eax=compare_value`；

若`compare_value!=dest`则`exchange_value=eax=exchange_value`；

`Atomic::cmpxchg(x,addr,e)`的语意：如果`e==addr`将`addr`指向的地址值改成`x`,并返回`e`;如果`e!=addr`,不修改`addr`的值,返回`x`。

注：CAS操作会出现ABA现象，即在执行`Atomic::cmpxchg(x,addr,e)`时(还未执行到`lock`汇编指令)，`addr`的值由`e`变成`b`,再变成`e`.这样返回的值为`e`。也就是说`Atomic::cmpxchg(x,addr,e)`对于ABA现象返回的还是A.