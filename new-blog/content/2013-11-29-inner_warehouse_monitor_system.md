Title: 仓库作业机器监控系统设计与实现
Date: 2013-11-29
Author: youngsterxyf
Tags: 技术, 总结, 笔记, Golang
Slug: inner_warehouse_monitor_system

近期在参与一个仓库作业机器监控项目。该项目的需求背景是：公司的电商业务在全国各地有多处或大或小的仓库，仓库的作业人员（没有IT技术背景）经常反馈/投诉作业机器断网、断电、连不了服务等问题。实际情况经常与反馈的不一致，但运维侧并没有数据可以证明，所以才有了这个项目的需求。

为了避免大量监控上报数据影响到生产系统的网络服务，系统采用如下结构：

