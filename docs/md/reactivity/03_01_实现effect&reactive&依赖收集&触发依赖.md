# 03_01_实现 effect & reactive & 依赖收集 & 触发依赖

### 一、reactivity happy path

首先我们知道`reactivity`的`happy path`（核心逻辑）就是: 通过`reactive`定义响应式变量，然后通过`effect`
去收集响应式变量的依赖，然后实现依赖的自动收集和自动触发。

那我们先来编写第一个测试案例，通过单测来带大家看一看功能需求。

首先删掉之前的`index.spec.ts`，建立`effect.spec.ts`，实现`reactivity`的`happy path`。

```ts
describe('effect', function () {
  it.skip('happy path', function () {
    // * 首先定义一个响应式对象
    const user = reactive({
      age: 10
    })

    // * get -> 收集依赖
    let nextAge;
    effect(() => {
      nextAge = user.age + 1;
    })

    // * effect默认会执行一次
    expect(nextAge).toBe(11);

    // * set -> 触发依赖
    user.age++;
    expect(nextAge).toBe(12);
  });
});
```

那么`reactivity`的`happy path`的单测书写完毕，因为核心逻辑的单测需要依赖于`reactive`和`effect`
两个api，所以此处 **it.skip**，先跳过这个测试用例，我们先来实现`reactive`。

---------------------------------------------------------------------------------------

### 二、reactive happy path

那在实现`reactive`之前，依旧先来写`reactive`核心逻辑的单测。

```ts
describe('reactive', function () {
  it('happy path', function () {
    const original = { foo: 1 };
    const observed = reactive(original);

    expect(observed).not.toBe(original);
    expect(observed.foo).toBe(original.foo);
  });
});
```

这个单测需要我们实现的功能主要有两点:

1. 响应式对象`observed`和原始对象`original`不全等；
2. `observed`也能取到`foo`属性的值，并且与`original`一致。

那带着这些需求，我们接着建立`reactive.ts`，来实现一下`reactive`的这个最核心的逻辑。

```ts
export function reactive(raw) {
  return new Proxy(raw, {
    // 此处使用proxy报错的话，需要进tsconfig.json中，配置"lib": ["DOM", "ES6"]。
    get(target, key) {
      const res = Reflect.get(target, key);

      // todo 依赖收集
      return res;
    },

    set(target, key, value) {
      const res = Reflect.set(target, key, value);

      // todo 触发依赖
      return res;
    }
  })
}
```

至此，`reactive`的`happy path`实现完毕，至于如何进行`依赖收集`和`触发依赖`，我们放到后面再去慢慢考虑。

那现在，先来看一下单测有没有通过。

```shell
yarn test reactive
```

<img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/83aa40ccb1ce427c9ff5fd3482d17274~tplv-k3u1fbpfcp-zoom-1.image" width="666" alt="03_01_reactive核心逻辑单测"/>

测试通过，那么接下来，我们继续完善`reactive`的逻辑代码。

接着，再去`reactive.spec.ts`和`effect.spec.ts`中引入`reactive`。

```ts
import { reactive } from '../reactive';
```

**ps:** 至于上述代码中采用`Reflect.get`而不是`target[key]`返回属性值，将在下一篇文章中详细做出阐述。

---------------------------------------------------------------------------------------

### 三、effect happy path

那只要再把`effect`完善，那`reactivity`的`happy path`的单测就不会报错了。

那么，现在，咱就去完善`effect`。建立`effect.ts`文件，并完善基础逻辑。

```ts
class ReactiveEffect {
  private _fn: any;

  constructor(fn) {
    this._fn = fn;
  }

  run() {
    this._fn();
  }
}

export function effect(fn) {
  const _effect = new ReactiveEffect(fn);

  _effect.run();
}
```

可以注意到，此处我们封装了`ReactiveEffect`类。是因为我们需要对传进来的依赖进行不同的操作，并且以后还会扩展出更多的功能，所以将其封装也是为了日后更好地维护。

> 此处多说两句：
>
> 封装概念通常由两部分组成，`封装数据`和`封装实现`，也就是：
>
> 1. 相关的数据（用于存储属性）
> 2. 基于这些数据所能做的事（所能调用的方法）；
>
> 封装的目的是将信息隐藏，即属性与方法的可见性。
>
> 从《设计模式》的角度出发，封装在更重要的层面体现为 **封装变化** 。
>
> 通过 **封装变化** 的方式，把系统中 **稳定不变的部分** 和 **容易变化的部分**
> 隔离开来。在系统的演变过程中，我们只需要替换那些容易变化的部分，如果这些部分是已经封装好的，替换起来也相对容易。这可以最大程度地保证程序的稳定性和可扩展性。

继续回归正题。💬

此时，去掉`effect` `happy path`中`it`的`skip`，然后注释掉`set -> 触发依赖`后的两行，先不看`update`的过程，运行一下测试。

```shell
yarn test
```

<img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3753e37ad2814fa28e7e76f1c51b33df~tplv-k3u1fbpfcp-zoom-1.image" width="666" alt="03_02_effect部分逻辑单测"/>

---------------------------------------------------------------------------------------

### 四、重中之重 依赖收集和触发依赖

那现在的难点就来了，如何让`user.age++`的时候，`nextAge`也自动更新。  
这其实就已经到了响应式系统的核心逻辑了，也就是 **`依赖收集`** 和 **`触发依赖`**，也就是 **`track`** 和 **`trigger`** 的实现。

#### 1. 依赖收集 track

回到`reactive.ts`，`Proxy`的`get`方法中增加`track`，当某个属性被读取时，进行依赖的收集。

> 我们这里所说的依赖，也叫副作用函数，即产生副作用的函数。也就是说，effect函数的执行，会直接或间接影响其它函数的执行，这时我们说effect函数产生了副作用。

```ts
const res = Reflect.get(target, key);

track(target, key);
return res;
```

由于依赖是被effect包裹的，所以就在`effect.ts`中来写`track`的逻辑。

```ts
export function track(target, key) {
  // * target -> key -> dep
}
```

从上可以看出，依赖对应的结构🌲应该如下：

<img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f374a78888b847f9b7f317c3cde154a3~tplv-k3u1fbpfcp-zoom-1.image" width="388" alt="03_03_依赖树形关系"/>

- target：被操作（读取）的代理对象（ **user** ）；
- key：被操作（读取）的字段名（ **age** ）；
- dep：用effect函数注册的副作用函数（ **() => { nextAge = user.age + 1; }** ），也就是我们要收集的依赖。

那这里分别使用`WeakMap`、`Map`、`Set`来存储。

- `WeakMap`由 **target -> Map** 构成；
- `Map`由 **key -> Set** 构成。

其中`WeakMap`的键是原始对象`target`，值是一个`Map`实例；  
而`Map`的键是原始对象`target`的`key`，值是一个由副作用函数组成的`Set`。

<img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5af3f9e840044aee98e4cc01b72a2dfc~tplv-k3u1fbpfcp-zoom-1.image" width="666" alt="03_04_WeakMap、Map和Set之间的关系"/>

搞清楚他们的关系之后，有必要解释一下这里为什么用`WeakMap`，这就涉及到了`WeakMap`和`Map`的区别。

> 简单地说，`WeakMap`对`key`是弱引用，不影响垃圾回收器的工作。但如果使用`Map`来代替`WeakMap`，那么即使用户侧的代码对`target`
> 没有任何引用，这个`target`也不会被回收，最终可能导致内存溢出。

理论完毕，接下来开始完善`track`的逻辑。

```ts
// * target -> key -> dep
export function track(target, key) {
  // * depsMap: key -> dep
  let depsMap = targetMap.get(target);
  if (!depsMap) {
    targetMap.set(target, (depsMap = new Map()));
  }

  // * dep
  let dep = depsMap.get(key);
  if (!dep) {
    depsMap.set(key, (dep = new Set()));
  }

  dep.add(activeEffect); // activeEffect是当前依赖，完整代码最后挂出
}
```

#### 2. 触发依赖 trigger

再次回到`reactive.ts`，`Proxy`的`set`方法中增加`trigger`，当某个属性被读取时，进行依赖的收集。

```ts
const res = Reflect.get(target, key);

track(target, key);
return res;
```

再接着完善`trigger`的逻辑，取出所有的依赖，依次执行。

```ts
export function trigger(target, key) {
  let depsMap = targetMap.get(target);
  let dep = depsMap.get(key);

  for (const effect of dep) {
    effect.run();
  }
}
```

感觉好像就这样，没啥问题了，那继续跑一下单测吧，看看是不是真的没问题了。

<img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f60ce586768645fd8950a5b90a47cf21~tplv-k3u1fbpfcp-zoom-1.image" width="666" alt="03_05_effect、reactive全流程单测"/>

可以，全部通过，美滋滋啊~ 🍉

那么至此，我们就实现了 **`effect`** & **`reactive`** & **`依赖收集`** & **`触发依赖`** 的`happy path`。

- - -

### end

当然这只是最简形态的`reactive`，就比如: 分支切换（三元表达式）、嵌套effect、++的情况，我们都完全还没有考虑进去，后续在`reactivity`
模块的末尾，我们将来完善对这些情况的处理。

### ps

这是一个 [早起俱乐部](https://juejin.cn/pin/7173512204149325861)！

⭐️ 适合人群：所有想有所改变的人，可以先从早起半小时开始！抽出30分钟，从初心开始！！  
⭐️ 没有任何其它意味，只是本人想寻找一起早起、志同道合的小伙伴。