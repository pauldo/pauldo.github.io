---
title: laravel-s 499 错误解决
date: 2018-08-17 09:57:04
tags:
---
> 线上 lumen 服务优化为 swoole 后, 性能得到显著提升, 但伴随而来一些奇怪的问题, 需要排查

#### 问题:

线上服务偶现 499 错误, 且问题期间伴随着少许 redis 或数据库连接超时

#### 分析现场:

- 通过 zabbix 观察报错期间, cpu load 和内存使用未发现异常, 且 cpu 使用率和内存使用率偏低
- 线上日志, 通过全链路 TraceID 分析, 在 499 发生的期间, nginx 有 access 日志, 但业务代码并未发现任何日志

#### 分析问题:

[laravel-s](https://github.com/hhxsv5/laravel-s) 默认设置的默认工作进程数(worker_num)为 cpu 核心数 * 2. 从官方文档查询 [worker_num 的定义](https://wiki.swoole.com/wiki/page/275.html): "业务代码为同步阻塞，需要根据请求响应时间和系统负载来调整; 最大不得超过SWOOLE_CPU_NUM * 1000". 推断是因为流量较高的情况下, 如果出现网络闪断/DB或redis超时, 正在处理的请求会产生积压, 没有足够的 worker 进程处理, 请求就会积压在 nginx, 直到网络调用方"不耐烦"主动断开连接, nginx 出现 499. 此时内存和 cpu 没有充分利用, 造成资源浪费

#### 通过压测验证:

测试环境: 4c 8g

测试准备: 将接口 sleep 若干秒时间, 开始压测

压测工具: [wrk](https://github.com/wg/wrk)

###### 先将接口 sleep 3 秒, worker_num 调整到 2, 重启 swoole, 模拟进程资源紧缺时请求等待, 复现 499 问题

使用压测工具10个线程并发, 持续5秒请求

```shell
$ wrk -s post.lua -t 10 -c 10 -d 5s --latency http://service.test/credit/user_credit
Running 5s test @ http://service.test/credit/user_credit
  10 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     0.00us    0.00us   0.00us    -nan%
    Req/Sec     0.00      0.00     0.00    100.00%
  Latency Distribution
     50%    0.00us
     75%    0.00us
     90%    0.00us
     99%    0.00us
  2 requests in 5.01s, 1.70KB read
  Socket errors: connect 0, read 0, write 0, timeout 2
Requests/sec:      0.40
Transfer/sec:     346.72B
```

nginx访问请求日志可以看出, 访问成功的只有2条, 剩下的均为499主动断开连接

```
112.126.95.181 - - [16/Aug/2018:19:32:10 +0800] "POST /credit/user_credit HTTP/1.1" 3.003 3.003 200 682 "-" "-" -{\x22user_id\x22:450119}
112.126.95.181 - - [16/Aug/2018:19:32:10 +0800] "POST /credit/user_credit HTTP/1.1" 3.004 3.004 200 682 "-" "-" -{\x22user_id\x22:450119}
112.126.95.181 - - [16/Aug/2018:19:32:12 +0800] "POST /credit/user_credit HTTP/1.1" 5.007 - 499 0 "-" "-" -{\x22user_id\x22:450119}
112.126.95.181 - - [16/Aug/2018:19:32:12 +0800] "POST /credit/user_credit HTTP/1.1" 5.008 - 499 0 "-" "-" -{\x22user_id\x22:450119}
112.126.95.181 - - [16/Aug/2018:19:32:12 +0800] "POST /credit/user_credit HTTP/1.1" 5.009 - 499 0 "-" "-" -{\x22user_id\x22:450119}
112.126.95.181 - - [16/Aug/2018:19:32:12 +0800] "POST /credit/user_credit HTTP/1.1" 5.010 - 499 0 "-" "-" -{\x22user_id\x22:450119}
```

php请求日志中, 接收到4条请求日志:

```
[2018-08-16 19:32:07.232160] INFO F0C45E8A-541B-EA05-AA8E-A2446761DDAE - - - F0C45E8A-541B-EA05-AA8E-A2446761DDAE - [] - - - ===>/credit/user_credit ip: [112.126.95.181]; params: {"user_id":450119} {"qd_type":"sys"}
[2018-08-16 19:32:07.233247] INFO A9EC701C-C2FC-829C-52F3-8B93EB2548F5 - - - A9EC701C-C2FC-829C-52F3-8B93EB2548F5 - [] - - - ===>/credit/user_credit ip: [112.126.95.181]; params: {"user_id":450119} {"qd_type":"sys"}
[2018-08-16 19:32:10.233018] INFO F0C45E8A-541B-EA05-AA8E-A2446761DDAE - - - F0C45E8A-541B-EA05-AA8E-A2446761DDAE - [] - - - <===/credit/user_credit cost: 3001 ms {"qd_type":"sys"}
[2018-08-16 19:32:10.233913] INFO F0C45E8A-541B-EA05-AA8E-A2446761DDAE - - - F0C45E8A-541B-EA05-AA8E-A2446761DDAE - [] - - - ===>/credit/user_credit ip: [112.126.95.181]; params: {"user_id":450119} {"qd_type":"sys"}
[2018-08-16 19:32:10.234961] INFO A9EC701C-C2FC-829C-52F3-8B93EB2548F5 - - - A9EC701C-C2FC-829C-52F3-8B93EB2548F5 - [] - - - <===/credit/user_credit cost: 3002 ms {"qd_type":"sys"}
[2018-08-16 19:32:10.235735] INFO A9EC701C-C2FC-829C-52F3-8B93EB2548F5 - - - A9EC701C-C2FC-829C-52F3-8B93EB2548F5 - [] - - - ===>/credit/user_credit ip: [112.126.95.181]; params: {"user_id":450119} {"qd_type":"sys"}
[2018-08-16 19:32:13.236693] INFO F0C45E8A-541B-EA05-AA8E-A2446761DDAE - - - F0C45E8A-541B-EA05-AA8E-A2446761DDAE - [] - - - <===/credit/user_credit cost: 3003 ms {"qd_type":"sys"}
[2018-08-16 19:32:13.237531] INFO A9EC701C-C2FC-829C-52F3-8B93EB2548F5 - - - A9EC701C-C2FC-829C-52F3-8B93EB2548F5 - [] - - - <===/credit/user_credit cost: 3002 ms {"qd_type":"sys"}
```

调整压测持续时间为1分钟, 观察系统负载, 发现系统毫无压力, 与线上现象匹配

###### worker_num 调整到 100

再次压测发现, 并发量已经上来了, 就不放示例了, 大家可以自行测试一下

#### 总结

在测试环境进行压测不仅要测试接口正常的性能, 还应该模拟响应耗时比长的场景对服务的影响, 然后想办法最大化利用系统资源提高并发量
