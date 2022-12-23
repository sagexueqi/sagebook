# 后端开发笔记

**Dubbo本地服务启动不注册到注册中心，添加JVM参数：**

```shell
-Ddubbo.registry.register=false
```

**Spring-Cloud-Discovery应用本地启动不注册到注册中心，添加JVM参数：**

```shell
-Dspring.cloud.service-registry.auto-registration.enabled=false
```

**Dubbo服务直连，添加JVM参数：**

```shell
-Dxxx.xxxFacade=dubbo://ip:port
```