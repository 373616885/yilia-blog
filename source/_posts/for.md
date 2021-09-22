---
title: 错误的优化原则-避免在循环体中创建对象
---



for 循环创建对象 变量声明在循环体内更好

<!--more-->

### **Java优化编程的法则有一条**

> 避免在循环体中创建对象,即使该对象占用内存空间不大

```java
for (int i = 0; i < 10000; ++i) {
  Object obj = new Object();
  System.out.println("obj= "+ obj);
} 
```

> 应该改成

```java
Object obj = null;

for (int i = 0; i < 10000; ++i) {
  obj = new Object();
  System.out.println("obj= "+ obj);
} 
```



### **上面的优化原则是错误的**

**循环外申明变量不但效率不会变高，在循环外申明变量，内存占用会更大！不但没有正面作用，反而有负面作用！**



原因：

1. 将变量声明在循环体外的方式没能能节省点空间，反而多了3 bit（多了一行代码导致偏移量增加，还有定义的Slot 不能复用，循环体外作用域还在不能复用，循环体内作用域已经结束，可以复用里面的Slot ）
2. 声明在循环体外部的变量，人为地将其的声明周期拉长了，数据有可能被自己不小心破坏了，隐性的bug增加，还有可能对其他变量命名带来冲突
3. 增强for循环是循环体内声明变量的方式--官方推荐的遍历方式，为什么官方要推崇这种方式，不解释，jdk源码都是写在循环体内的，官方都这样写了，我们还需要纠结什么呢



### **测试代码**

```java
package test;

public class VariableInsideOutsideLoopTest {
    public void outsideLoop() {
        Object o;
        int i = 0;
        while (++i < 100) {
            o = new Object();
            o.toString();
        }
        Object b = 1;
    }

    public void intsideLoop() {
        int i = 0;
        while (++i < 100) {
            Object o = new Object();
            o.toString();
        }
        Object b = 1;
    }
}
```

> 上面的代码编译成class，反编译出来的样子是这样的

```java
package test;

public class VariableInsideOutsideLoopTest {
    public VariableInsideOutsideLoopTest() {
    }

    public void outsideLoop() {
        int i = 0;

        while(true) {
            ++i;
            if(i >= 100) {
                Object b = 1;
                return;
            }

            Object o = new Object();
            o.toString();
        }
    }

    public void intsideLoop() {
        int i = 0;

        while(true) {
            ++i;
            if(i >= 100) {
                Object b = 1;
                return;
            }

            Object o = new Object();
            o.toString();
        }
    }
}
```



**反编译出来的代码一模一样！！结论不言而喻**



### **内存比较**

```java
jdk 11  vm参数
-Xlog:gc*    

public class Loop {
    public void loop() {
        Object o;
        int i = 0;
        while (++i < 100) {
            o = new Object();
            o.toString();
        }
    }
}    
循环体外：
[0.012s][info][gc,heap] Heap region size: 1M
[0.016s][info][gc     ] Using G1
[0.016s][info][gc,heap,coops] Heap address: 0x0000000701000000, size: 4080 MB, Compressed Oops mode: Zero based, Oop shift amount: 3
[0.193s][info][gc,heap,exit ] Heap
[0.194s][info][gc,heap,exit ]  garbage-first heap   total 262144K, used 6144K [0x0000000701000000, 0x0000000800000000)
[0.194s][info][gc,heap,exit ]   region size 1024K, 8 young (8192K), 0 survivors (0K)
[0.194s][info][gc,heap,exit ]  Metaspace       used 6476K, capacity 6511K, committed 6784K, reserved 1056768K
[0.194s][info][gc,heap,exit ]   class space    used 556K, capacity 570K, committed 640K, reserved 1048576K    
public class Loop {
    public void loop() {
        int i = 0;
        while (++i < 100) {
            Object o = new Object();
            o.toString();
        }
    }
}
循环体内
[0.012s][info][gc,heap] Heap region size: 1M
[0.016s][info][gc     ] Using G1
[0.016s][info][gc,heap,coops] Heap address: 0x0000000701000000, size: 4080 MB, Compressed Oops mode: Zero based, Oop shift amount: 3
[0.193s][info][gc,heap,exit ] Heap
[0.193s][info][gc,heap,exit ]  garbage-first heap   total 262144K, used 6144K [0x0000000701000000, 0x0000000800000000)
[0.193s][info][gc,heap,exit ]   region size 1024K, 8 young (8192K), 0 survivors (0K)
[0.193s][info][gc,heap,exit ]  Metaspace       used 6466K, capacity 6511K, committed 6784K, reserved 1056768K
[0.193s][info][gc,heap,exit ]   class space    used 555K, capacity 570K, committed 640K, reserved 1048576K                                            
```



**区别**

**注意：类名和方法名要一致，不然会出现一些差异**

class space 循环体外比循环体内多了 1K   （循环体外 多了一行代码）

Metaspace 方法区： 循环体外比循环体内多了 10K  

原因：循环体外和循环体内  o 这个变量 都用到了一个Slot ( 数组下标) （多了一行代码导致偏移量增加）

但 循环体外 作用域 [ 20, 20 + 8]  （[Start,Start+Length]）偏移量 + 长度

循环体内 作用域 [ 20, 20 + 5]  （[Start,Start+Length]）偏移量 + 长度

总结：都申请了一个Slot的空间（使用的空间一样）---循环体外多了一行代码导致导致偏移量增加--class内存增加

这影响很小，Metaspace 多了一点，但实际heap中还是用了一个Slot的空间



**用 javap 寻找原因（个人猜测）**

```java
javac -g:vars Loop.java
javap -v Loop   
    
 /**
  * 循环体内
  */
public class Loop {
    public void inp() {
        int i = 0;
        while (++i < 1000) {
            Object o = new Object();
            o.toString();
        }
    }
}
   public void inp();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: iconst_0
         1: istore_1
         2: iinc          1, 1
         5: iload_1
         6: sipush        1000
         9: if_icmpge     28
        12: new           #2                  // class java/lang/Object
        15: dup
        16: invokespecial #1                  // Method java/lang/Object."<init>":()V
        19: astore_2
        20: aload_2
        21: invokevirtual #3                  // Method java/lang/Object.toString:()Ljava/lang/String;
        24: pop
        25: goto          2
        28: return
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
           20       5     2     o   Ljava/lang/Object;
            0      29     0  this   Lcom/example/demo/test/Loop;
            2      27     1     i   I
      StackMapTable: number_of_entries = 2
        frame_type = 252 /* append */
          offset_delta = 2
          locals = [ int ]
        frame_type = 25 /* same */

 /**
  * 循环体外
  */   
public class Loop {
    public void out() {
        Object o;
        int i = 0;
        while (++i < 1000) {
            o = new Object();
            o.toString();
        }
    }
}           
    public void out();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: iconst_0
         1: istore_2
         2: iinc          2, 1
         5: iload_2
         6: sipush        1000
         9: if_icmpge     28
        12: new           #2                  // class java/lang/Object
        15: dup
        16: invokespecial #1                  // Method java/lang/Object."<init>":()V
        19: astore_1
        20: aload_1
        21: invokevirtual #3                  // Method java/lang/Object.toString:()Ljava/lang/String;
        24: pop
        25: goto          2
        28: return
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
           20       8     1     o   Ljava/lang/Object;
            0      29     0  this   Lcom/example/demo/test/Loop;
            2      27     2     i   I
      StackMapTable: number_of_entries = 2
        frame_type = 253 /* append */
          offset_delta = 2
          locals = [ top, int ]
        frame_type = 25 /* same */

```





### 总结

首先都用了一个Slot (4字节--4 * 8 bit =32位   1 byte = 8 bit)

循环体外  o  用了一个Slot里的 8 Length

循环体内  o  用了一个Slot里的 5 Length

这 o 变量的作用域少了3 Length

循环一次少移动3 bit ，1亿次少移动35M的空间





### slot复用导致可能导致内存进一步减少

```java

/**
 * @author qinjp
 * @date 2021/8/4
 */
public class Test {

    public void outsideLoop() {
        Object o;
        int i = 0;
        while (++i < 100) {
            o = new Object();
            o.toString();
        }
        Object b = 1;
    }

    public void intsideLoop() {
        int i = 0;
        while (++i < 100) {
            Object o = new Object();
            o.toString();
        }
        Object b = 1;
    }

}
javac -g:vars Test.java
javap -v Test   

  public void outsideLoop();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=4, args_size=1
         0: iconst_0
         1: istore_2
         2: iinc          2, 1
         5: iload_2
         6: bipush        100
         8: if_icmpge     27
        11: new           #2                  // class java/lang/Object
        14: dup
        15: invokespecial #1                  // Method java/lang/Object."<init>":()V
        18: astore_1
        19: aload_1
        20: invokevirtual #3                  // Method java/lang/Object.toString:()Ljava/lang/String;
        23: pop
        24: goto          2
        27: iconst_1
        28: invokestatic  #4                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
        31: astore_3
        32: return
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
           19       8     1     o   Ljava/lang/Object;
            0      33     0  this   Lcom/example/demo/Test;
            2      31     2     i   I
           32       1     3     b   Ljava/lang/Object;
      StackMapTable: number_of_entries = 2
        frame_type = 253 /* append */
          offset_delta = 2
          locals = [ top, int ]
        frame_type = 24 /* same */

  public void intsideLoop();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: iconst_0
         1: istore_1
         2: iinc          1, 1
         5: iload_1
         6: bipush        100
         8: if_icmpge     27
        11: new           #2                  // class java/lang/Object
        14: dup
        15: invokespecial #1                  // Method java/lang/Object."<init>":()V
        18: astore_2
        19: aload_2
        20: invokevirtual #3                  // Method java/lang/Object.toString:()Ljava/lang/String;
        23: pop
        24: goto          2
        27: iconst_1
        28: invokestatic  #4                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
        31: astore_2
        32: return
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
           19       5     2     o   Ljava/lang/Object;
            0      33     0  this   Lcom/example/demo/Test;
            2      31     1     i   I
           32       1     2     b   Ljava/lang/Object;
      StackMapTable: number_of_entries = 2
        frame_type = 252 /* append */
          offset_delta = 2
          locals = [ int ]
        frame_type = 24 /* same */



```



**outsideLoop在stack frame中定义了4个slot, 而intsideLoop只定义了3个slot!!!**

**outsideLoop中，变量o和b分别占用了不同的slot，在intsideLoop中，变量o和b复用一个slot**

**outsideLoop的stack frame比intsideLoop多占用4个字节内存（一个slot占用4个字节）**

**由于在intsideLoop中，o和b复用了同一个slot，所以，当b使用slot 2的时候，这是变量o已经“不复存在”，所以o原来引用的对象就没有任何引用，它有可能立即被GC回收（注意是有可能，不是一定），腾出所占用heap内存。**

**intsideLoop存在可能，在某些时间点，使用的heap内存比outsideLoop少不少**



**这个例子中少的内存微不足道，但是假设这个方法执行时间很长，o引用的对象是一个大对象时，还是有那么点意义。**



### 总结

 变量声明在循环体内比循环体外更好



























