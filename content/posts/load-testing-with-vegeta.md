+++
date = '2025-05-24T22:56:49+08:00'
title = 'Load Testing With Vegeta'
categories = ["linux", "loadtesting"]
description = "使用vegeta进行压力测试"
+++

## 获取vegeta
GitHub: `https://github.com/tsenart/vegeta`
获取最新的release, 解压并放到`/usr/local/bin`下, 给予读写执行权限

## 使用vegeta压力测试
```shell
# 测试目标
echo "POST http://localhost:8080" > target

# 参数
echo '{"username": "user", "password": "123123123"}' > body

# 每秒700次请求, 持续60秒, 使用上面定义的target和body, Content-Type为application/json
vegeta attack -rate=700 -duration=60s -targets=target -body=body -header="Content-Type: application/json" > result.bin

# 测试报告
vegeta report < result.bin

# 测试报告返回示例
Requests      [total, rate, throughput]         42000, 700.02, 571.65
Duration      [total, attack, wait]             1m13s, 59.998s, 13.414s
Latencies     [min, mean, 50, 90, 95, 99, max]  183.515µs, 7.063s, 6.807s, 12.419s, 13.113s, 14.134s, 16.653s
Bytes In      [total, mean]                     26018836, 619.50
Bytes Out     [total, mean]                     37643502, 896.27
Success       [ratio]                           99.92%
Status Codes  [code:count]                      0:34  200:41966
```

## 生成自定义报告
```shell
vegeta attack -duration=120s -rate=100 -targets=target.list -output=results.bin

# 生成HTML报告
vegeta plot -title=Attack-Results results.bin > results.html

# 生成json报告
vegeta report -type=json results.bin
```
