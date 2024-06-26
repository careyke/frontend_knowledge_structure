# 微前端（MicroFrontEnd） - 分而治之

最近两年微前端的概念比较热门，笔者对这个领域也非常感兴趣。查找了很多资料，看了很多圈内大佬对于微前端的分析，这里笔者站在巨人的肩膀上来总结一下自己对于微前端的理解。

首先我们来聊一下对于微前端的定义。

> Micro frontends, An architectural style where independently deliverable frontend applications are composed into a greater whole.

即，一种由独立交付的多个前端应用组成更大整体的**架构**。具体的，**将前端应该分解成一个个独立开发、测试、部署的部件，但是用户看起来仍然是内聚的单个产品。**

微前端的概念借助了后端中微服务的思想，可以理解为是”前端领域的微服务“。

**微前端是一种前端架构，为并非是框架**



我们从以下几个方面来分析微前端架构：

1. Why - 为什么需要微前端
2. What - 什么是微前端
3. How - 如何实现微前端



## 1. ”巨石“应用 — 为什么需要微前端

随着企业产品的发展和迭代，对应的中后台管理系统的复杂度也会越来越大，而且中后台应用的生命周期往往比较长，在功能的不断增加和迭代中，往往会涉及到多个团队参与开发，使得应用维护的难度越来越大，最终演变成一个”巨石应用“。

根据业务场景的不同，对应的巨石应用有这个的痛点，这里我们分析一下**巨石应用通用的痛点**：

1. 仓库代码多，构建效率低
2. 多个团队参与，代码风格不一致，新人难以快速上手
3. 功能模块之间边界模糊，耦合严重
4. 技术升级困难，牵一发动全身



微前端架构的提出就是用来解决这些问题。



## 2. 什么是微前端

### 2.1 微前端的特点

微前端的定义我们前面有提到过，下面我们来分析一下微前端的特点：
1. **独立性**：子应用仓库相互独立，独立开发、测试和部署
2. **技术栈无关**：子应用可以使用任何技术栈，不需要受其他应用的约束
3. **运行时隔离**：每个子应用之间状态隔离，运行时状态不共享



### 2.2 微前端架构的结构

微前端架构的应用在结构上分成两个部分：
1. **主应用** — 基座
   - 渲染公共的页面
   - 提供公共的数据，解决横切关注点。比如登录信息
   - 为子应用提供容器，根据条件调度子应用的渲染
2. **子应用** — 完全独立的前端应用，配置对应的规则即可接入到主应用中



那么子应用使用什么方式来集成到主应用中呢？大致有一下三种方式：

1. 服务端集成
2. 构建时集成
3. 运行时集成

前面两种方式都不推荐，现在主流的微前端框架都是使用**运行时集成**这种方案来实现的。



## 3. 如何实现微前端

目前大家常提及的三种微前端实现方案：

1. iframe
2. JS
3. Web Components

我们来分别聊一聊每种方案的优劣



### 3.1 iframe

iframe作为一个天生隔离的容器，非常适合用来渲染子应用。有以下**优势**：

1. 技术栈无关
2. 独立沙箱，可以实现运行环境的完全隔离
3. 使用起来非常简单

但是它的**劣势**同样也是非常明显的：

1. 样式体验差
2. 应用之间通信比较困难
3. 路由丢失：子应用的路由无法同步到主应用，随着页面刷新，会导致子应用的状态丢失
4. 加载时间长

其中最难以忍受的问题就是样式问题，往往让开发者焦头烂额，所以这种方案现在基本被抛弃了。
> eg: 腾讯开源的微前端框架 [wujie](https://github.com/Tencent/wujie) 使用的就是 iframe，可以学习一下。



### 3.2 JS方案

顾名思义，就是使用现代的`js`来处理，运行时动态加载子应用的资源并渲染子应用。目前业界比较成熟的框架有`Single-SPA`和`qiankun`，其中`qiankun`是基于`Single-SPA`来研发的。

这类方案的优势：

1. 技术栈无关
2. 没有样式问题
3. 路由统一管理
4. 应用之间通信简单

劣势：

1. 没有运行时沙箱
2. 有一定的接入成本

其中`qiankun`这个框架中实现了子应用的运行时沙箱，而且提供的API也相当简单，是笔者目前认为最好用的微前端框架。后续笔者会结合`qiankun`的源码来分析其中的实现原理

> TODO: 需要找时间对比其他去中心化微前端框架，比如[EMP](https://github.com/efoxTeam/emp)



### 3.3 Web Components

这种方案是微前端未来**值得期待**的方案，因为目前`Web Components`还没有广泛使用，浏览器兼容性比较差。

> 笔者对于Web Component了解得比较少，后续再补上



## 4. 关于微前端的讨论

以下是一些国内大佬对于微前端的看法和讨论，也十分推荐大家看看：

1. [可能是你见过最完善的微前端解决方案](https://juejin.cn/post/6844903917029965831#heading-2)

2. [微前端的核心价值](https://juejin.cn/post/6844904013700268046)
3. [探索微前端的场景极限](https://juejin.cn/post/6937123749619728421)
4. [微前端到底是什么？](https://zhuanlan.zhihu.com/p/96464401)

5. [《什么是微前端》](https://github.com/efoxTeam/emp/wiki/%E3%80%8A%E4%BB%80%E4%B9%88%E6%98%AF%E5%BE%AE%E5%89%8D%E7%AB%AF%E3%80%8B) - 一个EMP的微前端框架，和`qiankun`的实现不一样

