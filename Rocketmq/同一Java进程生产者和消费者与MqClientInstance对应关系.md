### 概述

在 RocketMQ 的 Java 客户端中，**MQClientInstance** 与 **生产者/消费者** 的对应关系可以总结为

* **一个 `MQClientInstance` 可以管理多个生产者和消费者（一对多），而一个生产者/消费者只属于唯一一个 `MQClientInstance`。**

* 在同一个 Java 进程中，默认情况下，每个生产者或者消费者启动**大概率**都会创建一个`MQClientInstance`实例。

为什么说大概率呢，因为生产者和消费者在启动时，若没有指定`instanceName`属性时，start方法中会调用`ClientConfig.changeInstanceNameToPID`方法，将`instanceName`修改为pid#当前纳秒数形式。形如91789#1381920211246958

```java
private String instanceName = System.getProperty("rocketmq.client.name", "DEFAULT");
public void changeInstanceNameToPID() {
    if (this.instanceName.equals("DEFAULT")) {
        this.instanceName = UtilAll.getPid() + "#" + System.nanoTime();
    }
}
```

`MQClientInstance`实例的创建过程

MQClientManager.java

```java
public MQClientInstance getOrCreateMQClientInstance(final ClientConfig clientConfig, RPCHook rpcHook) {
    String clientId = clientConfig.buildMQClientId();
    MQClientInstance instance = this.factoryTable.get(clientId);
    if (null == instance) {
        instance =
            new MQClientInstance(clientConfig.cloneClientConfig(),
                this.factoryIndexGenerator.getAndIncrement(), clientId, rpcHook);
        MQClientInstance prev = this.factoryTable.putIfAbsent(clientId, instance);
        if (prev != null) {
            instance = prev;
            log.warn("Returned Previous MQClientInstance for clientId:[{}]", clientId);
        } else {
            log.info("Created new MQClientInstance for clientId:[{}]", clientId);
        }
    }

    return instance;
}
```

ClientConfig.java

```java
public String buildMQClientId() {
    StringBuilder sb = new StringBuilder();
    sb.append(this.getClientIP());

    sb.append("@");
    sb.append(this.getInstanceName());
    if (!UtilAll.isBlank(this.unitName)) {
        sb.append("@");
        sb.append(this.unitName);
    }

    if (enableStreamRequestType) {
        sb.append("@");
        sb.append(RequestType.STREAM);
    }

    return sb.toString();
}

```

可以看到clientId生成规则是ip+instanceName+unitName+stream，默认情况下unitName以及enableStreamRequestType都是没有的。形如192.168.1.9@91789#1381920211246958，因此只要生产者或者消费者不是同一纳秒执行start方法初始化时，那么instanceName就会不一样。