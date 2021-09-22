---
title: JOL：查看Java 对象布局、大小工具
---

```xml
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.16</version>
</dependency>
```

<!--more-->

```java
System.out.println(ClassLayout.parseInstance(obj).toPrintable());
```

