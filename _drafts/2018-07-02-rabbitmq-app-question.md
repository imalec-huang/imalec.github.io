---
layout: post
title:  "RabbitMQ使用过程中出现的堵塞问题"
date:   2018-07-02 13:56:00
categories: 服务中间件
tags: rabbitmq java
---

* content
{:toc}

经过比较久的筛选，rackmq/kafka/rabbitmq/activemq中选择了rabbitmq，具体技术对比网上有很多文章，这里不再赘述，这里的场景使用重要从着眼于一个分布式的可监控的消息队列，性能方面可以通过增加节点解决，扩展性与监控考虑的重点





## 堵塞
(https://stackoverflow.com/questions/48728703/threads-are-all-blocked-when-publish-with-multi-threads-and-channels-rabbitmq/48729039#48729039)
（https://www.cnblogs.com/gordonkong/p/7061625.html）