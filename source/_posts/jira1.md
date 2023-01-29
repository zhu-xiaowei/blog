---
title: 研发管理（一）从现状到Jira
date: 2022-03-07 16:57:00
tags: Jira

---

![](https://img.carlwe.com/xs/jira_flow.webp)

本系列文章主要是围绕Jira来介绍研发过程管理，文章会以实际使用目的为主线，以实际页面和功能为切入点介绍其背后的实现逻辑，大概分为以下四篇文章：

<!--more-->

* [**研发管理（一）从现状到Jira**](/2022/03/07/jira1)
* [研发管理（二）Jira实现基本功能](/2022/05/06/jira2)
* [研发管理（三）从实际出发优化Jira使用](/2022/05/09/jira3)
* [研发管理（四）可用报表及项目管理](/2022/06/29/jira4)

### 背景

开始还得从提升研发效能说起，所在公司的业务部门经常会觉得研发效率低，提出的需求往往过一两个月还不能上线。同时研发内部需要沟通和协调的人员及分工也比较多，例如产品、开发、UI、测试。每个环节都需要更好的衔接才能集体高效的产出，刚开始我们通过所有任务都上墙来让研发任务更加直观。

#### 现有模式介绍

![](https://img.carlwe.com/xs/minjie_kanban.jpg)

**大卡片**：在一个版本的迭代中我们会有很多用户故事，每个用户故事我们会写到一个大的卡片上，标注好这个大卡片的提测时间，放在左侧栏，

**小卡片**：每个用户故事里面会有前端、后端、测试相关的开发工作，每个人会写上自己的小纸条，写明开发内容，姓名，开发时间。

**泳道**：分为代办，前端开发，后端开发，集成，测试和验收这几部分。同时里面又细分为正在做和完成的区别，

**流程**：前后端开发完后，讲自己的卡片挪动到开发完成，发起联调的一方会将开发完成的卡片挪动到集成中，进入集成，集成完之后挪动到集成完成，测试同学只关心集成完成的这一栏，由测试拖动到测试中，完成测试后移入测试完成，并交给产品验收，当大卡片中所有的任务都完成后，我们会放到Accepted一栏代表已经验收完成。

**总结回顾**：一个版本我们预定为2周时间，每两周的版本完成后，我们会对这两周里面的需求、前后端开发、测试及上线问题组织所有相关同学进行敏捷回顾，在敏捷回顾会议上我们会列举出做的好的、不好的以及疑惑的地方，最后总结出下一个版本的todo list进行改进。

![](https://img.carlwe.com/xs/scrum_review_small.jpg)

#### 遇到的困难

经过上述六七个版本的迭代之后，整个团队对于这套模式已比较熟悉，基本流程已经建立，但是但这种方式的缺点很明显，参与人过多时拖动不及时、任务多了不好放，历史记录不可追踪，统计困难等。

#### 我们的诉求

那么为了让大家的工作都更加可视化，可追踪，可量化，统计各个环节的等待，更好的实现上下游拉动式开发及问题解决，让每一步进展都实时在线，因此我们需要一个在线化的工具从线下的敏捷管理同步到线上实现以上的诉求，是时候介绍下Jira了。

### 初识Jira

为了解决上述问题，将线下的敏捷开发流程移动到线上，我们选择了Jira。

#### 为什么选择Jira

> 1. 大公司开发业界标杆
> 2. 文档丰富社区成熟
> 3. 项目管理、流程配置能做到可定制化
> 4. 自定义过滤器、可定制的工作流、丰富的插件
> 5. 团队有基础的同学较多，上手快
> 6. 针对敏捷开发设计的Scrum看板

#### Jira简介

[Jira](https://www.atlassian.com/software/jira)是Atlassian公司出品的项目与事务跟踪工具，被广泛应用于缺陷跟踪、客户服务、需求收集、流程审批、任务跟踪、项目跟踪和敏捷管理等工作领域。Jira中配置灵活、功能全面、部署简单、扩展丰富，其超过150项特性得到了全球115个国家超过19,000家客户的认可。以上是百度中给出的介绍，让我们看看官网的介绍：

> **敏捷团队的 首选软件开发工具，最优秀的软件团队交付频率高且速度快。Jira Software 专为软件团队中的每位成员构建，可用于规划、跟踪和发布卓越的软件。**

可见Jira的实力以及在业界的认可度还是相当不错的，而且能够提供丰富的功能使得我们能够更好更灵活的使用。

#### Jira主要功能

**1.敏捷看板**

![](https://img.carlwe.com/xs/jira_scrum.png)

> 敏捷看板适合有版本节奏的开发项目，利用可自定义的 Scrum 板，敏捷团队可集中精力尽可能迅速地交付迭代和增量价值。

**2.普通看板**

![](https://img.carlwe.com/xs/jira_kanban.png)

> 普通看板适用于没有固定版本节奏的项目。借助灵活的看板图，团队可以全面了解后续工作，从而让您可以在最短的加工时间内交付最大的输出。

**3.线路图Roadmap**

![](https://img.carlwe.com/xs/jira_roadmap.png)

> 线路图可以更好的来规划中长期多个项目之间的开发及资源占用整体情况。与利益相关者沟通计划事宜，并确保路线图与团队的工作相关联，所有这些任务只需在 Jira Road Map 中点击几下即可完成。

**4.Jira报告**

![](https://img.carlwe.com/xs/jira_report.png)

> 利用Jira报告我们可以很方便的进行当前项目进度信息的查看，例如燃耗图、累计流量图、控制图、版本报告等。

**5.缺陷管理和代码关联**

除了上述功能外，Jira通常被大家用来做缺陷管理。其内部有默认的缺陷管理流程，开箱即用。同时看板的任务和代码提交记录可以很好的进行关联。

### 总结

有了对Jira的初步了解，我们就可以着手开始利用Jira来配置我们现有的流程了，并在现有流程上利用Jira一些好用的功能，发现问题并提升现有流程的效率，下一篇文章，我们会利用Jira来模拟搭建现有的流程，将现有的线下流程搬到线上来运行。