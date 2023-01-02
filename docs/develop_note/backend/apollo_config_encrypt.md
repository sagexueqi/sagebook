# Apollo配置项加解密扩展

## 问题背景

在Apollo配置中心维护数据库密码等敏感信息，由于合规与安全的原因不允许明文保存；需要将密码信息保存至类似于AWS Secret Manage的平台，通过API在运行时解密获取。

Apollo官方关于加解密的[示例](https://github.com/apolloconfig/apollo-use-cases/tree/master/spring-boot-encrypt)，是基于`jasypt-spring-boot`实现的。问题在于需要在应用配置秘钥，仍然没有完全解决上述的脱敏问题。

## 解决思路

- 目标：在Apollo客户端配置更新或获取时，在内存中完成解密操作并注入到属性中使用

- Apollo操作配置属性的核心类是: `com.ctrip.framework.apollo.internals.DefaultConfig`，我们需要在拉取新配置后或每次获取key的value值时进行解密；这样我们可以做到从底层来解决这个问题，对上层应用的影响讲到最低

- `com.ctrip.framework.apollo.internals.DefaultConfig`是`com.ctrip.framework.apollo.Config`的子类，Apollo的`Config`对象是通过`com.ctrip.framework.apollo.spi.ConfigFactory`工厂创建的；

- 我们需要改写一个自己的`com.ctrip.framework.apollo.Config`的子类，扩展解密逻辑，并使它在Apollo框架中生效；同时，需要有一个自定义的`ConfigFactory`返回该`Config`对象；

- Apollo使用`com.ctrip.framework.apollo.internals.DefaultInjector`将框架使用的工具类对象注入到内存当中，而且是支持SPI的模式进行扩展的；通过实现`ApolloInjectorCustomizer`，改写获取的`ConfigFactory`对象是返回自定义的`ConfigFactory`

## 解决步骤

### 扩展配置解密逻辑

**拉取新配置后解密**

>
> 核心在于改写`DefaultConfig.updateConfig()`方法；但是该方法是private的，所以需要新建一个新的`DefaultConfigCustomizer`，整体复制`DefaultConfig`的逻辑
>
> 由于DefaultConfig使用了很多package可见范围的类，所以自定义的`DefaultConfigCustomizer`也需要在`com.ctrip.framework.apollo.internals`包下
>
> 优点：拉取配置后一次性解密；缺点：改写已有的代码，侵入性高

- 定义`DefaultConfigCustomizer`，参考`DefaultConfig`的代码实现；只添加`updateConfigBeforeProcessor()`和`updateConfigPostProcessor()`两个空方法

```java
package com.ctrip.framework.apollo.internals;

public class DefaultConfigCustomizer extends AbstractConfig implements RepositoryChangeListener {
    // .... 无需修改的逻辑 .... //

    private void updateConfig(Properties newConfigProperties, ConfigSourceType sourceType) {
        // 扩展：add update config before processor
        updateConfigBeforeProcessor(newConfigProperties, sourceType);

        m_configProperties.set(newConfigProperties);
        m_sourceType = sourceType;

        // 扩展：add update config post processor
        updateConfigPostProcessor(newConfigProperties, sourceType);
    }

    protected void updateConfigBeforeProcessor(Properties newConfigProperties, ConfigSourceType sourceType) {
        // default do nothing
    }

    protected void updateConfigPostProcessor(Properties newConfigProperties, ConfigSourceType sourceType) {
        // default do nothing
    }

    // .... 无需修改的逻辑 .... //
}
```

- 实现`DefaultConfigCustomizer`，实现`updateConfigPostProcessor`方法

```java
package com.muse.framework.apollo.asm;

public class ASMEncryptDefaultConfig extends DefaultConfigCustomizer {
    protected void updateConfigPostProcessor(Properties newConfigProperties, ConfigSourceType sourceType) {
        Set<Object> configKeys = newConfigProperties.keySet();
        if (configKeys != null) {
            SecretsManagerClientEnhance secretsManagerClient = null;

            for (Object keyObject : configKeys) {
                String key   = keyObject.toString();
                String value = newConfigProperties.getProperty(key);

                // check value is 'ASM{xxxx}' format
                if (isASMValue(value)) {
                    log.info("property key:{} value:{}, begin resolver ASM value", key, value);

                    // call aws secrets manager service
                    if (secretsManagerClient == null) {
                        secretsManagerClient = createsSecretsManagerClient();
                    }
                    processorASMValue(secretsManagerClient, newConfigProperties, key, value);
                }
            }

            // close secrets
            if (secretsManagerClient != null) {
                secretsManagerClient.close();
            }
        }
    }
}
```

**每次获取KEY/VALUE时解密**

> 直接继承`DefaultConfig`，重写`getProperty(String, String)`方法
>
> 需要处理好缓存，没必要每次获取都去解密；优点是实现简单，不需要维护一个和`DefaultConfig`逻辑差不多的类，不受框架升级带来的隐藏逻辑变更的风险

- 继承`DefaultConfig`，重写`getProperty()`方法

```java
package com.muse.framework.apollo.asm;

public class ASMEncryptDefaultConfig extends DefaultConfig {
    @Override
    public String getProperty(String key, String defaultValue) {
        String value = super.getProperty(key, defaultValue);

        if (isASMValue(value)) {

            log.info("property key:{} value:{}, begin resolver ASM value", key, value);

            // call aws secrets manager service
            SecretsManagerClientEnhance secretsManagerClient = createsSecretsManagerClient();
            value = processorASMValue(secretsManagerClient, key, value);

            secretsManagerClient.close();
        }

        return value;
    }
}
```

### 注册Config配置工厂

Apollo通过`DefaultInjector`实现组件的注入；同时，在获取组件实例时，支持基于`ApolloInjectorCustomizer`类的SPI扩展；这就为我们将自定义配置工厂注入到框架中提供了可能。

```java
public class DefaultInjector implements Injector {

  @Override
  public <T> T getInstance(Class<T> clazz) {
    try {
      for (ApolloInjectorCustomizer customizer : m_customizers) {
        T instance = customizer.getInstance(clazz);
        if (instance != null) {
          return instance;
        }
      }
      return m_injector.getInstance(clazz);
    } catch (Throwable ex) {
      Tracer.logError(ex);
      throw new ApolloConfigException(
          String.format("Unable to load instance for %s!", clazz.getName()), ex);
    }
  }

  @Override
  public <T> T getInstance(Class<T> clazz, String name) {
    try {
      for (ApolloInjectorCustomizer customizer : m_customizers) {
        T instance = customizer.getInstance(clazz, name);
        if (instance != null) {
          return instance;
        }
      }
      //Guice does not support get instance by type and name
      return null;
    } catch (Throwable ex) {
      Tracer.logError(ex);
      throw new ApolloConfigException(
          String.format("Unable to load instance for %s with name %s!", clazz.getName(), name), ex);
    }
  }

}
```

- 创建自定义配置工厂。Apollo框架采用工厂模式获取组件

```java
package com.muse.framework.apollo.asm;

public class ASMEncryptDefaultConfigFactory extends DefaultConfigFactory {

    @Override
    protected Config createRepositoryConfig(String namespace, ConfigRepository configRepository) {
        return new ASMEncryptDefaultConfig(namespace, configRepository);
    }
}
```

- 创建配置工厂的`ApolloInjectorCustomizer`实现类：`ASMEncryptConfigFactoryInjectorCustomizer`；在Apollo框架获取`ConfigFactory`实现类时，默认返回自定义配置工厂

```java
package com.muse.framework.apollo.asm;

public class ASMEncryptConfigFactoryInjectorCustomizer implements ApolloInjectorCustomizer {
    private static final ASMEncryptDefaultConfigFactory ASM_ENCRYPT_DEFAULT_CONFIG_FACTORY = new ASMEncryptDefaultConfigFactory();

    @Override
    public int getOrder() {
        return 0;
    }

    @Override
    public <T> T getInstance(Class<T> clazz) {
        if (clazz.equals(ConfigFactory.class)) {
            return (T) ASM_ENCRYPT_DEFAULT_CONFIG_FACTORY;
        }

        return null;
    }

    @Override
    public <T> T getInstance(Class<T> clazz, String name) {
        return getInstance(clazz);
    }
}
```

- 在`resources/META-INF/services`路径下创建`com.ctrip.framework.apollo.spi.ApolloInjectorCustomizer`文件

```
com.muse.framework.apollo.asm.ASMEncryptConfigFactoryInjectorCustomizer
```

至此，我们整体实现了Apollo配置中心配置项客户端解密的流程。我们可以在解密的逻辑中，关注特殊配置项，例如`ASM{}`进行解密操作。
