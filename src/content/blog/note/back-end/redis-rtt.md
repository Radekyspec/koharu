---
title: Redis RTT 优化
date: 2026-01-17
catalog: true
tags: 
    - Redis
    - TLS
categories:
    - [笔记, 后端]
description: "后端服务器和 Redis 数据库沟通时，多个操作需要多次网络往返导致 RTT 成倍增加，本文介绍了一些服务端常见的优化手段"
tocNumbering: false
---

记录一下几种在服务器中用到过的优化 Redis RTT 的方法，改善服务器响应延时

## 使用 Promise.all 并行发送请求

### 优势

* 简单易懂，对现有代码改动最小

* 充分利用 IO，提高并发能力

### 缺点

* 传输的数据量并没有变化

* 仍然需要等待所有请求完成

### 示例

```ts
await Promise.all([
    redis.hset(...),
    redis.incr(...),
    ...
]);
```

## 使用 Lua 脚本

### 优势

* 原子性操作，Lua 脚本在数据库中执行时是原子的，在脚本执行期间，所有服务器活动都会被阻塞。这意味着脚本的所有影响要么尚未发生，要么已经发生。

* 减少 RTT ，将多个操作打包成一个脚本发送，减少客户端-服务器通信次数，显著降低网络延迟影响

* 逻辑在服务器端执行，减少应用层处理，特别是 Redis 中的 Lua 脚本执行速度快

* 脚本可在数据库中存储和重复使用，无需每次都重新上传逻辑

* 将数据处理逻辑从应用层转移到数据库层，减少客户端代码复杂度

### 缺点

* Lua 脚本在数据库中执行，难以调试，错误追踪和日志输出受限

* 复杂的 Lua 脚本可能阻塞数据库，影响其他客户端的请求处理

* 业务逻辑分散在数据库和应用层，长期维护成本较高

* 不适合 CPU 密集型操作

### 示例

Redis 中可以现场加载 Lua 脚本；或将 Lua 脚本缓存，需要的时候传入脚本的 SHA-1 调用

需要注意的是 Redis 脚本缓存始终是易失性的。它不被视为数据库的一部分，也不会被持久化。服务器重启、故障转移期间副本接管主角色，或者通过显式操作，都可能清除缓存。这意味着缓存的脚本是短暂的，缓存内容随时可能丢失。

这是 Redis 官方对于 Lua 脚本的使用说明：

https://redis.io/docs/latest/develop/programmability/eval-intro/

* 现场加载：

```ts
const luaScript = "...";
await redis.eval(luaScript, numkeys, ...args);
```

* 保存后调用：

```ts
const luaScript = `...`;
let luaScriptSha = null;

try {
    return await redis.evalsha(luaScript, numkeys, ...args);
} catch {
    try {
        luaScriptSha = await redis.script("LOAD", luaScript);
        return await redis.evalsha(luaScript, numkeys, ...args);
    } catch (err: any) {
        err.message = `Load Script Error: ${err.message}, check script sha values.`;
        throw err;
    }
}
```

除了 Redis 返回的哈希值，脚本的 SHA-1 哈希值还可以通过以下方法在浏览器中生成（注意头尾换行符，推荐使用 `\n` 将脚本连成单行后执行）：

```js
async function sha1(message) {
    const encoder = new TextEncoder();
    const data = encoder.encode(message);
    const hashBuffer = await crypto.subtle.digest('SHA-1', data);
    const hashArray = Array.from(new Uint8Array(hashBuffer));
    return hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
}

sha1(luaScript).then(hex => console.log(hex));
```

## 使用 Redis pipelining

### 什么是 Redis pipelining?

https://redis.io/docs/latest/develop/using-commands/pipelining/

流水线技术是一种已广泛应用数十年的技术。请求/响应服务器可以实现成即使客户端尚未读取旧的响应也能处理新的请求。这样一来，就可以向服务器发送多个命令而无需等待回复，并最终一步读取所有回复。

许多 POP3 协议实现已经支持此功能，从而显著加快从服务器下载新邮件的速度。

Redis 从早期版本就支持流水线操作，因此无论你运行的是哪个版本，都可以将流水线与 Redis 结合使用。

### 优势

* 和 Lua 脚本一致，通过一次 RTT 即可执行多个命令，而无需等待前一个命令返回

* 通过减少系统调用和降低上下文切换开销，显著提高给定 Redis 服务器每秒可执行的操作数量

### 缺点

* 不适合一次处理大量命令，客户端使用流水线发送命令时，服务器将被迫对回复进行排队，这会占用内存。

* 回复的顺序必须严格对应请求顺序，过多命令堆积可能阻塞客户端，服务器响应延迟会累积

* 客户端需要负责具体数据处理逻辑

* 与 Lua 脚本相比，pipelining 缺乏原子性