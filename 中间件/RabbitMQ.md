# 1、springboot整合

1、添加依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

2、配置文件中进行相关配置

```yaml
spring: 
  rabbitmq:
    host: 192.168.118.128
    virtual-host: /
    publisher-returns: true #开启发送端消息抵达队列的确认
    template:
      mandatory: true #只要抵达队列，以异步发送优先回调我们这个returnconfirm
    listener:
      simple:
        acknowledge-mode: manual #手动ack消息
    publisher-confirm-type: correlated #开启发送端确认
```

3、添加注解@EnableRabbit