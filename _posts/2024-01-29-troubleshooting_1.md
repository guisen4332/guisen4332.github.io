---
layout: post
title:  "【问题排查】内存泄露：curl 版本缺陷"
date:   2024-01-29 00:00:00 +0800
tags: troubleshooting
excerpt_separator: <!--more-->
---

记录 curl/NSS 库导致的系统内存泄露问题 <!--more-->

* TOC
{:toc}

## 背景
网站开发反馈业务上线后内存使用较高，不符合预期，希望帮忙确认原因

```
服务器信息
系统版本：CentOS release 6.9 (Final)
业务环境：nginx + php
机器配置：8C16G * 2 台
curl 版本：7.19.7
```

## 排查
1. 观察系统整体内存使用总量不高，且无明显内存异常过大的进程

    ![top](/assets/images/img_v3_026q_1441436b-670b-4d9a-8ee4-61f2f258c70g.jpg)

1. 由于 php-fpm 使用的是 static 模式且设置的子进程数量为 256，怀疑机器内存使用过高是大量子进程内存累计的结果，统计了 php-fpm 所有子进程以及系统进程的使用内存，发现总使用量与监控相差甚大
    ```
    ps aux|grep php-fpm|awk -v sum=0 '{sum += $6} END{print sum}'
    ```

1. 查看 /proc/meminfo 时，发现其 SReclaimable 过于庞大，使用 slabtop 工具分析，发现主要为 dentry 内存占用
    ![meminfo](/assets/images/img_v3_027b_c8416bb1-b8a9-4b9c-9245-8a36167a8a8g.jpg)

1. 查找可能导致 dentry 内存上涨的原因，发现 curl-7.19.7依赖的 NSS 库存在 dentry 泄漏的bug，发起 https 请求时会触发，而机器刚好是使用 curl-7.19.7 版本。后续使用相同 curl 版本的测试机器成功复现

1. 同步相关发现至开发侧，了解是否有使用 php-curl 扩展，该扩展会使用相关依赖。开发反馈，业务使用 php-curl 扩展，且一直大量使用 https ，未出现过相关问题

1. 后续测试及代码更新也无法直接修复，使用祖传命令释放可回收内存。次日未能继续观察到内存上涨，原因未知
    ```
    /bin/sync && /bin/sync && /bin/echo 2 > /proc/sys/vm/drop_caches && /bin/echo 0 > /proc/sys/vm/drop_caches
    ```

1. 后续开发排查复现内存上涨，定位到使用微信支付 sdk 进行网络请求时会导致内存泄露，升级测试机器 curl 版本测试，内存不再上涨

1. 已定位到修复手段，但未能确认为何业务中普通 https 请求不会导致内存泄露而使用微信支付 sdk 时则会触发，建议开发确认微信支付 sdk 是否使用 php-curl 扩展，如使用，使用方式和业务中 https 请求的处理有什么不同

1. 开发侧排查发现业务封装的公共方法中特别设置了 curl_setopt($curl, CURLOPT_SSL_VERIFYPEER, 0) 以禁止curl验证对等证书, 而微信支付SDK底层使用的Guzzle网络请求框架默认开启了curl验证对等证书，所以微信转账功能开启后会触发系统内存上涨

1. 了解业务侧关闭 CURLOPT_SSL_VERIFYPEER 的背景，开发侧表示可能是为了本地测试方便，针对该请求，建议开发后续评估下开启 CURLOPT_SSL_VERIFYPEER 参数

## 结论

>服务器内存上涨问题是由 curl-7.19.7 版本依赖的 NSS 库的dentry泄露bug引起。业务系统中请求其它带有 https 的第三方接口并没有出现的内存上涨问题，这是因为封装的公共方法中特别设置了 curl_setopt($curl, CURLOPT_SSL_VERIFYPEER, 0) 以禁止curl验证对等证书，而微信支付SDK底层使用的 Guzzle 网络请求框架默认开启了curl验证对等证书，所以微信转账功能开启后会触发系统内存上涨

## 修复

### curl 版本升级
```
# 证书备份
cp /etc/pki/tls/certs/ca-bundle.crt /etc/pki/tls/certs/ca-bundle.crt.bak
curl http://curl.haxx.se/ca/cacert.pem -o /etc/pki/tls/certs/ca-bundle.crt

# 新增 yum 源
vim /etc/yum.repos.d/city-fan-for-curl.repo

[CityFanforCurl]
name=City Fan Repo
baseurl=http://www.city-fan.org/ftp/contrib/yum-repo/rhel6/x86_64/
enabled=0
gpgcheck=0

# 升级 curl 
yum update curl --enablerepo=CityFanforCurl -y

# 检查版本
curl -V
php -r 'var_dump(curl_version());'
```

### 业务公共方法开启 curl 验证对等证书
该操作主要是为了提高业务系统安全性，在该设置开启前，需要保证已对 curl-7.19.7 版本机器升级

## 参考
- https://bugzilla.redhat.com/show_bug.cgi?id=1057388
