# Spring Boot Proxies

A comprehensive guide to understanding and working with proxies in the Spring Boot ecosystem.

---

## Table of Contents

1. [Overview](#overview)
2. [What is a Proxy?](#what-is-a-proxy)
3. [Types of Proxies in Spring Boot](#types-of-proxies-in-spring-boot)
4. [JDK Dynamic Proxies](#jdk-dynamic-proxies)
5. [CGLIB Proxies](#cglib-proxies)
6. [JDK vs CGLIB: Key Differences](#jdk-vs-cglib-key-differences)
7. [Spring AOP and Proxies](#spring-aop-and-proxies)
8. [Proxy in Common Spring Annotations](#proxy-in-common-spring-annotations)
9. [Proxy Configuration](#proxy-configuration)
10. [Self-Invocation Problem](#self-invocation-problem)
11. [HTTP/Network Proxies in Spring Boot](#httpnetwork-proxies-in-spring-boot)
12. [Proxy Design Pattern vs Spring Proxies](#proxy-design-pattern-vs-spring-proxies)
13. [Best Practices](#best-practices)
14. [Common Pitfalls](#common-pitfalls)
15. [Debugging Proxies](#debugging-proxies)

---

## Overview

Spring Boot relies heavily on the **Proxy pattern** to deliver many of its core features — including dependency injection, transaction management, caching, security, and aspect-oriented programming (AOP). When Spring Boot says it is "managing" a bean, it often means it has wrapped that bean inside a proxy object that intercepts method calls and applies cross-cutting concerns transparently.

Understanding how Spring proxies work is essential for debugging unexpected behavior, designing clean architecture, and avoiding subtle bugs in production applications.

---

## What is a Proxy?

A **proxy** is an object that acts as a substitute or wrapper for another object (the *target*). When a caller invokes a method on the proxy, the proxy intercepts the call, optionally executes additional logic (before and/or after the real method), and then delegates to the actual target object.

```
Caller  →  Proxy (intercepts & decorates)  →  Target Bean (real logic)
```

Spring uses proxies to non-invasively inject behavior like:
- Transaction boundaries (`@Transactional`)
- Caching (`@Cacheable`, `@CacheEvict`)
- Security checks (`@PreAuthorize`, `@Secured`)
- Asynchronous execution (`@Async`)
- Retry logic (`@Retryable`)
- Custom AOP advice

---

## Types of Proxies in Spring Boot

Spring Boot supports two proxy mechanisms:

| Mechanism | Based On | Requires Interface | Default For |
|---|---|---|---|
| **JDK Dynamic Proxy** | `java.lang.reflect.Proxy` | Yes | Interface-based beans |
| **CGLIB Proxy** | Bytecode generation (subclassing) | No | Class-based beans, `@Configuration` |

Spring Boot auto-selects the proxy type based on context, but you can override this behavior explicitly.

---

## JDK Dynamic Proxies

JDK Dynamic Proxies are part of the Java standard library and work by creating a runtime proxy that **implements one or more interfaces**.

### How It Works

1. The target bean must implement at least one interface.
2. Spring creates a proxy class at runtime that also implements those interfaces.
3. All method calls go through `java.lang.reflect.InvocationHandler`, which applies advice and delegates to the real bean.

### Example

```java
public interface OrderService {
    void placeOrder(Order order);
}

@Service
public class OrderServiceImpl implements OrderService {
    @Override
    @Transactional
    public void placeOrder(Order order) {
        // business logic
    }
}
```

When Spring injects `OrderService`, it returns a JDK dynamic proxy, not an `OrderServiceImpl` instance directly.

### Limitations

- The bean **must** implement an interface.
- You cannot inject the concrete class type directly (e.g., `OrderServiceImpl`) — only the interface type.
- Slightly slower due to reflection overhead compared to CGLIB.

---

## CGLIB Proxies

CGLIB (Code Generation Library) works by **subclassing the target class** and overriding its methods with proxy logic. It does not require an interface.

### How It Works

1. CGLIB generates a subclass of the target bean at runtime.
2. The subclass overrides all non-final, non-private methods.
3. Method calls are intercepted via a `MethodInterceptor`.

### Example

```java
@Service
public class PaymentService {
    @Transactional
    public void processPayment(Payment payment) {
        // business logic
    }
}
```

Since `PaymentService` doesn't implement an interface, Spring creates a CGLIB subclass automatically.

### Limitations

- Cannot proxy **final classes** or **final methods** (subclassing is blocked by the JVM).
- Cannot proxy **private methods** (not visible to subclasses).
- Constructor is called twice during bean creation (once for the actual bean, once for the subclass proxy) — relevant when constructors have side effects.

### CGLIB as the Default in Spring Boot

Since Spring Boot 2.x, **CGLIB is the default proxy mechanism** even for beans that implement interfaces, unless:
- The bean is explicitly configured to use JDK proxies.
- The context uses `proxyTargetClass = false`.

```java
// Force JDK proxy for a specific AOP proxy
@EnableAspectJAutoProxy(proxyTargetClass = false)
```

---

## JDK vs CGLIB: Key Differences

| Aspect | JDK Dynamic Proxy | CGLIB |
|---|---|---|
| Mechanism | Interface-based | Subclass-based |
| Interface required | Yes | No |
| Final class support | N/A | No |
| Final method support | N/A | No |
| Performance | Slower (reflection) | Faster (bytecode) |
| Default in Spring Boot | No | Yes (since 2.x) |
| `@Configuration` class | Always CGLIB | Always CGLIB |

---

## Spring AOP and Proxies

**Spring AOP (Aspect-Oriented Programming)** is the primary consumer of Spring's proxy infrastructure. AOP allows you to define **cross-cutting concerns** (logging, transactions, security) as *aspects* that are woven into the application without modifying business code.

### AOP Concepts

| Concept | Description |
|---|---|
| **Aspect** | A module encapsulating cross-cutting concerns |
| **Advice** | The action taken at a join point (before, after, around) |
| **Join Point** | A point in program execution (e.g., method call) |
| **Pointcut** | An expression matching one or more join points |
| **Weaving** | Linking aspects with target objects; Spring does this at runtime via proxies |

### Types of Advice

```java
@Aspect
@Component
public class LoggingAspect {

    // Runs before the matched method
    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore(JoinPoint joinPoint) {
        System.out.println("Before: " + joinPoint.getSignature().getName());
    }

    // Runs after the matched method (always)
    @After("execution(* com.example.service.*.*(..))")
    public void logAfter(JoinPoint joinPoint) {
        System.out.println("After: " + joinPoint.getSignature().getName());
    }

    // Wraps the method — has full control (proceed or not)
    @Around("execution(* com.example.service.*.*(..))")
    public Object logAround(ProceedingJoinPoint pjp) throws Throwable {
        System.out.println("Before proceed");
        Object result = pjp.proceed();
        System.out.println("After proceed");
        return result;
    }

    // Runs only when method returns successfully
    @AfterReturning(pointcut = "execution(* com.example.service.*.*(..))", returning = "result")
    public void logAfterReturning(Object result) {
        System.out.println("Returned: " + result);
    }

    // Runs only when method throws an exception
    @AfterThrowing(pointcut = "execution(* com.example.service.*.*(..))", throwing = "error")
    public void logAfterThrowing(Throwable error) {
        System.out.println("Exception: " + error.getMessage());
    }
}
```

### How Spring AOP Creates Proxies

1. Spring scans for beans that match AOP pointcut expressions.
2. For each matching bean, a proxy is created (JDK or CGLIB based on configuration).
3. The proxy intercepts matching method calls and applies the defined advice.
4. The rest of the application interacts with the proxy, unaware of the interception.

---

## Proxy in Common Spring Annotations

Many well-known Spring annotations are **implemented via proxies** under the hood:

### `@Transactional`

```java
@Service
public class UserService {
    @Transactional
    public void createUser(User user) {
        userRepo.save(user);
        emailService.sendWelcome(user); // also participates in same transaction
    }
}
```

Spring wraps `UserService` in a proxy. When `createUser()` is called externally, the proxy opens a transaction, delegates to the real method, then commits or rolls back based on the outcome.

### `@Cacheable` / `@CacheEvict`

```java
@Service
public class ProductService {
    @Cacheable("products")
    public Product getProduct(Long id) {
        return productRepo.findById(id).orElseThrow();
    }
}
```

The proxy checks the cache before delegating. If a cache hit occurs, the real method is never called.

### `@Async`

```java
@Service
public class NotificationService {
    @Async
    public void sendEmail(String to) {
        // executed in a separate thread pool
    }
}
```

The proxy intercepts the call and submits it to a `TaskExecutor`, returning immediately to the caller.

### `@Retryable` (Spring Retry)

```java
@Service
public class ExternalApiService {
    @Retryable(value = IOException.class, maxAttempts = 3)
    public String callApi() {
        // retried up to 3 times on IOException
    }
}
```

The proxy catches the specified exceptions and retries the method call.

---

## Proxy Configuration

### Force CGLIB Globally

```java
@SpringBootApplication
@EnableAspectJAutoProxy(proxyTargetClass = true)
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

Or via `application.properties`:

```properties
spring.aop.proxy-target-class=true
```

### Force JDK Proxies Globally

```properties
spring.aop.proxy-target-class=false
```

### `@Configuration` Classes Always Use CGLIB

Spring's `@Configuration` classes use CGLIB **unconditionally** to ensure that `@Bean` method calls are intercepted and return singleton instances from the context — not new instances on every call.

```java
@Configuration
public class AppConfig {
    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceB()); // serviceB() goes through the proxy → returns singleton
    }

    @Bean
    public ServiceB serviceB() {
        return new ServiceB();
    }
}
```

If you use `@Configuration(proxyBeanMethods = false)`, CGLIB is skipped and `@Bean` methods act as plain factory methods (useful for lightweight performance optimization when bean singleton behavior is not needed).

---

## Self-Invocation Problem

One of the most common proxy-related pitfalls is **self-invocation** (calling a proxy-advised method from within the same bean).

### The Problem

```java
@Service
public class OrderService {

    public void placeOrder(Order order) {
        // Calls processPayment() directly — bypasses the proxy!
        processPayment(order.getPayment());
    }

    @Transactional
    public void processPayment(Payment payment) {
        // Transaction annotation has NO effect here
    }
}
```

Because `placeOrder()` calls `processPayment()` on `this` (the raw bean, not the proxy), the proxy's interception is bypassed and `@Transactional` has no effect.

### Solutions

**Option 1: Inject the proxy of the current bean**

```java
@Service
public class OrderService {

    @Autowired
    private ApplicationContext context;

    public void placeOrder(Order order) {
        // Fetch the proxy from context
        OrderService self = context.getBean(OrderService.class);
        self.processPayment(order.getPayment());
    }

    @Transactional
    public void processPayment(Payment payment) { ... }
}
```

**Option 2: Inject self**

```java
@Service
public class OrderService {

    @Autowired
    @Lazy
    private OrderService self;

    public void placeOrder(Order order) {
        self.processPayment(order.getPayment());
    }

    @Transactional
    public void processPayment(Payment payment) { ... }
}
```

**Option 3 (Recommended): Refactor into separate beans**

```java
@Service
public class OrderService {
    @Autowired
    private PaymentService paymentService;

    public void placeOrder(Order order) {
        paymentService.processPayment(order.getPayment()); // Goes through PaymentService's proxy
    }
}

@Service
public class PaymentService {
    @Transactional
    public void processPayment(Payment payment) { ... }
}
```

---

## HTTP/Network Proxies in Spring Boot

Beyond Spring's internal bean proxies, Spring Boot applications sometimes need to route **outbound HTTP requests** through a network proxy (corporate firewall, forward proxy, etc.).

### Configure for `RestTemplate`

```java
@Bean
public RestTemplate restTemplate() {
    HttpHost proxy = new HttpHost("proxy.example.com", 8080);
    HttpClient httpClient = HttpClientBuilder.create()
        .setProxy(proxy)
        .build();
    HttpComponentsClientHttpRequestFactory factory =
        new HttpComponentsClientHttpRequestFactory(httpClient);
    return new RestTemplate(factory);
}
```

### Configure for `WebClient` (Reactive)

```java
@Bean
public WebClient webClient() {
    HttpClient httpClient = HttpClient.create()
        .proxy(proxy -> proxy
            .type(ProxyProvider.Proxy.HTTP)
            .host("proxy.example.com")
            .port(8080));
    return WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(httpClient))
        .build();
}
```

### Configure via JVM System Properties

```properties
# In application.properties or JVM args
-Dhttps.proxyHost=proxy.example.com
-Dhttps.proxyPort=8080
-Dhttp.nonProxyHosts=localhost|127.*|*.internal
```

---

## Proxy Design Pattern vs Spring Proxies

| | Design Pattern (GoF) | Spring Proxy |
|---|---|---|
| **Purpose** | Structural — control access to an object | AOP — apply cross-cutting concerns |
| **Creation** | Manual, compile-time | Automatic, runtime |
| **Scope** | Single responsibility | Multiple concerns on same bean |
| **Transparency** | Caller may or may not know | Caller is unaware |

Spring proxies are an automated, runtime application of the classic **Proxy** and **Decorator** design patterns combined with **Chain of Responsibility** for multiple aspects.

---

## Best Practices

1. **Prefer interface-based design** — it works with both JDK and CGLIB proxies and improves testability.
2. **Never call proxy-advised methods internally** — extract to a separate bean to ensure proxy interception.
3. **Don't make Spring beans `final`** — CGLIB cannot subclass final classes.
4. **Avoid final methods on proxied beans** — they cannot be overridden by CGLIB.
5. **Use `@Configuration(proxyBeanMethods = false)`** for lightweight config classes where inter-bean calls are not needed, to improve startup performance.
6. **Be explicit with `@Transactional` propagation** — understand how proxies propagate transactions across bean boundaries.
7. **Keep aspects focused** — each aspect should address exactly one cross-cutting concern.
8. **Use pointcut expressions carefully** — overly broad expressions can unintentionally wrap framework internals.

---

## Common Pitfalls

| Pitfall | Cause | Fix |
|---|---|---|
| `@Transactional` not working | Self-invocation bypasses proxy | Refactor to a separate bean |
| `@Async` not working | Same bean call bypasses proxy | Use a separate service |
| `ClassCastException` on injection | Injecting CGLIB proxy as JDK-proxy type | Match injection type to proxy type |
| Proxying `final` class fails | CGLIB cannot subclass final classes | Remove `final` or use interface |
| Double bean initialization | CGLIB calls constructor twice | Avoid side effects in constructors |
| AOP advice not applied | Bean not loaded via Spring context | Ensure bean is a Spring-managed component |

---

## Debugging Proxies

### Check if a Bean is Proxied

```java
@Autowired
private OrderService orderService;

public void inspect() {
    System.out.println(orderService.getClass().getName());
    // JDK proxy:  com.sun.proxy.$Proxy42
    // CGLIB proxy: com.example.OrderService$$EnhancerBySpringCGLIB$$...
}
```

### Programmatic Check

```java
boolean isProxy = AopUtils.isAopProxy(orderService);
boolean isCglib = AopUtils.isCglibProxy(orderService);
boolean isJdk   = AopUtils.isJdkDynamicProxy(orderService);
```

### Unwrap the Target Bean

```java
Object target = AopProxyUtils.getSingletonTarget(orderService);
// Returns the actual underlying bean, not the proxy
```

### Enable Debug Logging

```properties
logging.level.org.springframework.aop=DEBUG
logging.level.org.springframework.transaction=DEBUG
```

---

## Summary

Spring Boot proxies are the invisible engine powering many of the framework's most powerful features. They transparently intercept method calls to apply transactions, caching, security, async execution, and custom cross-cutting logic — all without modifying your business code.

Key takeaways:
- Spring uses **JDK Dynamic Proxies** (interface-based) and **CGLIB proxies** (subclass-based).
- **CGLIB is the default** in Spring Boot 2.x+.
- `@Transactional`, `@Cacheable`, `@Async`, and AOP aspects all work via proxies.
- **Self-invocation bypasses proxies** — always route proxy-advised calls through the proxy.
- **Final classes and methods** cannot be proxied by CGLIB.
- Understanding proxies is fundamental to diagnosing subtle Spring Boot bugs.

---

*For further reading, see the [Spring Framework AOP Documentation](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#aop).*
