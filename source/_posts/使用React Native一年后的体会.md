---
title: 使用React Native一年后的感受
date: 2016-06-12
desc: React Native, 体会
from: https://discord.engineering/using-react-native-one-year-later-91fd5e949933
---

当我在面试[Discord](https://discordapp.com/)的时候，技术主管Stanislav跟我说：

> [React Native](https://facebook.github.io/react-native/)代表着未来。等它一发布，我们就会用它从零构建iOS应用。

作为一名原生iOS开发者，基于先前使用PhoneGap的经验，我非常怀疑使用Web技术构建移动应用的这种方式。但是当我学习并使用React Native一段时间之后，我非常庆幸我们做了这个决定。

<!-- more -->

## 开发效率

虽然我们的iOS“团队”只有我自己一个人，但是iOS应用开发依然可以赶上Web和桌面应用开发闪电般的速度。Apple公司已经允许开发者使用`JavaScriptCore`进行应用的升级，而无需等待App Store的审核流程。这对于那些缺乏专业的iOS QA（质量保障）团队的小公司来说是非常便利的，因为iOS团队可以在发布新功能之后进行热更新。

使用React Native一年之后，我们的iOS开发周期明显变快了，这得益于很高的开发效率。比如：

- 基于现有的前端架构，我们在两周之内就发布了V1.0的版本。

- 相比于`Auto Layout`，基于[Flexbox](https://css-tricks.com/snippets/css/a-guide-to-flexbox/)的样式可以节省一半的代码，并且更容易理解。

- 使用Flux设计模式，iOS和Web应用共享了`store`和`action`的98%的代码。

## 性能

React Native在后台线程运行JavaScript并发送极小的代码到主线程中。事实证明，React Native相比于Objective-C或Swift编写的原生iOS应用来说有一些性能差异！

![](/img/discord-perf.gif)

> Reactiflux小组的性能演示，该组有超过1.1万个会员 —— UI和JS线程大多数都是60FPS

然而，我们当初开始构建iOS应用时发现**聊天滚动视图**的性能并不令人满意，尤其是一些活跃的聊天分组。于是，我们决定使用[ComponentKit](http://componentkit.org/)构建聊天视图并编写必要的桥接代码代替原有的方案。当JS线程在完成一些繁重任务的时候，类库也无法提供原生那样流畅的动画（译注：之前动画是在JS线程执行，[目前有人提交了一份代码](https://github.com/facebook/react-native/commit/19e2388a76a7792ace166b64b9f1fc4695b62f1f)，有望使用原生iOS动画接口），因此我们在抽屉侧滑动画上继续使用[PopAnimation](https://github.com/facebook/pop)。

> 注： 作者称该应用仅聊天视图和抽屉动画是原生代码实现的，其他均由React Native实现。

当React Native Android版本发布时，我们也尝试在Android设备上运行应用，但遗憾的是，我们遇到了一些性能问题，只好暂时放弃。Android开发主管Miguel是这样说的：

<div class="tip">
很遗憾，不同Android设备的性能差异很大，这点明显落后于iOS。我们可以让应用运行地很快，但是性能——尤其是触摸事件，即使在更高端设备上也不能令人满意。并且在早期，由于React Native Android缺乏完善的功能，我们从产品原型过渡到成品应用比iOS花费了更多时间。
</div>

## 可用性

![](/img/discord-usability.png)

React Native让开发工作更简洁，使得开发者可以专注于每个新版本核心功能的开发。应用内自带的开发者菜单为我节省了大量的时间。

其中我最喜欢的一个功能是`Show Inspector`（审查工具），它可以即时展现交互视图的层级结构以及被选组件中所有必要的样式信息，这无疑是我用过的最棒的iOS审查工具。

## 社区

React Native项目每**两周**会发布一个新版本，其中包含一些新的特性以及修复的bug。这有利有弊，好比iOS几个月的稳定版本的发布，新的代码需要额外的时间进行升级，尤其是生产环境中的应用。因此，这也是到目前为止我们fork的React Native仓库只有四次主要升级的原因。

由于React Native还不太成熟，资源有限，也不完整。但随着它越来越流行，在不久之后一定能赶上其他成熟的技术。下面列出了一些实用的资源，我也经常在它的仓库上提问和获取最新的信息：

- Reactiflux上的[#react-native](https://discord.gg/0ZcbPKXt5bWVQmld)。

- [js.coach](https://js.coach/react-native)—React Native开源组件列表。

- [awesome-react-native](https://github.com/jondot/awesome-react-native)—大量的React Native文章、教程和示例。

> 译注：中文资源：[React Native学习指南](https://github.com/reactnativecn/react-native-guide)

总的来说，React Native很有潜力，它把我们团队的移动应用开发带上了一个新的水准。像我这样原生的iOS开发者可以很平滑地过渡到React Native，这有些出乎我的意料。同时，它也帮助我扩展职业技能，因为我也可以很轻松地向React编写的Web应用贡献代码了。


