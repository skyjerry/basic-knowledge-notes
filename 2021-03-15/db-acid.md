# 简述数据库中的 ACID 分别是什么？

## 原子性（Atomicity）

一个事务中的所有操作，或者全部完成，或者全部不完成，不会结束在中间某个环节。事务在执行过程中发生错误，会被回滚到事务开始前的状态，就像这个事务从来没有执行过一样。即事务不可分割。

## 一致性（Consistency）

在事务开始之前和事务结束以后，数据库的完整性没有被破坏。

## 隔离性（Isolation）

数据库允许多个并发事务同时对其数据进行读写和修改的能力，隔离性可以防止多个事务并发执行时由于交叉执行而导致数据的不一致。

## 持久性（Durability）

事务处理结束后，对数据的修改就是永久的，即便系统故障也不会丢失。



## Reference

https://zh.wikipedia.org/wiki/ACID