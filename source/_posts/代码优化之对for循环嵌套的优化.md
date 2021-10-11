---
title: 代码优化之对for循环嵌套的优化
---

### 循环嵌套使用原则

答：外大内小

<!--more-->

```java
package com.qin;

public class ForTest {
    public static void main(String[] args) {
        // 外大内小
        outBigInsideSmall();
        // 内大外小
        insideBigOutSmall();
    }

    /**
     * 外大内小
     */
    public static void outBigInsideSmall() {
        // 1. 测试外循环比内循环大
        Long startTime = System.nanoTime();
        for (int i = 0; i < 10000000; i++) {
            for (int j = 0; j < 100; j++) {

            }
        }
        Long endTime = System.nanoTime();
        System.out.println("外大内小耗时： " + (endTime - startTime));
    }


    /**
     * 内大外小
     */
    public static void insideBigOutSmall() {
        // 1. 测试外循环比内循环小
        Long startTime = System.nanoTime();
        for (int i = 0; i < 100; i++) {
            for (int j = 0; j < 10000000; j++) {

            }
        }
        Long endTime = System.nanoTime();
        System.out.println("内大外小耗时： "+(endTime-startTime));
    }

}

```



外大内小耗时： 1965000

内大外小耗时： 9732100