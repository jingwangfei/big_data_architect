

# Spring 动态代理

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


### 总结  
Spring 中的动态代理是“**根据目标对象是否有接口，在 JDK 代理和 CGLIB 代理之间二选一**”，而非两者结合使用。这种设计既利用了 JDK 代理的原生性（无需第三方依赖），又通过 CGLIB 弥补了无接口场景下的代理能力。
