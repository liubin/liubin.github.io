---
layout: post
title: "Bash 常用 if 判断条件"
date: 2021-01-05 11:47:07 +0800
comments: true
categories: 
---

作为一个运维，Shell 技能是必不可少的，只是 Shell 语法确实有些古怪，经常要查文档或者参考之前的代码。

## 数值比较

```
if [ ${foo} -eq 0 ]; then
```

|参数|说明|备注|
|---|---|
|-eq | 相等     | equal                |
|-ne | 不相等 | not equal            |
|-lt | 小于     | less than            |
|-le | 小于等于       | less than or equal   |
|-gt | 大于     | greater than         |
|-ge | 大于等于       | greater than or equal|

## 字符串比较

```
if [ "${foo}" = "bar" ]; then
```

|参数|说明|
|---|---|
|=  | 相等 |
|!= | 不相等 |


## 字符串长度的比较

```
  if [ -z ${foor} ]; then
```

|参数|说明|
|---|---|
|-z | 长度为 0    |
|-n | 长度大于 0 |


## 文件、文件夹判断

```
if [ -d ${foo} ]; then
```

|参数|说明|
|---|---|
|-d | 如果是文件夹则返回true        |
|-f | 是否为普通文件        |
|-s | 文件大小是否大于 0  |
|-e | 存在           |
|-r | 文件可读         |
|-w | 文件可写        |
|-x | 可执行文件        |


最后更新： 2020-1-5



