---
layout: post
title:  "【问题排查】Nginx缓存域名解析缓存导致请求异常"
date:   2025-02-26 00:00:00 +0800
tags: troubleshooting
excerpt_separator: <!--more-->
---

记录因 Nginx 缓存域名解析缓存导致请求异常的排查和处理<!--more-->

* TOC
{:toc}

## 现象
1. 业务研发反馈接口调用超时，而该接口地址是一台反向代理机器


## 排查

1. 进入业务容器排查，发现偶发请求耗时过慢的情况，1 分钟后才会有结果返回，正常情况在 200 ms 左右

2. nginx 许多超时设置默认为 60s，怀疑与nginx代理有关，但查看 nginx 无错误日志。继续排查异常请求日志，比对正常与超时请求日志，发现超时请求的 upstream 地址数量为 2 个，第一个地址请求超时地址，返回 504 状态码，第二个地址请求正常，返回 200，符合“1 分钟后才会有结果返回”的情况

    日志格式：
    ```
    log_format main  '$remote_addr $remote_user $time_local $host $request $status $request_time $body_bytes_sent $upstream_addr $upstream_status $upstream_connect_time $upstream_header_time $upstream_response_time $http_referer $http_user_agent $http_x_forwarded_for';
    ```

    超时日志
![20250226-d20da7cdbd.jpg](/assets/images/20250226-d20da7cdbd.jpg)

3. 整理超时 upstream 地址，主要为 119.167.232.204，120.220.145.142。因为反向代理使用是域名地址，而 nginx 默认对域名解析有缓存，启动后解析缓存一直保留，仅会在重启或reload后更新

    nginx 代理配置：
    ![20250226-de7eac7d90d1955b4caf4faeb7669596.png](/assets/images/20250226-de7eac7d90d1955b4caf4faeb7669596.png)

4. 怀疑 openapi-fxg.jinritemai.com 域名的解析ip有更新，在机器上请求域名解析，无 119.167.232.204，120.220.145.142 两个ip。同时，过滤对 119.167.232.204 的请求日志，发现在 ’18/Dec/2024:14:27’ 前的请求都是正常的

5. 基本可以得出结论，本次nginx代理偶发超时是由于upstream域名的解析ip有更新，而nginx的域名解析缓存未更新导致


## 修复
1. 增加域名解析缓存有效时间
另注意，如果其他位置也使用了相同域名做upstream，需要一并处理，否则 resolver 不生效
    ```
    server {
        listen 18080;
        resolver 100.96.0.3 100.96.0.2 valid=60s;
        resolver_timeout 3s;

        server_name  101.126.41.114;
        location / {
            set $proxy_url "openapi-fxg.jinritemai.com";
            proxy_pass https://$proxy_url;
        }
    }
    ```

2. 使用三方模块nginx-upstream-dynamic-servers、ngx_upstream_jdomain 等

3. 使用其他版本 Nginx，如 Tengine、NGINX Plus