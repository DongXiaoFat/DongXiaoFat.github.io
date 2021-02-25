---
title: Android OOM Low Memory Killer 机制
author: DongXiaoFat
date: 2021-02-26 08:45:00 +0800
categories: [Android, Framework]
tags: [Framework]
excerpt: 介绍 Android oom_adj.
math: true
mermaid: true
thumbnail: /assets/img/favicons/av.jpg
---
# Android OOM Low Memory Killer 机制
本文主要介绍Android的low memory killer的 oom_adj的相关概念，以及根据一些案例来阐述了解oom_adj对于做Android应用开发的重要意义。

## 1. Low Memory Killer中进程的分类以及各类进程的adj值

 > 在Android的low memroy killer机制中，会对于所有进程进行分类，对于每一类进程会有其oom_adj值的取值范围，oom_adj值越高则代表进程越不重要，在系统执行低杀操作时，会从oom_adj值越高的开始杀。系统low memeory killer机制下对于进程的级别的以变量的形式定义在framework/base/core/java/com/android/server/am/ProcessList.java类中，可总结成下表：

 |名称|说明|值(P)|值(Q)|
 |:----|:----|:----|:----|
 |INVALID_ADJ   |无效值| -10000|-10000|
 |NATIVE_ADJ    |native进程不被系统管理| -1000|-1000|
 |SYSTEM_ADJ    |系统进程|-900|-900|
 |PERSISTENT_PROC_ADJ   |持久化的系统进程|-800|-800|
 |PERSISTENT_SERVICE_ADJ|关联系统或持久的进程|-700|-700|
 |FOREGROUND_APP_ADJ|前台应用|0|0|
 |PERCEPTIBLE_RECENT_FOREGROUND_APP_ADJ||-|50|
 |VISIBLE_APP_ADJ|可见APP|100|100|
 |PERCEPTIBLE_APP_ADJ|可感知APP进程，如后台音乐播放|200|200|
 |PERCEPTIBLE_LOW_APP_ADJ||-|250|
 |BACKUP_APP_ADJ|当前托管备份操作的进程|300|300|
 |HEAVY_WEIGHT_APP_ADJ|后台重量级进程|400|400|
 |SERVICE_ADJ|托管应用服务的进程|500|500|
 |HOME_APP_ADJ|Home应用进程|600|600|
 |PREVIOUS_APP_ADJ|前一个应用，应用切换|700|700|
 |SERVICE_B_ADJ|B类服务|800|800|
 |CACHED_APP_MIN_ADJ|托管不可见activity进程的最小 ADJ|900|900|
 |CACHED_APP_LMK_FIRST_ADJ||-|950|
 |CACHED_APP_MAX_ADJ|托管不可见activity进程的最大 ADJ|906|999|
 |UNKNOWN_ADJ|不知道用在那的 ADJ|1001|1001|

