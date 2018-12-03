---
layout:     post
title:      "redis pub/sub模式详解以及在Spring中的应用"
subtitle:   "redis pub/sub模式的实现原理"
date:       2018-12-03 17:00:00
author:     "王磊磊"
header-img: "img/home-bg-o.jpg"
tags:
    - redis
    - Spring
---

Redis pub/sub实现异步操作。之前没有了解过，最近需要临时使用redis实现异步操作，故研究了下如何使用。
### 劣势
使用之前必须声明一般情况下,不推荐使用redis的pub/sub作为消息中间件使用:
<ul>
	<li> 消息可靠性无法保证，发送即丢失，没有可靠性可言</li>
	<li> 数据堵塞风险，时间越久越容易丢失</li>
	<li> 资源消耗问题，sub需要单独占用一个redis链接</li>
</ul>

如果看完以上不足，还是想用，请继续

### 原理
redis pub/sub模式 与redis database无关，当pub message 到db 10， 在db1的consumer也能收到；redis每个服务器上保存了每个channel对应的订阅者的对应关系，每次有client pub新的消息，redis会读取这个对应关系，将消息转发给对应的sub client上。具体参见[redis pubsub](https://redisbook.readthedocs.io/en/latest/feature/pubsub.html)


### 实现
基于spring-data-redis实现redis的pub/sub模式
<br>实现描述：在项目启动时，初始化RedisMessageListenerContainer（用于注册redis 消费者列表）

```java

package com.test.gio.config.redis;

import com.test.gio.service.IoMessageService;
import java.util.List;
import java.util.Optional;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.listener.RedisMessageListenerContainer;

/**
 * @author wade0817wang 
 */
@Configuration
public class GioRedisConfig {

  /**
   * @param connectionFactory
   * @param listeners 依据spring的自动注入listeners，其中IoMessageService接口实现了MessageListener接口
   * @return
   */
  @Bean
  RedisMessageListenerContainer container(RedisConnectionFactory connectionFactory, List<IoMessageService> listeners) {

    RedisMessageListenerContainer container = new RedisMessageListenerContainer();
    container.setConnectionFactory(connectionFactory);
    Optional.ofNullable(listeners).ifPresent(list -> list.forEach(item -> {
      container.addMessageListener(item, item.topic());
    }));
    return container;
  }

}
```
<br>IoMessageService接口继承了MessageListener(redis回调实现接口)
```java
package com.test.gio.service;

import org.springframework.data.redis.connection.MessageListener;
import org.springframework.data.redis.listener.ChannelTopic;

/**
 * 消息队列服务接口
 * @author wade0817wang
 */
public interface IoMessageService extends MessageListener {
  /**
   * 消息队列对应的topic
   */
  ChannelTopic topic();
}

```
<br> 实现类IoMessageServiceImpl
```java
package com.test.gio.service.impl;

import com.test.gio.service.IoMessageService;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.PropertySource;
import org.springframework.data.redis.connection.Message;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.listener.ChannelTopic;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

/**
 * 实现类
 *
 * @author wade0817wang
 */
@Slf4j
@Service
public class IoMessageServiceImpl implements IoMessageService {

  /**
   * 保证唯一
   */
  public static String PUB_CHANNEL = "pub_channel" + System.currentTimeMillis();

  @Autowired
  private RedisTemplate redisTemplate;

  @Override
  @Transactional(propagation = Propagation.REQUIRED, rollbackFor = Exception.class)
  public void onMessage(Message message, byte[] pattern) {
    try {
      Object obj = redisTemplate.getValueSerializer().deserialize(message.getBody());
      //TODO do somthing
    } catch (Exception e) {
      log.error("[Fetch data from Tianyancha] onMessage execute error: {}", e.getMessage());
    }
  }

  @Override
  public ChannelTopic topic() {
    return new ChannelTopic(PUB_CHANNEL);
  }

}

```
<br> pub test类
```java
package com.test.web.service.gio.impl;

import com.test.gio.exception.GioException;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;

/**
 * 测试类
 * 
 * @author wade0817wang
 */
public class PubTest {

  @Autowired
  private RedisTemplate redisTemplate;

  @Override
  public void testPub() throws GioException {
    Integer i = 10;
    redisTemplate.convertAndSend(IoMessageServiceImpl.PUB_CHANNEL, item);
  }
}

```

### 总结
这篇就到这吧。。。






