---
title: "Azure Log Analytics：与 Power BI 集成"
date: 2022-03-12T08:03:50+00:00
draft: false
tags:
  - Power BI
  - Azure Log Analytics
categories:
  - 技术笔记
author: "Davis ZHANG"
featured_image: "/images/30.jpg"
avatar: "/images/Davis.jpg"
level: "入门"
description: "本文介绍如何通过 Azure Log Analytics 监控 Power BI 工作区性能。"
---



### 前述

Azure Log Analytics是Azure Monitor中的一项分析服务。本文将讲述通过Log Analytics与Power BI集成的方式，获取Power BI工作区内的日志信息，包括各PBI数据集的CPU消耗，查询数量及性能，以及数据刷新时长、数据刷新失败监测及用户查询活动等信息，这将极大地帮助Power BI管理员及其成员对工作区进行优化和管理，以及在出现负载问题时，快速找出问题的根源。

### 原理

基本流程与原理：

![在这里插入图片描述](https://img-blog.csdnimg.cn/a2c32c19482b43629744386291149eee.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

### 实施

##### 准备工作

1.在Azure Portal中创建Log Analytics，可参考[此文档](https://docs.microsoft.com/en-us/azure/azure-monitor/logs/quick-create-workspace)。

![在这里插入图片描述](https://img-blog.csdnimg.cn/86dcb5e247674416b7eb27a93abc32c2.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

2.启用'microsoft insights', 可参考[此文档](https://docs.microsoft.com/en-us/power-bi/transform-model/log-analytics/desktop-log-analytics-configure#enable-the-microsoftinsights-resource-provider)。

3.在Power BI中启用Log Analytics

![在这里插入图片描述](https://img-blog.csdnimg.cn/d7353dcddcc14431ae460688d0079072.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

##### 连接到Log Analytics

1.配置权限。此步骤，你也可以参考[微软文档](https://docs.microsoft.com/en-us/power-bi/transform-model/log-analytics/desktop-log-analytics-configure#set-permissions)，但其中内容并非最佳实践（截止到本文发布时的版本）。文档提到需要Power BI Service以及配置Log Analytics的用户都拥有Owner权限，但依据**最低权限原则**，Power BI Service只需拥有Monitor Contributor权限即可。

![请添加图片描述](https://img-blog.csdnimg.cn/7a7c6eecc1684ea6aa2387cfed1da0f5.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

2.连接PBI工作区与Log Analytics工作区

![在这里插入图片描述](https://img-blog.csdnimg.cn/d466d6e9582a4e6dbb629b3bde3a0978.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

出现以下错误，一般是权限设置错误，确保Power BI Serice拥有Monitor Contributor权限（而非reader）
![在这里插入图片描述](https://img-blog.csdnimg.cn/e020b62e191249f2b9363331c4745604.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

连接成功后如下所示：

![请添加图片描述](https://img-blog.csdnimg.cn/41aa2ff14ed745e497af32fc33f4368a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

### Log Analytics 应用

利用Log Analytics分析Power BI工作区内容有两种方式：1.直接在Log Analytics工作区中查询；2.安装Power BI App: Log Analytics for Power BI Datasets Template ([安装教程](https://docs.microsoft.com/en-us/power-bi/transform-model/log-analytics/install-as-engine-app))

##### 在Log Analytics工作区中查询和分析PBI内容

点击Logs选项卡，就可以使用查询语句来检索所有有关PBI工作区的日志数据，如下：

![请添加图片描述](https://img-blog.csdnimg.cn/78b91fceb1b944d1a1403d2d95ad28e7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

也许我们不熟悉查询的写法，但[文档中这些用例](https://docs.microsoft.com/en-us/power-bi/transform-model/log-analytics/desktop-log-analytics-configure#sample-log-analytics-kql-queries)是非常不错的参考资料。

##### 使用Power BI App: Log Analytics for Power BI Datasets

使用此方法则无需自行编写查询，利用既有模板提供的丰富可视化进行分析，虽然缺少灵活性，但覆盖的内容十分全面。

1.安装设置。此处设置该App读取Log数据的起止时间，例如下图即取近7日数据：

![在这里插入图片描述](https://img-blog.csdnimg.cn/51cf76da02d04e7a91c9bf3b74391364.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

此处注意是读取LA工作区，而非PBI工作区：

![请添加图片描述](https://img-blog.csdnimg.cn/f6943f945df74944970686570612f485.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

2.运行App。配置完成后，一个包含样本数据的App会显示（如下），接下来只需连接数据源即可读取我们在LA中关联的PBI工作区的日志数据了。

![在这里插入图片描述](https://img-blog.csdnimg.cn/45679aad9dbb422fa16a6bf68afd50c6.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

Power BI App 本质上也是一份Import模式的PBI数据集，因此可以为其设置定时数据刷新 (如下)。此外REST API也可以读取到该App并对其执行GET/POST操作。

![在这里插入图片描述](https://img-blog.csdnimg.cn/8e42771d278348719bbada00f0229fa6.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

接下来就可以使用该App诊断PBI工作区的性能问题，你可以由大到小直接定位到影响性能的DAX查询，以便后续优化：

![在这里插入图片描述](https://img-blog.csdnimg.cn/85c9f078191148c8bdf55f3e223eae51.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_19,color_FFFFFF,t_70,g_se,x_16)

顺便一提，Power BI App目前不支持deployment pipeline, 且LA工作区与PBI工作区是一对一关系，因此测试环境与生产环境的Log Analytics App 需要分别单独安装配置。



***End~***


-----------------

**关注作者： [知乎](https://www.zhihu.com/people/zhang-zhe-hong-01/posts)   [Power BI官方社区](https://community.powerbi.com/t5/user/viewprofilepage/user-id/220984)**
