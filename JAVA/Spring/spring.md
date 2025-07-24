

# 一：Spring 动态代理

## 1.1 Spring 动态代理的机制

Spring 中的动态代理机制并非简单的“CGLIB 和 JDK Proxy 两者结合”，而是**根据目标对象的特征自动选择合适的代理方式**，两者是“二选一”的关系，而非同时使用。

### 具体逻辑如下：

1. **JDK 动态代理（`java.lang.reflect.Proxy`）**  
   - **适用场景**：当目标对象**实现了至少一个接口**时，Spring 默认优先使用 JDK 动态代理。  
   - **原理**：通过 `Proxy.newProxyInstance()` 生成一个**接口的代理对象**（实现了目标对象的所有接口），代理逻辑通过 `InvocationHandler` 接口实现。  
   - **特点**：只代理接口中的方法，不代理目标类本身的非接口方法（如类中独有的 `private` 或 `default` 方法）。


2. **CGLIB 动态代理（Code Generation Library）**  
   - **适用场景**：当目标对象**没有实现任何接口**时，Spring 会使用 CGLIB 代理。  
   - **原理**：通过生成目标类的**子类**（继承目标类）来实现代理，子类中重写目标类的非 `final` 方法，将代理逻辑嵌入其中。  
   - **特点**：可以代理类中的所有非 `final` 方法（包括接口未定义的方法），但无法代理 `final` 类或 `final` 方法（因为子类不能继承/重写）。


3. **Spring 对代理方式的选择逻辑**  
   - 默认规则：有接口则用 JDK 代理，无接口则用 CGLIB 代理。  
   - 可配置强制使用 CGLIB：通过 `@EnableAspectJAutoProxy(proxyTargetClass = true)` 或 XML 配置 `<aop:aspectj-autoproxy proxy-target-class="true"/>`，强制 Spring 对所有目标对象（无论是否有接口）都使用 CGLIB 代理。  


## 1.2 JAVA动态代理机制

### 一、核心机制：**基于接口创建代理类**
`Proxy.newProxyInstance` **不会创建接口的子类**（接口无法被继承），而是**动态生成一个实现了所有目标接口的具体类**，并实例化该类的对象。这个过程可以拆解为：

1. **动态生成字节码**：  
   JVM 在运行时创建一个新类（通常名为 `$ProxyN`），该类实现了所有传入的接口（如 `interface A`, `interface B`）。

2. **继承 `java.lang.reflect.Proxy`**：  
   生成的代理类继承自 `java.lang.reflect.Proxy`，并实现目标接口。例如：
   ```java
   public final class $Proxy0 extends Proxy implements A, B {
       // 自动生成的方法实现
   }
   ```

3. **实例化代理对象**：  
   通过反射创建 `$Proxy0` 的实例，并关联 `InvocationHandler` 用于处理方法调用。


### 二、关键代码示例
假设存在接口 `Subject`：
```java
public interface Subject {
    void request();
}
```

使用 `Proxy.newProxyInstance` 创建代理：
```java
Subject proxy = (Subject) Proxy.newProxyInstance(
    Subject.class.getClassLoader(),  // 类加载器
    new Class<?>[]{Subject.class},   // 代理实现的接口
    new InvocationHandler() {        // 方法调用处理器
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println("Before method call");
            // 调用目标对象的方法（此处省略目标对象）
            System.out.println("After method call");
            return null;
        }
    }
);

proxy.request(); // 调用代理方法
```

**运行时生成的代理类大致结构**：
```java
public final class $Proxy0 extends Proxy implements Subject {
    private InvocationHandler h;

    public $Proxy0(InvocationHandler h) {
        super(h);
        this.h = h;
    }

    @Override
    public void request() {
        try {
            Method m = Subject.class.getMethod("request");
            h.invoke(this, m, null); // 委托给 InvocationHandler 处理
        } catch (Exception e) {
            throw new UndeclaredThrowableException(e);
        }
    }
}
```


### 三、核心特性总结
1. **仅支持接口代理**：  
   代理类必须实现至少一个接口，无法代理具体类（除非使用 CGLIB）。

2. **方法调用委派**：  
   代理类的所有方法调用都会转发到 `InvocationHandler` 的 `invoke` 方法，由用户自定义处理逻辑。

3. **动态生成与加载**：  
   代理类的字节码在运行时生成，并通过指定的类加载器加载到 JVM 中。


### 四、与 CGLIB 的对比
| **特性**               | **JDK 动态代理**               | **CGLIB 代理**                |
|------------------------|------------------------------|-----------------------------|
| **代理机制**           | 实现接口                     | 继承具体类                   |
| **目标对象要求**       | 必须实现接口                 | 可以是具体类或抽象类         |
| **方法限制**           | 只能代理接口方法             | 可以代理非 `final` 的 public/protected 方法 |
| **性能**               | 调用开销略高（反射）         | 调用开销较低（直接继承）     |
| **典型应用**           | Spring AOP（默认接口代理）   | Spring AOP（无接口时）       |


### 五、常见误区澄清
1. **代理类是否继承接口？**  
   ❌ 错误。接口无法被继承，代理类是**实现**接口的具体类。

2. **能否代理类而非接口？**  
   ❌ JDK 动态代理不行，需使用 CGLIB。

3. **代理类的父类是谁？**  
   ✅ 所有 JDK 动态代理类都继承自 `java.lang.reflect.Proxy`。


### 六、源码验证
通过以下代码可以查看动态生成的代理类字节码：
```java
// 保存代理类的字节码到文件
System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");

// 创建代理
Subject proxy = (Subject) Proxy.newProxyInstance(...);
```

生成的 `$Proxy0.class` 文件可通过反编译工具查看其结构。


### 七、总结
`Proxy.newProxyInstance` 的核心逻辑是：
1. **动态生成实现目标接口的具体类**（继承自 `Proxy`）。
2. **实例化该类对象**，并绑定 `InvocationHandler`。
3. **通过反射调用处理器的 `invoke` 方法**处理所有接口方法。

这种机制使得 JDK 动态代理只能作用于接口，而 CGLIB 通过继承类的方式弥补了这一限制。
