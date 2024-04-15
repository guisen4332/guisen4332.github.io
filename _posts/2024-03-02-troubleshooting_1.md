---
layout: post
title:  "【问题排查】while read line 异常退出：TMOUT 非无限"
date:   2024-03-02 00:00:00 +0800
tags: troubleshooting
excerpt_separator: <!--more-->
---

记录因 TMOUT 参数设置导致的 while read 循环非预期退出的排查及解决<!--more-->

* TOC
{:toc}

## 现象
1. 业务使用 inotifywait 监听文件改动，while 循环持续读取文件改动以完成后续更新
```
# 简单示意
inotifywait -mq -e close_write,delete,moved_to ${DIR}/ | while read file
do
    ....
done
```

2. 正常情况下应有三个进程
![image](/assets/images/c760e315-40b4-4438-af9e-f041e32d396d.png)

3. 出现 while 循环异常退出的情况，无法正常进行更新


## 排查
1. 加执行日志，多次手动同步，while 正常执行，无法复现异常

2. 观察系统调用，read line 等待中
![image](/assets/images/20240302_alngaalg.png)

3. read line 非预期退出，退出时收到 SIGALRM 信号，该信号主要用于定时器机制下的超时中断处理
![image](/assets/images/20240302_lngljre.png)

4. 怀疑 read 命令有默认超时，查阅源码未发现相关内容

5. 排查等待过程中，经常发生 ssh 连接自动断开，而该现象一般 TMOUT 参数有关，观察 TMOUT 设置，为 300 s，刚好和 read 调用收到 SIGALRM 信号的间隔吻合
![image](/assets/images/20240302_cnlglnt.png)

## TMOUT 解释
![image](/assets/images/20240302_alnreelntg.png)

## 修复
了解到 TMOUT 参数是针对等保认证专门设置为 300，系统全局设置不方便调整，故在相关监听脚本中调整
```
export TMOUT=0
```
