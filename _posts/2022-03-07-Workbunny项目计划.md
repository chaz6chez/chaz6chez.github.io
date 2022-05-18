---
layout: post
title:  "Workbunny项目计划"
image: ''
date:   2022-03-07 18:00:00
image: ''
description: '一个基于golang开发的轻量级高性能任务调度服务的开发计划'
tags:
 - Golang
 - 计划
 - 服务
categories:
 - Chaz6chez
---

# Workbunny/workbunny

    一个基于golang开发的轻量级高性能任务调度服务

- 轻量
- 易用
- 快速
- 集群

## 功能

1. 任务
   1. 任务/子任务/关联任务
   2. 任务启停
   3. 任务查询
2. 作业
   1. 作业添加
   2. 作业监控：日志、耗时、总耗时、占用、总占用等
3. 事件推送/查询
   1. http-webhook
   2. 服务端长轮询API
4. 服务监控：日志、占用、集群状态等
5. Raft集群
6. RESTful Open-API
7. cli-command

## 计划

- 网络库 
  - [Gnet](https://github.com/panjf2000/gnet)
  - 基于libevent、libuv、libev等自己实现 workbunny/loop
- web库 
  - [go-Fiber](https://github.com/gofiber/fiber)
  - 基于fast-http自己实现 workbunny/express
- cli库
  - [urfave/cli](https://github.com/urfave/cli)
  - [cobra](https://github.com/spf13/cobra)
- 数据持久化
  - 基于文件与链表等数据结构自己实现数据持久化存储 workbunny/room
- 调度器
  - 自己实现调度库 workbunny/scheduler
- Raft集群
  - [dragonboat库](https://github.com/lni/dragonboat)

---
    更新于 2022-05-16 11:45