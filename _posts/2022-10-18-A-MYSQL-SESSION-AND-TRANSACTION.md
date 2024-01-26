---
layout:     post
title:      Mysql：Session && Transaction
subtitle:   Mysql基础回顾：会话与事务
date:       2022-08-18
author:     pandaychen
catalog:    true
tags:
    - Mysql
---


##  0x00    前言

会话（Session）： 指客户端与MySQL服务器之间的一个连接。当客户端连接到MySQL服务器时，服务器会为客户端分配一个会话。会话是数据库操作的基本单位，每个会话都有一个唯一的ID。在会话中，用户可以执行SQL语句来查询和修改数据。会话可以持续很长时间，直到客户端断开与服务器的连接。

事务（Transaction）：事务是数据库操作的一个逻辑单元，它是由一系列的SQL语句组成。事务具有ACID特性（原子性/一致性/隔离性/持久性），这些特性保证了数据库在并发操作和系统故障的情况下，仍然能够保持数据的一致性和完整性。事务可以在一个会话中执行，也可以跨多个会话执行


##  0x0	参考
-   [XORM的七种武器](https://xorm.io/zh/blog/xorm%E7%9A%84%E4%B8%83%E7%A7%8D%E6%AD%A6%E5%99%A8/)