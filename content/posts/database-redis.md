---
title: "Database：Redis 设计与实现"
author: "Chenghao Zheng"
tags: ["Database"]
categories: ["Reading notes"]
date: 2021-05-08T13:19:47+01:00
draft: false
---

Redis 是一个开源的，内存中的数据结构存储系统，它可以用作数据库、缓存和消息中间件，支持多种类型的数据结构和范围查询。 Redis 内置了复制（replication）、事务（transactions）和不同级别的磁盘持久化（persistence），并通过 Redis哨兵（Sentinel）和自动分区（Cluster）提供高可用性（high availability）。

本篇将结合 [《Redis 设计与实现》](https://book.douban.com/subject/25900156/) 讲解 Redis 的内部机制、单机特性以及多机特性。

# Redis 数据结构与功能实现

### Redis 数据结构

#### 字符串

#### 链表

#### 字典

#### 跳跃表

#### 整数集合

#### 压缩列表

#### 对象



### 发布与订阅

### 事务

### 排序

# 单机数据库的实现

### 数据库

### RDB 持久化

### AOF 持久化

### 服务器

# 多机数据库的实现

### 复制

### Sentinel

### 集群

