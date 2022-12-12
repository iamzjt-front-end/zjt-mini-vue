# 02_TDD开发环境搭建

从现在开始，正式的进行 `mini-vue`的开发。

首先搭建 `TDD`（测试驱动开发）开发环境，本项目采用 `jest`作为单元测试框架，采用 `typescript`作为主要开发语言，采用 `yarn`作为包管理工具。

> 这里也浅浅说一下`TDD`的好处：
>
> 回顾过往：
>
> 1. 需求分析，想不清楚细节，管他呢，先开始写，一把梭哈。
>
> 2. 发现需求细节不明确，再去跟业务人员确认，讨论，好的情况下改一小点，不好的情况下，现有代码设计推翻，完全重构。
>
> 3. 然后再确认好几次终于写完所有逻辑。
>
> 4. 运行起来测试一下，啊，果然不工作呢，调试。
>
> 5. 调试好久终于工作了。
>
> 6. 转测试，`QA` 测出 `bug`，`debug`， 打补丁。
>
> 7. 终于，代码可以工作了。
>
> 8. 没有规划，还步子迈得太大，就导致方法拆分的不够细，所以就导致了很多过程式的代码。
>
> 9. 一看代码烂的像坨💩，不敢动，动了还得手工测试，还得让 QA 测试，还得加班...
>
> 10. 兴致不好，项目紧急，就随它去了，反正祖传代码。偶尔有时间，突发兴致来了，重构一下。结果重构后要么懒得测试，要么测试不彻底，然后经常引发出其他隐性bug。

> 是不是你的现状？😅 想要打破现状吗？那就加入`剃刀党`吧。`jest`、`vitest`、`cypress`任你挑选。
>
> 不好意思，偏题了，回归正题。
>
> `TDD`的全称是`测试驱动开发`
> ，是一种旨在提升代码质量的开发实践。这种开发实践的主要步骤是在编写产品代码之前，先编写单元测试代码，然后再由测试代码来决定写什么产品代码，其目的是取得快速反馈，并使用`illustrate the main line`
> 方法来构建程序。
>
> 测试驱动开发是戴两顶帽子思考的开发方式：
>
> 先戴上`实现功能`的帽子，在测试的辅助下，快速实现其功能；
>
> 再戴上`重构`的帽子，在测试的保护下，通过去除冗余的代码，提高代码质量。
>
> 测试驱动着整个开发过程：首先，驱动代码的设计和功能的实现；其后，驱动代码的再设计和重构。

> 那`TDD`的开发编码方式又是什么样的呢？
>
> 1. 首先进行任务拆分，分离关注点；
>
> 2. 列 `Example`，用实例化需求，澄清需求细节；
>
> 3. 写测试断言，只关注需求，程序的输入输出，不关心中间过程；
>
> 4. 写实现，不考虑别的需求，用最简单的方式满足当前这个小需求即可，即：`happy path`；
>
> 5. 重构，用手法消除代码里的坏味道；
>
> 6. 写完，手动测试一下，基本没什么问题，有问题补个用例，修复；
>
> 7. 转测试，小问题，补用例，修复；
>
> 8. 代码整洁且用例齐全，信心满满地提交；
>
> 不禁想说一句，妙啊...😌

### 一、环境搭建

1. 初始化项目

   ```shell
   # 生成默认的package.json
   yarn init -y
   ```

2. 安装和配置 `typescript`

   ```shell
   # 安装typescript
   yarn add typescript -D
   # 生成配置文件tsconfig.json
   npx tsc --init
   ```

3. 引入 `jest`

   ```shell
   yarn add jest @types/jest -D
   ```

4. 配置 `tsconfig.json`（要包含的类型声明文件名列表，也就是上面安装的@types/jest）

   ```json
   {
      "types": ["jest"]  
   }
   ```

### 二、定义一下目录结构

```
zjt-mini-vue3  
│  
├── docs // 笔记  
│ ├── images  
│ └── md  
├── node_modules  
├── examples // 案例  
│ ├── rollup-vue // sfc打包体验  
│ └── testCase // 测试用例  
├── src  
│ └── reactivity  
│ ├── tests  
│ │ └── index.spec.ts // jest单测  
│ └── index.ts  
├── .gitignore  
├── babel.config.js  
├── LICENSE  
├── package.json  
├── README.md  
├── tsconfig.json  
└── yarn.lock
```

1. index.ts编写一个方法用于测试

   ```ts
   export function add(a, b) {
      return a + b;
   }
   ```

2. index.spec.ts编写第一个测试用例

   ```ts
   import { add } from '../index';

   it('init', function () {
      expect(add(1, 1)).toBe(2);
   });
   ```

3. 配置package.json的script

   ```json
   {
      "scripts": {
         "test": "jest"
      }
   }
   ```

### 三、测试

   ```shell
   yarn test
   ```

<img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e276501adc854b518522fdedb6221552~tplv-k3u1fbpfcp-zoom-1.image" width="666" alt="02_jest第一个测试用例测四结果"/>

### ps

这是一个 [早起俱乐部](https://juejin.cn/pin/7173512204149325861)！

⭐️ 适合人群：所有想有所改变的人，可以先从早起半小时开始！抽出30分钟，从初心开始！！  
⭐️ 没有任何其它意味，只是本人想寻找一起早起、志同道合的小伙伴。