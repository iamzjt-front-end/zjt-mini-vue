# 17_实现相对完善的effect

### 一、我们要优化哪些点？

在上篇[实现相对完善的reactive](https://github.com/iamzjt-front-end/zjt-mini-vue3/blob/main/docs/md/reactivity/16_实现相对完善的reactive.md)
后，那我们继续来实现相对完善的`effect`。

首先我们还是列举一些对`effect`的简单考虑：

- `分支切换`，也就是不同条件执行不同代码，例如：三元表达式。
- 嵌套`effect`的情况，应该如何处理？
- `prop++`的情况，既读又取导致无限递归，栈溢出的情况。

### 二、effect相关考虑完善

#### （一）分支切换问题

##### 1. 单测用例

```ts
it('should discover new branches while running automatically', () => {
  let dummy;
  const obj = reactive({ prop: 'value', run: false });

  const conditionalSpy = jest.fn(() => {
    dummy = obj.run ? obj.prop : 'other';
  });
  effect(conditionalSpy);

  expect(dummy).toBe('other');
  expect(conditionalSpy).toHaveBeenCalledTimes(1);

  obj.prop = 'Hi';
  expect(dummy).toBe('other');
  expect(conditionalSpy).toHaveBeenCalledTimes(1);

  obj.run = true;
  expect(dummy).toBe('Hi');
  expect(conditionalSpy).toHaveBeenCalledTimes(2);

  obj.prop = 'World';
  expect(dummy).toBe('World');
  expect(conditionalSpy).toHaveBeenCalledTimes(3);
});
```

```ts
it('should not be triggered by mutating a property, which is used in an inactive branch', () => {
  let dummy;
  const obj = reactive({ prop: 'value', run: true });

  const conditionalSpy = jest.fn(() => {
    dummy = obj.run ? obj.prop : 'other';
  });
  effect(conditionalSpy);

  expect(dummy).toBe('value');
  expect(conditionalSpy).toHaveBeenCalledTimes(1);

  obj.run = false;
  expect(dummy).toBe('other');
  expect(conditionalSpy).toHaveBeenCalledTimes(2);

  obj.prop = 'value2';
  expect(dummy).toBe('other');
  expect(conditionalSpy).toHaveBeenCalledTimes(2);
});
```

##### 2. 完善逻辑

###### 2.1. 前置概念

首先，我们需要明确的一点是：什么是分支切换？

在上述单测中，可以看到，`conditionalSpy`中存在一个三元表达式，根据`obj.run`的值不同，会执行不同的代码分支。并且当`obj.run`
的值发生变化时，分支也会随之变更，这就是所谓的分支切换。

根据某个响应式对象值的变化，可能会增加或减少“活跃”响应式对象。  
增加倒是还好，`get`操作会触发`track`进行收集起来；  
减少的话，我们似乎目前并没有进行处理。那就意味着，会存在冗余依赖，那再次`trigger`的时候，也就会触发不必要地更新。

###### 2.2. 通过第一个单测

下面我们分步来具体讲解一下。

在第一段单测中，运行的过程中会发现报错了，报错信息如下：

<img src="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/202212130528482.png" width="999" alt="17_01_第一段单测报错信息"/>

然后在点击进入到报错位置，打上断点，开始调试。

<img src="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/202212130544132.png" width="999" alt="17_02_第一段单测调试"/>

发现`dep`的值是`undefined`，而下面还接着去遍历了`undefined`，所以报错了。  
然后再看`depsMap`中，只有`run`属性的依赖。这个很容易理解，因为`effect`首次运行时，只读取了`run`的值，自然就只有`run`被收集起来。

那完善思路就很明确了，只要`dep`存在，才继续往下运行。

```ts
// src/reactivity/effect.ts

export function trigger(target, key) {
  let depsMap = targetMap.get(target);
  // + 考虑到depsMap可能也会没有，这里我们也加上
  if (!depsMap) return;

  let dep = depsMap.get(key);
  // + 如果没有dep，则直接终止运行
  if (!dep) return;

  triggerEffects(dep);
}
```

再跑一遍第一段单测。

<img src="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/202212130605186.png" width="999" alt="17_07_第一段测试通过"/>

可以看到测试已通过。

###### 2.3. 通过第二个单测

在第二段单测中，初始的依赖对应关系如下：

<img src="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/202212060614734.png" width="388" alt="17_03_分支切换前依赖对应关系"/>

当`obj.run`的值变成`false`时，分支随之切换，再次对应的依赖关系应该如下图：

<img src="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/202212060618463.png" width="388" alt="17_04_分支切换后依赖对应关系"/>

而当我们另外在调试过程中，却发现：

<img src="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/202212060700376.png" width="999" alt="17_05_run变化调试"/>

当`obj.run`变成`false`时，`targetMap`中依旧对应的是`run`和`prop`的依赖。

<img src="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/202212060702642.png" width="999" alt="17_06_props变化调试"/> 

继续往下走，当`obj.prop`变化时，也触发了`trigger`，并且取到了依赖，触发更新。

但这并不是我们所期望的，一方面，我们希望的是当分支不活跃时，理应冗余依赖应从`targetMap`中删除；  
另一方面，就算不活跃分支中的响应式对象发生变化，也不需要去进行这种不必要地更新，因为无论更不更新都不会影响程序运行的结果且浪费性能。

那实现思路就很清晰了，我们只需要在每次收集依赖前将依赖全部清空，然后再重新收集即可。

```ts
// src/reactivity/effect.ts

export class ReactiveEffect {
  // ... 省略部分代码

  run() {
    // 已经被stop，那就直接返回结果
    if (!this.active) {
      return this._fn();
    }
    // 未stop，继续往下走
    // 此时应该被收集依赖，可以给activeEffect赋值，去运行原始依赖
    shouldTrack = true;
    // + 清空依赖
    cleanupEffect(this);
    activeEffect = this;
    const result = this._fn();
    // 由于运行原始依赖的时候，会触发代理对象的get操作，会重复进行依赖收集，所以调用完以后就关上开关，不允许再次收集依赖
    shouldTrack = false;

    return result;
  }

  // ... 省略部分代码
}
```

看上去好像结束了，就这么一行代码。  
但如果你尝试运行单测，会发现目前的实现会导致无限循环执行。

<img src="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/202212160628264.png" width="666" alt="17_08_第二段单测无限循环"/>

原因在哪呢？

问题出在`triggerEffects`的`for of`循环中:  
这里会遍历执行`run`，而`run`运行中会`cleanupEffect(this)`清空所有依赖;
然后重新运行`this._fn()`原始依赖时，会继续进行依赖的收集，会重新添加到`dep`中。

就相当于下面这段代码：

```ts
const set = new Set([1]);

set.forEach(item => {
  set.delete(1);
  set.add(1);
  console.log('遍历中');
})
```

在浏览器中运行，会发现无限执行下去，内存暴增，最后卡死。

> 语言规范中对此有明确的说明：在调用`forEach`遍历`Set`
> 集合时，如果一个值已经被访问过了，但该值被删除并重新添加到集合，如果此时`forEach`遍历没有结束，那么该值会重新被访问。

因此，上面的代码会无限执行。  
解决办法很简单，那就是构造另外一个`Set`集合并遍历它，或者拓展成数组进行遍历。  
遍历的是新`Set`，而增删操作的是旧的`Set`，并不会造成什么影响。

```ts
const set = new Set([1]);
const newSet = new Set(set);
// const newArr = [...set];

newSet.forEach(item => {
  set.delete(1);
  set.add(1);
  console.log('遍历中');
})
```

那我们此处也采用同样的思路去解决。

```ts
export function triggerEffects(dep) {
  // + 重新构建一个新的 Set
  const effects = new Set<any>(dep);

  for (const effect of effects) {
    if (effect.scheduler) {
      // ps: effect._fn 为了让scheduler能拿到原始依赖
      effect.scheduler(effect._fn);
    } else {
      effect.run();
    }
  }
}
```

<img src="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/202212160632767.png" width="999" alt="17_09_第二段单测通过截图"/>

#### （二）嵌套effect问题

##### 1. 单测用例

```js
it('should allow nested effects', () => {
  const nums = reactive({ num1: 0, num2: 1, num3: 2 });
  const dummy: any = {};

  const childSpy = jest.fn(() => (dummy.num1 = nums.num1));
  const childEffect = effect(childSpy);
  const parentSpy = jest.fn(() => {
    dummy.num2 = nums.num2;
    childEffect();
    dummy.num3 = nums.num3;
  });
  effect(parentSpy);

  expect(dummy).toEqual({ num1: 0, num2: 1, num3: 2 });
  expect(parentSpy).toHaveBeenCalledTimes(1);
  expect(childSpy).toHaveBeenCalledTimes(2);

  // * 应该只触发childEffect
  nums.num1 = 4;
  expect(dummy).toEqual({ num1: 4, num2: 1, num3: 2 });
  expect(parentSpy).toHaveBeenCalledTimes(1);
  expect(childSpy).toHaveBeenCalledTimes(3);

  // * 触发parentEffect，触发一次childEffect
  nums.num2 = 10;
  expect(dummy).toEqual({ num1: 4, num2: 10, num3: 2 });
  expect(parentSpy).toHaveBeenCalledTimes(2);
  expect(childSpy).toHaveBeenCalledTimes(4);

  // * 触发parentEffect，触发一次childEffect
  nums.num3 = 7;
  expect(dummy).toEqual({ num1: 4, num2: 10, num3: 7 });
  expect(parentSpy).toHaveBeenCalledTimes(3);
  expect(childSpy).toHaveBeenCalledTimes(5);
});
```

首先明确一点：`effect`是可以嵌套的。

简单举个栗子就是：组件嵌套、计算属性。  
那有朋友就要问了，组件嵌套和effect嵌套有什么关系吗？  
其实关系就在于，组件中的`template`会被转成`render`函数，而组件要实现响应式，就得将`render`函数作为`ReactiveEffect`
的参数进行依赖收集。而当组件嵌套或者使用计算属性时，此时就会产生`effect`的嵌套，而这我们是需要支持的。

上面的单测就展示了`effect(parentSpy)`中嵌套了`childEffect`的情况，然后分别触发`num1`、`num2`和`num3`变化，然后观察`dummy`
的变化及`父子effect`的执行情况。

##### 2. 完善逻辑

首先先走一遍单测，看一下我们现有的代码哪里会不满足用例的需求。

<img src="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/202212160639280.png" width="999" alt="17_09_effect嵌套单测报错截图"/>

###### 2.1. 出现第一个问题

通过报错信息可以看到，我们期望`num3`为7，但是实际上`num3`还是2。  
很显然，`num3`并没有被更新，也就是`nums.num3 = 7`，并没有触发到`parentSpy`的执行。  
那我们反推回去，可以猜测`依赖收集`时，`depsMap`中并没有收集到`num3`的依赖。

为了验证这个猜想，我们在`nums.num3 = 7`这一行，打上断点，我们来调试一下。

<img src="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/202212160643089.png" width="888" alt="17_10_effect嵌套调试截图"/>

通过调试，可以看出，`depsMap`中果然只有`num1`、`num2`的依赖。

那为什么会造成这个情况呢？  
`num1`和`num2`的依赖都能收集到，那意思就是`num3`的`get`操作被触发时，没有`track`到相关的依赖。  
并且可以看到`depsMap`中连`num3`这个键都没有，那肯定就是`isTracking()`为`false`时，直接`return`掉了。

那我们再次给`parentSpy`中的`dummy.num3 = nums.num3;`这一行打上断点，调试一下看看。

<img src="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/202212190653535.png" width="555" alt="17_10_shouldTrack为false"/>

通过断点进到`num3`触发`get`后的`track`操作，可以看到`shouldTrack`为`false`，那这样的话，我们上面的猜想也就成立了，果然是这个原因。

分析一下过程，首先`shouldTrack`为一个全局变量。  
当`effect(parentSpy)`开始运行时，会运行`run`方法，`shouldTrack`被置为`true`；  
再当嵌套的`childEffect()`运行时，也会运行里层`_effect`的`run`方法，`shouldTrack`先被置为`true`
，进行原始依赖的运行，后被置为`false`，不允许再次收集依赖。  
当`childEffect`运行结束后，到我们断点的这一行，`shouldTrack`依旧为`false`，所以`nums3`的`track`就直接跳出了。  
但我们需要的是：`shouldTrack`应该为`true`，因为此时`父级effect`还并未执行结束。

###### 2.2. 解决第一个问题

那我们就可以用一个`lastShouldTrack`来存储上一次的`shouldTrack`，再当执行完时，恢复上一次的状态。

```ts
// src/reactivity/effect.ts

export class ReactiveEffect {
  // ... 省略部分代码

  run() {
    // 已经被stop，那就直接返回结果
    if (!this.active) {
      return this._fn();
    }

    let lastShouldTrack = shouldTrack;
    // 此时应该被收集依赖，可以给activeEffect赋值，去运行原始依赖
    activeEffect = this;
    shouldTrack = true;
    cleanupEffect(this);
    const result = this._fn();
    // 由于运行原始依赖的时候，会触发代理对象的get操作，会重复进行依赖收集
    // 调用完以后就恢复上次的状态
    shouldTrack = lastShouldTrack;

    return result;
  }

  // ... 省略部分代码
}
```

此时，再次调试一下，可以发现，`shouldTrack`已经是`true`了。

<img src="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/202212190721211.png" width="666" alt="17_11_shouldTrack正确恢复状态"/>

而且我们也已经可以给`nums3`收集到依赖了。

<img src="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/202212190724501.png" width="777" alt="17_12_num3收集到依赖"/>

###### 2.3. 出现第二个问题

这时，我们再次跑单测时，会发现`dummy.nums3`还是不对，还是没有更新。

<img src="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/202212190731875.png" width="777" alt="17_13_依赖收集到但nums3并未正确更新"/>

梳理一下现在的情况：

1. 收集到了`nums3`的依赖
2. `nums3`的依赖触发正常触发
3. `nums3`的值并未被更新

综上，那只能怀疑一点，那就是`nums3`的依赖收集的并不对。  
为什么会出现这个情况呢？

我们使用`activeEffect`这个全局变量来存储通过`effect`注册的依赖，而这么做的话，我们一次只能存储一个依赖。  
当从`外层effect`进入`里层effect`时，内层函数的执行会覆盖`activeEffect`的值，`activeEffect`的指向从`parentSpy`转向`childSpy`。  
并且，这个指向的变化是不可逆的，没办法从里向外层转。  
所以，导致`nums3`的依赖虽然收集到了，但是收集的`activeEffect`是`childSpy`，而不是`parentSpy`。

为了验证是不是这个问题，我们需要调整一下测试用例`nums.num3 = 7;`后面代码的执行顺序。

```ts
// src/reactivity/__tests__/effect.spec.ts

nums.num3 = 7;
expect(childSpy).toHaveBeenCalledTimes(5);
expect(parentSpy).toHaveBeenCalledTimes(3);
expect(dummy).toEqual({ num1: 4, num2: 10, num3: 7 });
```

<img src="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/202212200542307.png" width="666" alt="17_14_nums3变化parentSpy和childSpy调用次数"/>

果然如此，通过截图可以看出：  
`childSpy`的期望是通过的，那么`childSpy`的执行次数为`5`，而`parentSpy`的执行次数可以看到实际执行次数为`2`。  
那就证明`nums3`变化后触发了`childSpy`的执行，那收集到的依赖不对，就不言而喻了。

###### 2.4. 解决第二个问题

归根结底的原因，其实也可以理解成：由于`effect`的嵌套，导致`activeEffect`的指向未能恢复到上一次的状态。  
那我们要做的就是，将`activeEffect`先行保存，结束完以后再恢复。  
做法其实和解决第一个问题类似。

```ts
// src/reactivity/effect.ts

export class ReactiveEffect {
  // ... 省略部分代码

  run() {
    // 已经被stop，那就直接返回结果
    if (!this.active) {
      return this._fn();
    }

    let parent = activeEffect;
    let lastShouldTrack = shouldTrack;

    try {
      parent = activeEffect;
      // 此时应该被收集依赖，可以给activeEffect赋值，去运行原始依赖
      activeEffect = this;
      shouldTrack = true;

      cleanupEffect(this);
      return this._fn();
    } finally {
      // 由于运行原始依赖的时候，会触发代理对象的get操作，会重复进行依赖收集
      // 调用完以后就恢复上次的状态
      activeEffect = parent;
      shouldTrack = lastShouldTrack;
    }
  }

  // ... 省略部分代码
}
```

<img src="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/202212200809205.png" width="666" alt="17_15_解决effect后完整effect单测"/>

###### 2.5. 其他思路

仔细再回想一下解决上述两个问题的过程，其实都是，内层函数运行结束时，对应的变量没有恢复到外层的状态。  
这种类似嵌套和递归的过程，都是一种进到最里层，然后再一层一层向外走。  
有点类似于`洋葱圈模型`，类似这种，都可以类比成`入栈`和`出栈`的操作。  
那这样的话，我们就可以模拟一个`栈`结构，来进行这样的操作。

这里附上代码。

```ts
// src/reactivity/effect.ts

const effectStack: any = [];

export class ReactiveEffect {
  // ... 省略部分代码

  run() {
    // 已经被stop，那就直接返回结果
    if (!this.active) {
      return this._fn();
    }

    if (!effectStack.includes(this)) {
      cleanupEffect(this);
      let lastShouldTrack = shouldTrack;
      try {
        // 此时应该被收集依赖，可以给activeEffect赋值，去运行原始依赖
        shouldTrack = true;
        // 入栈
        effectStack.push(this);
        activeEffect = this;
        return this._fn();
      } finally {
        // 出栈
        effectStack.pop();
        // 由于运行原始依赖的时候，会触发代理对象的get操作，会重复进行依赖收集，所以调用完以后就关上开关，不允许再次收集依赖
        // 恢复 shouldTrack 开启之前的状态
        shouldTrack = lastShouldTrack;
        activeEffect = effectStack[effectStack.length - 1];
      }
    }
  }

  // ... 省略部分代码
}
```

[//]: # (###### 2.6. 代码重构 todo)

#### （三）无限递归循环

##### 1. 单测用例

```js
it('should avoid implicit infinite recursive loops with itself', () => {
  const counter = reactive({ num: 0 });
  const counterSpy = jest.fn(() => counter.num++);
  effect(counterSpy);

  expect(counter.num).toBe(1);
  expect(counterSpy).toHaveBeenCalledTimes(1);

  counter.num = 4;
  expect(counter.num).toBe(5);
  expect(counterSpy).toHaveBeenCalledTimes(2);
});
```

##### 2. 完善逻辑

还是先来跑一下这个单测，看看有什么问题。

<img src="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/202212210713293.png" width="666" alt="17_16_自增导致栈溢出"/>

通过单测可以看出`effect`中的依赖是一个自增操作`counter.num++`，单测运行过程中引起了`栈溢出`。

那为什么会出现`栈溢出`呢？  
那就要对比`自增`跟我们`之前的依赖`有什么不同？  
可以注意到，我们`之前的依赖`都是单一操作，要么`读`，要么`写`。  
而`自增`可以分成两步来看，先读取自身的值，然后再加一并写入。

再来分析一下执行过程:

首先读取`counter.num`的值，这会触发`track`操作，将当前副作用函数收集到`depsMap`中，接着将其加`1`后再赋值给`counter.num`
，此时会触发`trigger`操作，即把`depsMap`中的副作用函数取出并执行。但问题是该副作用函数正在执行中，还没有执行完毕，就要开始下一次的执行。这样会导致无限递归地调用自己，于是就产生了栈溢出。

解决办法并不难。  
通过分析这个问题我们能够发现，读取和设置操作是在同一个副作用函数内进行的。  
此时无论是`track`时收集的副作用函数，还是`trigger`时要触发执行的副作用函数，都是`activeEffect`。基于此，我们可以在`trigger`
动作发生时增加守卫条件：如果`trigger`触发执行的副作用函数与当前正在执行的副作用函数相同，则不触发执行，如以下代码所示：

```ts
// src/reactivity/effect.ts

export function triggerEffects(dep) {
  const effects = new Set<any>();

  // + 如果trigger触发执行的副作用函数与当前正在执行的副作用函数相同，则不触发执行
  dep && dep.forEach(effect => {
    if (effect !== activeEffect) {
      effects.add(effect);
    }
  });

  for (const effect of effects) {
    if (effect.scheduler) {
      // ps: effect._fn 为了让scheduler能拿到原始依赖
      effect.scheduler(effect._fn);
    } else {
      effect.run();
    }
  }
}
```

##### 3. 单测结果

 <img src="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/202212221503840.png" width="888" alt="17_17_解决自增引起的栈溢出"/>
