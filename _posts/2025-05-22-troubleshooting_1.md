---
layout: post
title:  "【问题排查】grafana 显示 pod 内存使用过高"
date:   2025-05-22 00:00:00 +0800
tags: troubleshooting
excerpt_separator: <!--more-->
---

记录 grafana 上显示 pod 内存与实际内存使用不符合的问题排查<!--more-->

* TOC
{:toc}

## 现象
1. grafana 监控面板上发现业务容器内存使用过高
![20250522-0646D5B4-12AC-424A-BDC2-ECD3704D0781.jpeg](/assets/images/20250522-0646D5B4-12AC-424A-BDC2-ECD3704D0781.jpeg)

2. 使用 kubectl top、ps、top 等工具发现业务进程实际内存均大幅低于监控数值
![20250522-683F6A62-078E-49C9-9F77-D77928E4A552.jpeg](/assets/images/20250522-683F6A62-078E-49C9-9F77-D77928E4A552.jpeg)

## 排查
1. 首先确认面板中指标查询规则，发现使用 container_memory_usage_bytes 指标
![20250522-0FF51667-078F-40A2-8BF6-3C2D1C0C2818.png](/assets/images/20250522-0FF51667-078F-40A2-8BF6-3C2D1C0C2818.png)

2. kubectl top 命令使用的是container_memory_working_set_bytes 指标，ps、top 内存也主要查看 RSS/RES 指标

3. 进一步查询发现
- container_memory_usage_bytes = container_memory_rss + container_memory_cache + kernel memory
- container_memory_working_set_bytes = container_memory_usage_bytes - total_inactive_file（未激活的匿名缓存页）

    可以发现 container_memory_usage_bytes 相对普通内存使用多了 container_memory_cache 和 kernel memory。那 container_memory_cache 具体表示啥？对主要对应 Memory Cgroup文件中 memory.stat 的 cache 字段，也即是页缓存（Page Cache），主要用于提升系统对磁盘文件的读写性能，减少直接访问慢速磁盘 I/O 的次数

4. 那 container_memory_usage_bytes 过高是否会影响容器正常运行呢？比如是否会触发内存 limit 限制导致 OOM 呢？
其实并不会，limit 重启的判断指标是container_memory_working_set_bytes，也即是容器真实使用内存量。但同时，我们也需要关注容器是否有非预期读写操作，导致大量使用页缓存，避免影响系统稳定

## 参考
- https://help.aliyun.com/zh/prometheus/support/why-are-the-memory-values-obtained-in-containers-inconsistent
- https://www.orchome.com/6745
- https://help.aliyun.com/zh/arms/application-monitoring/developer-reference/memory-metrics#p-fn5-bew-0ll