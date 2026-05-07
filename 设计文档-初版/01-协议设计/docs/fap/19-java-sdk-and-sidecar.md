# 19 - Java SDK 与 Sidecar

## 1. Java SDK

```java
public interface FapCapabilityPlugin {
    PluginMetadata metadata();
    List<Capability> listCapabilities();
    InvocationResult invoke(InvocationContext context, InvokeRequest request);
    default void cancel(String invocationId) {}
    default PluginHealth health() { return PluginHealth.up(); }
}
```

## 2. Sidecar 部署

```text
Rust FAP Core
  → Plugin RPC (gRPC over UDS/TCP)
  → Java Sidecar
  → Spring Service / Enterprise System
```

## 3. Spring Boot Starter

```xml
<dependency>
  <groupId>com.future-agent</groupId>
  <artifactId>fap-spring-boot-starter</artifactId>
  <version>1.0.0</version>
</dependency>
```

```java
@FapCapability(id = "code.review", riskLevel = RiskLevel.MEDIUM)
public class CodeReviewPlugin implements FapCapabilityPlugin {
    @Override
    public InvocationResult invoke(InvocationContext ctx, InvokeRequest req) {
        // ...
    }
}
```

## 4. 互操作

Java SDK 必须通过 conformance matrix 与 Rust Core 互操作测试。
