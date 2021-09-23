---
title: Spring 升级到5.3.x之后，使用BeanUtils 导致性能下降，GC次数急剧增加，性能急剧下降
---

Spring 升级到5.3.x之后，使用BeanUtils 会导致性能下降，GC次数急剧增加，性能急剧下降

<!--more-->

从 Spring 5.3.x 开始，BeanUtils  开始通过创建 ResolvableType这个 进行属性复制

> Spring 5.3.x 版本的 对象复制

```java

private static void copyProperties(Object source, Object target, @Nullable Class<?> editable,
			@Nullable String... ignoreProperties) throws BeansException {

		Assert.notNull(source, "Source must not be null");
		Assert.notNull(target, "Target must not be null");

		Class<?> actualEditable = target.getClass();
		if (editable != null) {
			if (!editable.isInstance(target)) {
				throw new IllegalArgumentException("Target class [" + target.getClass().getName() +
						"] not assignable to Editable class [" + editable.getName() + "]");
			}
			actualEditable = editable;
		}
		PropertyDescriptor[] targetPds = getPropertyDescriptors(actualEditable);
		List<String> ignoreList = (ignoreProperties != null ? Arrays.asList(ignoreProperties) : null);

		for (PropertyDescriptor targetPd : targetPds) {
			Method writeMethod = targetPd.getWriteMethod();
			if (writeMethod != null && (ignoreList == null || !ignoreList.contains(targetPd.getName()))) {
				PropertyDescriptor sourcePd = getPropertyDescriptor(source.getClass(), targetPd.getName());
				if (sourcePd != null) {
					Method readMethod = sourcePd.getReadMethod();
					if (readMethod != null) {
                        // 这里每一个属性都进行 创建ResolvableType ，导致每次复制 会创建出来大量的 ResolvableType 
						ResolvableType sourceResolvableType = ResolvableType.forMethodReturnType(readMethod);
						ResolvableType targetResolvableType = ResolvableType.forMethodParameter(writeMethod, 0);

						// Ignore generic types in assignable check if either ResolvableType has unresolvable generics.
						boolean isAssignable =
								(sourceResolvableType.hasUnresolvableGenerics() || targetResolvableType.hasUnresolvableGenerics() ?
										ClassUtils.isAssignable(writeMethod.getParameterTypes()[0], readMethod.getReturnType()) :
										targetResolvableType.isAssignableFrom(sourceResolvableType));

						if (isAssignable) {
							try {
								if (!Modifier.isPublic(readMethod.getDeclaringClass().getModifiers())) {
									readMethod.setAccessible(true);
								}
								Object value = readMethod.invoke(source);
								if (!Modifier.isPublic(writeMethod.getDeclaringClass().getModifiers())) {
									writeMethod.setAccessible(true);
								}
								writeMethod.invoke(target, value);
							}
							catch (Throwable ex) {
								throw new FatalBeanException(
										"Could not copy property '" + targetPd.getName() + "' from source to target", ex);
							}
						}
					}
				}
			}
		}
	}
```

> spring-beans 5.2.16.RELEASE 的 对象复制

```java
private static void copyProperties(Object source, Object target, @Nullable Class<?> editable,
			@Nullable String... ignoreProperties) throws BeansException {

		Assert.notNull(source, "Source must not be null");
		Assert.notNull(target, "Target must not be null");

		Class<?> actualEditable = target.getClass();
		if (editable != null) {
			if (!editable.isInstance(target)) {
				throw new IllegalArgumentException("Target class [" + target.getClass().getName() +
						"] not assignable to Editable class [" + editable.getName() + "]");
			}
			actualEditable = editable;
		}
		PropertyDescriptor[] targetPds = getPropertyDescriptors(actualEditable);
		List<String> ignoreList = (ignoreProperties != null ? Arrays.asList(ignoreProperties) : null);

		for (PropertyDescriptor targetPd : targetPds) {
			Method writeMethod = targetPd.getWriteMethod();
			if (writeMethod != null && (ignoreList == null || !ignoreList.contains(targetPd.getName()))) {
				PropertyDescriptor sourcePd = getPropertyDescriptor(source.getClass(), targetPd.getName());
				if (sourcePd != null) {
					Method readMethod = sourcePd.getReadMethod();
					if (readMethod != null &&
							ClassUtils.isAssignable(writeMethod.getParameterTypes()[0], readMethod.getReturnType())) {
						try {
							if (!Modifier.isPublic(readMethod.getDeclaringClass().getModifiers())) {
								readMethod.setAccessible(true);
							}
							Object value = readMethod.invoke(source);
							if (!Modifier.isPublic(writeMethod.getDeclaringClass().getModifiers())) {
								writeMethod.setAccessible(true);
							}
							writeMethod.invoke(target, value);
						}
						catch (Throwable ex) {
							throw new FatalBeanException(
									"Could not copy property '" + targetPd.getName() + "' from source to target", ex);
						}
					}
				}
			}
		}
	}
```





### 原因

Spring 5.3.x 开始，BeanUtils  开始通过创建 ResolvableType这个进行属性复制

每一个属性都进行 创建ResolvableType ，导致每次复制 会创建出来大量的 ResolvableType 

最终导致 YoungGC 明显增高



### 测试

```java

public class TestBean {

    private Integer age;
    private String name;
    private String address;

    public TestBean() {
    }

    public TestBean(Integer age, String name, String address) {
        this.age = age;
        this.name = name;
        this.address = address;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getAddress() {
        return address;
    }

    public void setAddress(String address) {
        this.address = address;
    }
    
    public static void main(String[] args) {
        TestBean testBean1 = new TestBean(32, "qinjp", "东三巷" );
        TestBean testBean2 = new TestBean();
        for (int i = 0; i > -1; i++) {
            BeanUtils.copyProperties(testBean1, testBean2);
            System.out.println(i);
        }
    }
    
}

spring-beans 5.2.16.RELEASE 和 spring-beans 5.3.9 这两个依赖去执行这个代码

<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-beans</artifactId>
    <version>5.3.9</version>
</dependency>

<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-beans</artifactId>
    <version>5.2.16.RELEASE</version>
</dependency>

    
    
```



> VM 参数使用

```java
-XX:+UnlockExperimentalVMOptions -XX:+UseEpsilonGC -Xmx32m
```

使用 EpsilonGC，也就是在堆内存满的时候，不执行 GC

最大堆内存是 32m

在内存耗尽之前，**不同版本的** `BeanUtils.copyProperties` **分别能执行多少次**



试验结果是：

**spring-beans 5.2.16.RELEASE** **是 72456次，**

**spring-beans 5.3.9 是 7474 次**

> 耗時

```
spring-beans 5.3.9
StopWatch '': running time = 19488899601 ns
---------------------------------------------
ns         %     Task name
---------------------------------------------
19488899601  100%  

19488

spring-beans 5.2.16.RELEASE
StopWatch '': running time = 1709024300 ns
---------------------------------------------
ns         %     Task name
---------------------------------------------
1709024300  100%  

1709

```

**spring-beans 5.3.9 耗時 19488 ms**

**spring-beans 5.2.16.RELEASE 耗時 1709 ms**

**这是相当大的差距超十倍了啊**





针对这个问题，spring-framework github 里有 [Issue](https://github.com/spring-projects/spring-framework/issues/27246).

https://github.com/spring-projects/spring-framework/issues/27246

### 解决

BeanUtils.copyProperties 的地方，替换成使用 BeanCopier，并且封装了一个简单类：

```java
public class BeanUtils {
    
    private static final Cache<String, BeanCopier> CACHE = Caffeine.newBuilder().build();

    public static void copyProperties(Object source, Object target) {
        Class<?> sourceClass = source.getClass();
        Class<?> targetClass = target.getClass();
        BeanCopier beanCopier = CACHE.get(sourceClass.getName() + " to " + targetClass.getName(), k -> {
            return BeanCopier.create(sourceClass, targetClass, false);
        });
        beanCopier.copy(source, target, null);
    }
}

耗时：性能相对最好
StopWatch '': running time = 1076779400 ns
---------------------------------------------
ns         %     Task name
---------------------------------------------
1076779400  100%  

1076

```

或者 hutool 的 BeanUtil

```java
<dependency>
    <groupId>cn.hutool</groupId>
    <artifactId>hutool-all</artifactId>
    <version>5.7.12</version>
</dependency>

BeanUtil.copyProperties(testBean1, testBean2);


思路缓存source和target 的 bean 属性信息 


/**
 * Bean属性缓存<br>
 * 缓存用于防止多次反射造成的性能问题
 * @author Looly
 *
 */
public enum BeanDescCache {
	INSTANCE;

	private final SimpleCache<Class<?>, BeanDesc> bdCache = new SimpleCache<>();

	/**
	 * 获得属性名和{@link BeanDesc}Map映射
	 * @param beanClass Bean的类
	 * @param supplier 对象不存在时创建对象的函数
	 * @return 属性名和{@link BeanDesc}映射
	 * @since 5.4.2
	 */
	public BeanDesc getBeanDesc(Class<?> beanClass, Func0<BeanDesc> supplier){
		return bdCache.get(beanClass, supplier);
	}
}

然后循环执行
public PropDesc setValue(Object bean, Object value) {
    if (null != this.setter) {
    ReflectUtil.invoke(bean, this.setter, value);
    } else if (ModifierUtil.isPublic(this.field)) {
    ReflectUtil.setFieldValue(bean, this.field, value);
    }
    return this;
}

hutool BeanUtil 耗时：
StopWatch '': running time = 8414185100 ns
---------------------------------------------
ns         %     Task name
---------------------------------------------
8414185100  100%  

8414


```











