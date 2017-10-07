---
title: awk命令的应用
date: 2017-10-07 22:37:25
tags: [linux]
---

### awk是linux下的一个强大的文本数据处理工具，熟练运用这个工具在分析日志文件，查找问题的时候帮助很大。
awk脚本是由模式和操作组成的。
模式：
  1. /正则表达式/：使用通配符的扩展集
  2. 关系表达式
  3. 模式匹配，~表示匹配，~!表示不匹配
  4. BEGIN,pattern,END语句块
<!--more-->
操作：大括号内，变量或数组赋值，内置函数和控制流语句

awk命令的结构：
awk 'BEGIN{ commands } pattern{ commands } END{ commands }'

awk内置变量
$n 当前的第n个字段，$0表示当前行的文本内容
NF: 表示当前行的字段数
NR: 表示当前行的行数

### awk使用
统计当前网络连接数:
netstat -n | awk '/^tcp/ {++state[$NF]} END {for(key in state) print key,"\t",state[key]}'

统计前10访问的IP:
cat access.log|awk ‘{print $1}’|sort|uniq -c|sort -nr|head -10

