---
title: RedisShake TLS证书设置
date: 2026-01-17
tags: 
    - Redis
    - TLS
categories:
    - 笔记
link: note/redis-shake-tls
description: "使用 RedisShake 向从数据库同步信息时，如果数据库开启了 mTLS 则必须要配置证书验证，但 RedisShake 的文档并没有提供配置证书的选项，那么应该如何配置呢？"
---

## 前言

最近有给 Redis 从数据库同步主数据库信息的需求，通过搜索找到了一款叫 RedisShake 的工具

https://github.com/tair-opensource/RedisShake

如果数据库开启了 mTLS 则必须要配置证书验证，但 RedisShake 的文档并没有提供配置证书的选项，提供的配置文件 `shake.toml` 也没有关于证书的配置项

https://tair-opensource.github.io/RedisShake/zh/reader/sync_reader.html

文档写明的是「不需要配置证书因为 RedisShake 没有校验服务器证书」，但是在开启 mTLS 的情况下服务端必须配置证书，否则会拒绝连接

## 动手！

通过 `github.dev` 发现 RedisShake 其实有编写了加载 ca 证书的功能，但是不知道为什么并没有写进文档中

首先需要确保对应 reader/writer 的 `tls` 设置项已经开启，我使用 `sync_reader` 和 `redis_writer` 进行读取和写入，只需要在设置文件中加上：

```toml
[sync_reader.tls_config]
ca_cert = "/home/ubuntu/redis/ca.crt"
cert    = "/home/ubuntu/redis/redis.crt"
key     = "/home/ubuntu/redis/redis.key"

[redis_writer.tls_config]
ca_cert = "/home/ubuntu/redis/ca.crt"
cert    = "/home/ubuntu/redis/redis.crt"
key     = "/home/ubuntu/redis/redis.key"
```

等号后面填写证书文件的路径即可，由于我使用了 crontab 所以这里填写的是绝对路径，只要确保 `redis-shake` 能够有权限读取到就可以啦~
