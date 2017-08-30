---
title: FailOver数据库切换方案
tags: 数据库,切换,FailOver
grammar_cjkRuby: true
---


### 设计目的

	为防止系统因数据库出现问题导致整体服务不可用，能够快速进行数据库切换，做到秒级数据库切换，能够将数据库硬件软件等故障的风险降到最低。