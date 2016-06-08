---
layout: post
title: Code Reading - lua-resty-kafka
date: 2016-06-07 23:05:20
tags: openresty
---

## 0

最近在做一个数据采集的需求，使用nginx做为网关，然后将数据写到kafka，但是官方的API是基于Java语言的，另外感觉在中间再加一层没有必要，所以在GitHub上找了一个lua语言版的API。

[lua-resty-kafka](https://github.com/doujiang24/lua-resty-kafka)



## Overview

1. `producer.lua`

  封装kafka中的producer，用户可以指定sync或async模式，该模块实现不同的逻辑

2. `client.lua`

  封装对kafka集群的连接，获取和更新metadata，根据topic和partition选择broker

3. `broker.lua`

  封装对kafka集群中单个broker的连接

4. `senbuffer.lua`

  发送缓存，无论是sync还是async，消息都是先进入该buffer，发送的时候根据broker进行aggregator

5. `ringbuffer.lua`

  消息缓存，当设为async模式时，消息先进入该buffer

6. `request.lua` `response.lua`

  封装request和response的编解码

7. `errors.lua`

  封装response错误码

通常来说，我们都是使用async模式，所以笔者将着重分析async模式下缓存消息、维护metadata、选择broker、发送消息部分的代码，至于Kafka协议的详细内容，可以参考[A Guide To The Kafka Protocol](https://cwiki.apache.org/confluence/display/KAFKA/A+Guide+To+The+Kafka+Protocol)。

## producer:new

```lua
--producer.lua

function _M.new(self, broker_list, producer_config, cluster_name)
    local name = cluster_name or DEFAULT_CLUSTER_NAME
    local opts = producer_config or {}
    local async = opts.producer_type == "async"
    
    -- 如果是异步模式，并且已经初始化，那么直接返回
    -- 也就是说一个worker中只有一个producer实例
    if async and cluster_inited[name] then
        return cluster_inited[name]
    end

    local cli = client:new(broker_list, producer_config)
    local p = setmetatable({
        client = cli,
        correlation_id = 1,
        request_timeout = opts.request_timeout or 2000,
        retry_backoff = opts.retry_backoff or 100,   -- ms
        max_retry = opts.max_retry or 3,
        required_acks = opts.required_acks or 1,
        -- 计算partition_id的函数
        partitioner = opts.partitioner or default_partitioner,
        error_handle = opts.error_handle,
        async = async,
        socket_config = cli.socket_config,
        -- 消息缓存
        ringbuffer = ringbuffer:new(opts.batch_num or 200, opts.max_buffering or 50000),   -- 200, 50K
        -- 发送缓存
        sendbuffer = sendbuffer:new(opts.batch_num or 200, opts.batch_size or 1048576)
                        -- default: 1K, 1M
                        -- batch_size should less than (MaxRequestSize / 2 - 10KiB)
                        -- config in the kafka server, default 100M
    }, mt)

    if async then
        cluster_inited[name] = p
        -- flush缓存的定时任务
        _timer_flush(nil, p, (opts.flush_time or 1000) / 1000)  -- default 1s
    end
    return p
end
```

```lua
-- client.lua

function _M.new(self, broker_list, client_config)
    local opts = client_config or {}
    local socket_config = {
        socket_timeout = opts.socket_timeout or 3000,
        keepalive_timeout = opts.keepalive_timeout or 600 * 1000,   -- 10 min
        keepalive_size = opts.keepalive_size or 2,
    }

    local cli = setmetatable({
        broker_list = broker_list,
        topic_partitions = {},
        brokers = {},
        client_id = "worker" .. pid(),
        socket_config = socket_config,
    }, mt)

    -- 根据官方Guide的建议，通常不需要定时更新metadata
    -- 发送失败重试的时候更新即可
    if opts.refresh_interval then
        meta_refresh(nil, cli, opts.refresh_interval / 1000) -- in ms
    end

    return cli
end
```

```lua
-- ringbuffer.lua

function _M.new(self, batch_num, max_buffering)
    local sendbuffer = {
        -- 存储单元(topic, key, message)
        queue = new_tab(max_buffering * 3, 0),
        -- 需要flush的数量阈值
        batch_num = batch_num,
        -- buffer的最大容量
        size = max_buffering * 3,
        start = 1,
        -- buffer当前使用量
        num = 0,
    }
    return setmetatable(sendbuffer, mt)
end
```

```lua
-- sendbuffer.lua

function _M.new(self, batch_num, batch_size)
    local sendbuffer = {
        -- topics[topic][partition_id]每个单独建缓存
        topics = {},
        queue_num = 0,
        batch_num = batch_num * 2,
        -- 此处的单位是bytes，包含msg和key
        batch_size = batch_size,
    }
    return setmetatable(sendbuffer, mt)
end
```

## bp:send

```lua
-- producer.lua

function _M.send(self, topic, key, message)
    if self.async then
        local ringbuffer = self.ringbuffer

        -- 消息先进入ringbuffer
        local ok, err = ringbuffer:add(topic, key, message)
        if not ok then
            return nil, err
        end

        -- 检查是否可以、是否需要flush缓存
        if not self.flushing and (ringbuffer:need_send() or is_exiting()) then
            _flush_buffer(self)
        end

        return true
    end

    -- 以下是sync模式，先不关注
    local partition_id, err = choose_partition(self, topic, key)
    if not partition_id then
        return nil, err
    end

    local sendbuffer = self.sendbuffer
    sendbuffer:add(topic, partition_id, key, message)

    local ok = _batch_send(self, sendbuffer)
    if not ok then
        sendbuffer:clear(topic, partition_id)
        return nil, sendbuffer:err(topic, partition_id)
    end

    return sendbuffer:offset(topic, partition_id)
end
```

```lua
-- ringbuffer.lua

function _M.add(self, topic, key, message)
    local num = self.num
    local size = self.size

    if num >= size then
        return nil, "buffer overflow"
    end

    local index = (self.start + num) % size
    local queue = self.queue

    queue[index] = topic
    queue[index + 1] = key
    queue[index + 2] = message

    self.num = num + 3

    return true
end

function _M.need_send(self)
    -- 如果大于opts.batch_num
    return self.num / 3 >= self.batch_num
end
```

## \_flush\_buffer

在async模式下，有两个时机flush缓存，第一个是初始化producer时设置的`_timer_flush`，第二个是send消息时检查是否需要`_flush_buffer`

```lua
-- producer.lua

_flush_buffer = function (self)
    -- 建立一个0-delay timer的意义在于异步执行_flush逻辑
    local ok, err = timer_at(0, _flush, self)
    if not ok then
        ngx_log(ERR, "failed to create timer at _flush_buffer, err: ", err)
    end
end

local function _flush(premature, self)
    if not _flush_lock(self) then
        if debug then
            ngx_log(DEBUG, "previous flush not finished")
        end
        return
    end

    local ringbuffer = self.ringbuffer
    local sendbuffer = self.sendbuffer

    while true do
        -- 从ringbuffer中取数据
        local topic, key, msg = ringbuffer:pop()
        if not topic then
            break
        end

        -- 计算partition_id
        -- 需要获取kafka metadata
        local partition_id, err = choose_partition(self, topic, key)
        if not partition_id then
            partition_id = -1
        end

        -- 添加到发送缓存
        -- 溢出有两个条件：(数量大于batch_num) or (字节数大于batch_size)
        local overflow = sendbuffer:add(topic, partition_id, key, msg)
        if overflow then    -- reached batch_size in one topic-partition
            break
        end
    end

    -- 批量发送
    local all_done = _batch_send(self, sendbuffer)

    if not all_done then
        -- 如果有发送失败的消息，执行配置的error_handle函数
        for topic, partition_id, buffer in sendbuffer:loop() do
            local queue, index, err, retryable = buffer.queue, buffer.index, buffer.err, buffer.retryable

            if self.error_handle then
                local ok, err = pcall(self.error_handle, topic, partition_id, queue, index, err, retryable)
                if not ok then
                    ngx_log(ERR, "failed to callback error_handle: ", err)
                end
            else
                ngx_log(ERR, "buffered messages send to kafka err: ", err,
                    ", retryable: ", retryable, ", topic: ", topic,
                    ", partition_id: ", partition_id, ", length: ", index / 2)
            end

            -- 清空发送失败的buffer
            sendbuffer:clear(topic, partition_id)
        end
    end

    _flush_unlock(self)

    -- 此处判断是否需要flush可能没有必要
    if ringbuffer:need_send() then
        _flush_buffer(self)

    elseif is_exiting() and ringbuffer:left_num() > 0 then
        -- 如果正在退出，那么flush
        -- still can create 0 timer even exiting
        _flush_buffer(self)
    end

    return true
end
```

```lua
-- producer.lua

local function choose_partition(self, topic, key)
    local brokers, partitions = self.client:fetch_metadata(topic)
    if not brokers then
        return nil, partitions
    end

    -- 执行配置的partitioner函数
    return self.partitioner(key, partitions.num, self.correlation_id)
end


-- client.lua

function _M.fetch_metadata(self, topic)
    local brokers, partitions = _metadata_cache(self, topic)
    if brokers then
        return brokers, partitions
    end

    _fetch_metadata(self, topic)

    return _metadata_cache(self, topic)
end

local function _metadata_cache(self, topic)
    if not topic then
        return self.brokers, self.topic_partitions
    end

    local partitions = self.topic_partitions[topic]
    if partitions and partitions.num and partitions.num > 0 then
        return self.brokers, partitions
    end

    return nil, "not foundd topic"
end

local function _fetch_metadata(self, new_topic)
    local topics, num = {}, 0
    -- 取出旧topics
    for tp, _p in pairs(self.topic_partitions) do
        num = num + 1
        topics[num] = tp
    end

    -- 当前需要的new_topic
    if new_topic and not self.topic_partitions[new_topic] then
        num = num + 1
        topics[num] = new_topic
    end

    if num == 0 then
        return nil, "not topic"
    end

    local broker_list = self.broker_list
    local sc = self.socket_config
    -- 构建metadata请求
    local req = metadata_encode(self.client_id, topics, num)

    -- 遍历配置的broker列表，只要有一个返回信息即可
    for i = 1, #broker_list do
        local host, port = broker_list[i].host, broker_list[i].port
        local bk = broker:new(host, port, sc)

        local resp, err = bk:send_receive(req)
        if not resp then
            ngx_log(INFO, "broker fetch metadata failed, err:", err, host, port)
        else
            local brokers, topic_partitions = metadata_decode(resp)
            self.brokers, self.topic_partitions = brokers, topic_partitions

            return brokers, topic_partitions
        end
    end

    ngx_log(ERR, "all brokers failed in fetch topic metadata")
    return nil, "all brokers failed in fetch topic metadata"
end

```

```lua
-- producer.lua

local function _batch_send(self, sendbuffer)
    local try_num = 1
    while try_num <= self.max_retry do
        -- aggregator
        -- 将sendbuffer中的消息按照发送的broker聚合
        local send_num, sendbroker = sendbuffer:aggregator(self.client)
        if send_num == 0 then
            break
        end

        for i = 1, send_num, 2 do
            local broker_conf, topic_partitions = sendbroker[i], sendbroker[i + 1]

            -- 发送消息
            _send(self, broker_conf, topic_partitions)
        end

        -- 如果全部发送成功，那么返回true
        if sendbuffer:done() then
            return true
        end

        -- 更新metadata
        -- 因为发送失败的原因可能是leader更换
        self.client:refresh()

        try_num = try_num + 1
        if try_num < self.max_retry then
            ngx_sleep(self.retry_backoff / 1000)   -- ms to s
        end
    end
end

```

```lua
-- sendbuffer.lua

function _M.aggregator(self, client)
    local num = 0
    local sendbroker = {}
    local brokers = {}

    local i = 1
    -- 遍历所有缓存队列
    for topic, partition_id, queue in self:loop() do
        if queue.retryable then
            -- 选择leader broker
            local broker_conf, err = client:choose_broker(topic, partition_id)
            if not broker_conf then
                -- 如果找不到对应的broker，那么标记err
                self:err(topic, partition_id, err, true)

            else
                if not brokers[broker_conf] then
                    brokers[broker_conf] = {
                        topics = {},
                        topic_num = 0,
                        size = 0,
                    }
                end

                local broker = brokers[broker_conf]
                if not broker.topics[topic] then
                    brokers[broker_conf].topics[topic] = {
                        partitions = {},
                        partition_num = 0,
                    }

                    broker.topic_num = broker.topic_num + 1
                end

                local broker_topic = broker.topics[topic]

                broker_topic.partitions[partition_id] = queue
                broker_topic.partition_num = broker_topic.partition_num + 1

                broker.size = broker.size + queue.size

                -- 限制一次发送的数据量不要大于batch_size太多
                if broker.size >= self.batch_size then
                    sendbroker[num + 1] = broker_conf
                    sendbroker[num + 2] = brokers[broker_conf]

                    num = num + 2
                    brokers[broker_conf] = nil
                end
            end
        end
    end

    -- sendbroker就是需要发送的(broker, messages)
    for broker_conf, topic_partitions in pairs(brokers) do
        sendbroker[num + 1] = broker_conf
        sendbroker[num + 2] = brokers[broker_conf]
        num = num + 2
    end

    return num, sendbroker
end
```

```lua
-- producer.lua

local function _send(self, broker_conf, topic_partitions)
    local sendbuffer = self.sendbuffer
    local resp, retryable = nil, true

    local bk, err = broker:new(broker_conf.host, broker_conf.port, self.socket_config)
    if bk then
        local req = produce_encode(self, topic_partitions)

        resp, err, retryable = bk:send_receive(req)
        if resp then
            local result = produce_decode(resp)

            for topic, partitions in pairs(result) do
                for partition_id, r in pairs(partitions) do
                    local errcode = r.errcode

                    if errcode == 0 then
                        sendbuffer:offset(topic, partition_id, r.offset)
                        -- 发送成功，清空buffer
                        sendbuffer:clear(topic, partition_id)
                    else
                        err = Errors[errcode]

                        -- XX: only 3, 5, 6 can retry
                        local retryable0 = retryable
                        if errcode ~= 3 and errcode ~= 5 and errcode ~= 6 then
                            retryable0 = false
                        end

                        -- 如果返回错误码，那么标记错误，并且判断是否可以重试
                        local index = sendbuffer:err(topic, partition_id, err, retryable0)

                        ngx_log(INFO, "retry to send messages to kafka err: ", err, ", retryable: ", retryable0,
                            ", topic: ", topic, ", partition_id: ", partition_id, ", length: ", index / 2)
                    end
                end
            end

            return
        end
    end

    -- when broker new failed or send_receive failed
    -- 如果没有收到broker的相应，那边标记错误
    -- 如果是receive timeout，那么不要重试
    for topic, partitions in pairs(topic_partitions.topics) do
        for partition_id, partition in pairs(partitions.partitions) do
            sendbuffer:err(topic, partition_id, err, retryable)
        end
    end
end
```
