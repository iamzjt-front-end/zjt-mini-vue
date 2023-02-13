<p align="center">
  <a href="https://github.com/vuejs/core">
    <img width="100" src="https://vuejs.org/images/logo.png" alt="Vue logo" />
  </a>
</p>

<p align="center">
  <a href="https://github.com/iamzjt-front-end">
    <img src="https://img.shields.io/badge/Github-iamzjt--front--end-blue" alt="IamZJT" />
  </a>&emsp;
  <a href="https://github.com/vuejs/core">
    <img src="https://img.shields.io/badge/-Vue.js-%232c3e50?style=flat-square&logo=vuedotjs" alt="Vue">
  </a>&emsp;
  <a href="https://github.com/iamzjt-front-end">
    <img src="https://komarev.com/ghpvc/?username=iamzjt-front-end&label=++visitorStatistics++&color=lightgrey" alt="visitorStatistics" />
  </a>&emsp;
</p>

<h1 align="center">
  zjt-mini-vue3
</h1>

<p align='center'>
  <a href='./README.md'>简体中文</a> | English
</p>


## Introduction

🙋 Hello!

This repository is `Vue3` source code learning record, to `TDD` as the driver, committed to the least code to achieve `happy path`, to learn to understand `Vue3` source code in depth.

The three modules of `Vue3`, `reactivity`, `runtime` and `compiler`, have been written by hand to achieve the minimalist `Vue3` model!

Why should I read the source code?

> ✨ Enhanced understanding of programming languages

> 🔥 Improve programming skills

> 🚀 Improving commissioning capabilities

> ⚡️ Improve code quality


## Series of articles

The following is a series of articles on the <a href="https://juejin.cn/column/7168612212133593095"><img src="https://img.shields.io/badge/juejin-juejin column-487DF8"></a> that are updated simultaneously.

👍 welcome all bosses to praise! 😂

#### 一、reactivity

[📑 01_Vue3源码的介绍](https://juejin.cn/post/7168664872547254285)

[📑 02_TDD开发环境搭建](https://juejin.cn/post/7169351734051995678)

[📑 03_01_实现effect&reactive&依赖收集&触发依赖](https://juejin.cn/post/7170480677614256158)

[📑 03_02_理解Proxy和Reflect](https://juejin.cn/post/7171655019425431583)

[📑 04_实现effect返回runner](https://juejin.cn/post/7172683900282634254)

[📑 05_实现effect的scheduler功能](https://juejin.cn/post/7173498493334454285)

[📑 06_实现effect的stop和onStop功能](https://juejin.cn/post/7174161779264585741)

[📑 07_实现readonly功能](https://juejin.cn/post/7175279305327378490)

[📑 08_实现isReactive和isReadonly](https://juejin.cn/post/7176086344815837242)

[📑 09_优化stop功能](https://juejin.cn/post/7179866542857781285)

[📑 10_实现reactive和readonly的嵌套对象转换功能](https://juejin.cn/post/7179867852877332517)

[📑 11_实现shallowReadonly和isProxy功能](https://juejin.cn/post/7180887790899920956)

[📑 12_实现ref功能](https://juejin.cn/post/7181710097863671864)

[📑 13_实现isRef和unRef功能](https://juejin.cn/post/7182379390183931960)

[📑 14_实现proxyRefs功能](https://juejin.cn/post/7185443608827265061)

[📑 15_实现computed计算属性](https://juejin.cn/post/7189847454152392760)

[📑 16_实现相对完善的reactive](https://juejin.cn/post/7194275202212036667)

[📑 17_实现相对完善的effect](https://juejin.cn/post/7196690584286462008)


#### 二、runtime-core

[📃 01_实现初始化component流程]()


#### 三、compiler

[📰 01_编译模块概述.md]()

✍️ To Be Continue...


## Currently implemented function points

### reactivity

- [x] 实现 reactive
- [x] 实现 ref
- [x] 实现 readonly
- [x] 实现 嵌套reactive和readonly
- [x] 实现 computed
- [x] 实现 effect 并支持分支切换、嵌套及++情况下的依赖收集
- [x] 实现 track 依赖收集
- [x] 实现 trigger 触发依赖
- [x] 实现 effect.scheduler 调度器
- [x] 实现 基于runner、effect.stop 和 onStop的响应式开启关闭控制
- [x] 实现 isReactive和isReadonly
- [x] 实现 isProxy
- [x] 实现 isRef和unRef
- [x] 实现 shallowReadonly
- [x] 实现 proxyRefs


### runtime-core

- [x] 支持 component 类型
- [x] 支持 element 类型
- [x] 支持 Text 类型节点
- [x] 初始化 props
- [x] setup 可获取 props 和 context
- [x] 支持 $el、$slots等api
- [x] 支持 component emit
- [x] 支持 单节点、多节点slots
- [x] 可以在 render 函数中获取 setup 返回的对象
- [x] nextTick 的实现
- [x] 支持 proxy
- [x] 支持 getCurrentInstance
- [x] 支持 provide/inject
- [x] 支持 watchEffect

### compiler-core
- [x] 解析插值
- [x] 解析 element
- [x] 解析 text

### runtime-dom
- [x] 支持 custom renderer 

### runtime-test
- [x] 支持 测试 runtime-core 的逻辑

### infrastructure
- [x] support monorepo with pnpm


## Corrigendum

If you find an error, you can make a corrigendum to the corresponding issues or contact me on <a href="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/IamZJT-WeChat.jpg">WeChat：IamZJT_</a>.

If you like or inspire, welcome star, which is also an encouragement to the author.
Of course, You can also add my WeChat to communicate and grow together.


## Licence

All content is licensed by [MIT](https://spdx.org/licenses/MIT).
