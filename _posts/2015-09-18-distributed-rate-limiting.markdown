---
layout: post
title:  "分布式限流"
date:   2015-09-18
categories: redis ratelimiter
---

# 背景
限流是生产中经常遇到的一个场景, 目前现有的一个工具大部分是提供单机限流的能力, 例如 google 的 guava 中提供的 RateLimiter. 但是生产环境大部分是分布式环境, 在多台机器的环境下, 需要的是能对多台机器一起限流的分布式限流. 分布式限流依赖公共的后端存储, 所以还需要自己搭建.

# 算法
说到限流, 首先依赖的是限流的算法, 限流的算法很多包括[令牌桶](https://en.wikipedia.org/wiki/Token_bucket), [漏桶](https://en.wikipedia.org/wiki/Leaky_bucket).

## 滑动窗口
滑动窗口算法的优点在于可以在滑动时间内计算出相对精确的限流数据. 想象一个简单的限流算法, 例如限制在一分钟内最多访问 10 次. 我们在后端存储的结构如下

![一分钟为一个存储单元](https://docs.google.com/drawings/d/1U1CxxEIetqO7YDx2Hcwj6F1DOP7xDsg2R5v7d8sHmVo/pub?w=323&h=130)

假设每个框代表了一分钟, 框中存储了一分钟内的限流数据, 那么问题在于我们想要红色框的限流的数据时将无法计算, 也就是说我们的限流的时间节点的起止时间是固定的. 而滑动窗口之所以为"滑动", 则是为了解决这个问题诞生. 而实际上, 这个方法也是令牌桶的一种变相实现.

滑动实现的核心思想, 在于将时间块切分到更细的精度, 假如我们继续将 1 分钟切分为更小的维度, 例如 5 秒, 那么以后我们的频率计算的时间节点就可以变得更精确, 例如 10:11:00 ~ 10:12:00, 10:11:05 ~ 10:12:05, 10:11:10 ~ 10:12:10 ... 做到 5 秒的精度, 如图

![切分更小的精度的存储单元](https://docs.google.com/drawings/d/1-mXhJj4eseWbc4cwc5cEl5WKXffwMaw2kobUpyutLTc/pub?w=646&h=259)

从另一个角度来讲, 一分钟时间的间隔, 实际上也是一种滑动的特殊情况, 只不过精度一分钟.

# 后端存储
既然说到分布式实现, 则需要考虑公共的后端存储服务, 此处我们选择 redis, 因为 redis 提供了方便的数据结构供我们实现滑动窗口, 主要会用到 redis 中的 map. 具体实现可以参照代码.

## 实现
为了保证单次限流各种操作的原子性, 我们选择使用 lua 脚本执行限流逻辑, 最终会返回是否达到流量限制的结果. 参考 [Part 1](http://www.binpress.com/tutorial/introduction-to-rate-limiting-with-redis/155) [Part 2](http://www.binpress.com/tutorial/introduction-to-rate-limiting-with-redis-part-2/166)
```lua
-- KEYS[1] map key
-- ARGV[1] current time
-- ARGV[2] duration
-- ARGV[3] limitation
-- ARGV[4] precision
-- ARGV[5] permits

local function clear(i1, i2, key, count_key, dele)
    local sum = 0
    for id = i1, i2 do
        local bkey = count_key .. ":" .. id;
        local bcount = redis.call('HGET', key, bkey)
        if bcount then
            sum = sum + tonumber(bcount)
            table.insert(dele, bkey)
        end
    end
    return sum
end

local count_key = "cnt"
local ts_key = "ts"

local key = KEYS[1]
local now = tonumber(ARGV[1])
local duration = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])
local precision = tonumber(ARGV[4])
local permits = tonumber(ARGV[5])

local blocks = math.ceil(duration / precision)
local block_id = math.floor(now / precision) % blocks
local last_ts = redis.call('HGET', key, ts_key)
last_ts = last_ts and tonumber(last_ts) or 0

if last_ts ~= 0 then
    local decr = 0;
    local dele = {}
    local last_id = math.floor(last_ts / precision) % blocks
    local elapsed = now - last_ts;

    if elapsed >= duration then
        -- clear all
        clear(0, blocks - 1, key, count_key, dele)
        if permits > 0 then
            redis.call('HSET', key, ts_key, now)
            redis.call('HINCRBY', key, count_key, permits)
            redis.call('HINCRBY', key, count_key .. ":" .. block_id, permits)
            redis.call('PEXPIRE', key, duration)
        end
        return false
    elseif block_id > last_id then
        decr = decr + clear(last_id + 1, block_id, key, count_key, dele)
    elseif block_id < last_id then
        decr = decr + clear(0, block_id, key, count_key, dele)
        decr = decr + clear(last_id + 1, blocks - 1, key, count_key, dele)
    end

    local cur
    if #dele > 0 then
        redis.call('HDEL', key, unpack(dele))
        cur = redis.call('HINCRBY', key, count_key, -decr)
    else
        cur = redis.call('HGET', key, count_key)
    end

    if tonumber(cur or '0') + permits > limit then
        return true
    end
end

if permits > 0 then
    redis.call('HSET', key, ts_key, now)
    redis.call('HINCRBY', key, count_key, permits)
    redis.call('HINCRBY', key, count_key .. ":" .. block_id, permits)
    redis.call('PEXPIRE', key, duration)
end
return false
```

参数解释
- key : 限流记录的 key, 此处的 key 由外部传入, 一般根据我们需要限流的维度来生成. 例如如果是按 ip 对某个 url 做访问限流限制, 则 key 可能是 url:/test:ip:192.168.1.1
- current time : 当前时间, 使用服务端 redis 时间, 为了保证分布式情况下时间的一致性, 这里的使用通过 redis.time 获取并传入 lua 脚本  
- duration : 限流的总时长, 例如 1 分钟则是 60 * 1000 ms
- limitation : 最高流量限制, 例如每分钟 10 次, 则为 10
- precision : 限流精度, 例如精度是 1s, 则为 1000 ms, 限流精度也是保证能实现上图红框内限流的关键, 精度越小, 限流越精确, block 数也越多, 占用的内存也越大. 实际上上图的简单限流即是 duration = precision 的一种特殊情况
- permits : 本次需要增加多少流量, 对于频率来说一般是 1, 而对于流量来说则是数据流量的字节数

至此分步实现的后端关键实现已经基本完成, 剩下的是做好客户端调用流量的接口.

# 考虑的问题

## redis 集群问题
由于 redis 是集群环境, 集群环境下实际上直接执行 lua 脚本是有问题的. 试想 lua 脚本内可能涉及到多个 key 的操作, 而 redis 实际执行节点的选择也是通过 key 来选择的. 在多 key 情况下可能会造成 lua 脚本内 key 的执行混乱, 所以我们需要先手动选择好 redis 节点. 此处我们可以先用限流的 key 将 redis 选择出来, 再将 lua 脚本传到某个 redis 节点执行. 也就是我们必须要可以通过限流 key 唯一确定一个 redis 节点, 例如 url:/test:ip:192.168.1.1 是可以确定使用某个 redis 节点的.

## 分布式时间问题
分布式系统需要考虑多客户端时间不一致问题, 此处使用 redis 时间解决

## 客户端性能问题
由于这是一个公用的限流服务, 也就是所有接入该服务的应用的每次请求都会调用该服务, 再加上所有接入服务的应用共用一个 redis, 显然如果客户端使用同步等待限流服务的返回结果并不太合适, 会影响客户端的服务调用性能. 所以我们可以使用一种折中策略, 即将限流结果保存到本地, 每次请求直接检查本地限流结果是否被限流, 同时使用异步的方式调用限流服务, 并在异步回调中更新限流结果. 这种做法会让限流数据略有延迟, 但是影响不大.

## 限流服务本身的负载
作为限流服务, 一个主要的作用是限制恶意流量对正常业务造成冲击, 但如果所有流量都需要经过限流服务, 当流量激增的时候, 谁来保证限流服务自己不被压垮? 我的建议是设定一个阈值, 当流量超过某个阈值(这个阈值可以设置为 机器数 * 限流阈值)时, 直接退化为本地限流.