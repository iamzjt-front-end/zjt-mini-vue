# 08_实现isReactive和isReadonly

### 一、实现isReactive

> `isReactive`: 检查一个对象是否是由 reactive 创建的响应式代理。

#### 1. 单元测试

```ts
// src/reactivity/__tests__/reactive.spec.ts

import { reactive, isReactive } from '../reactive';

describe('reactive', function () {
  it('happy path', function () {
    const original = { foo: 1 };
    const observed = reactive(original);

    expect(observed).not.toBe(original);
    expect(observed.foo).toBe(original.foo);

    // + isReactive
    expect(isReactive(observed)).toBe(true);
    expect(isReactive(original)).toBe(false);
  });
});
```

#### 2. 代码实现

其实我们在`baseHandlers`中`createGetter`的时候，我们就已经传递过`isReadonly`的标识，那我们只要想办法将这个标识传递出来，就可以了。

那就得触发代理对象的`get`操作，那先在`reactive.ts`中导出一个`isReactive`。

```ts
// src/reactivity/reactive.ts

export function isReactive(value) {
  return value['is_reactive'];
}
```

接着在`get`中判断读取的`key`是否是`is_reactive`，然后返回对应结果即可。

```ts
// src/reactivity/baseHandlers.ts

function createGetter(isReadonly = false) {
  return function get(target, key) {
    const res = Reflect.get(target, key);

    // + 如果读取的 key 是 is_reactive, 则返回 true
    if (key === 'is_reactive') {
      return !isReadonly;
    }

    if (!isReadonly) {
      track(target, key);
    }

    return res;
  };
}
```

至此，基本逻辑已经实现，看一下单测结果。

```shell
yarn test reactive
```

<img src="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/202211110608378.png" width="888" alt="08_01_isReactive首次单测结果"/>

果然失败了呢，通过报错信息我们可以看见期望`Expected`是`false`，而实际值`Received`是`undefined`。

其实也很容易明白🐴，普通对象没有被代理，自然不会走我们封装的`get`，而且也没有这个属性，所以就返回`undefined`。

但是`isReactive(observed)`的测试是通过的，所以可知对代理过后的对象的判断是正确的，达到了期望。那只要想办法将`undefined`
转成`false`，且不影响`isReactive(observed)`返回的`true`即可。

两个思路：

- 思路1：如果是`true`就正确返回，如果是`undefined`，就返回`false`。那就利用空值合并运算符`??`捕获`undefined`。

   ```ts
   return value['is_reactive'] ?? false;
   ```

- 思路2：利用`!!运算符`，将表达式强转成`布尔类型`。

   ```ts
   return !!value['is_reactive'];
   ```

其实能够看出来，思路1比起思路2不太优雅，所以我们选择思路2。

<img src="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/202211110626780.png" width="666" alt="08_02_isReactive实现单测结果"/>

### 二、实现isReadonly

实现了`isReactive`之后，`isReadonly`就很类似了。

#### 1. 单元测试

```ts
// src/reactivity/__tests__/readonly.spec.ts

it('happy path', () => {
  const original = { foo: 1, bar: { baz: 2 } };
  const wrapped = readonly(original);

  expect(wrapped).not.toBe(original);
  expect(wrapped.foo).toBe(1);

  // ! 不能被set
  wrapped.foo = 2;
  expect(wrapped.foo).toBe(1);

  // + isReadonly
  expect(isReadonly(wrapped)).toBe(true);
  expect(isReadonly(original)).toBe(false);
});
```

#### 2. 代码实现

同上，即可。

```ts
// src/reactivity/reactive.ts

export function isReadonly(value) {
  return !!value['is_readonly'];
}
```

```ts
// src/reactivity/baseHandlers.ts

function createGetter(isReadonly = false) {
  return function get(target, key) {
    const res = Reflect.get(target, key);

    if (key === 'is_reactive') {
      return !isReadonly;
    } else if (key === 'is_readonly') { // + is_readonly
      return isReadonly;
    }

    if (!isReadonly) {
      track(target, key);
    }

    return res;
  };
}
```

走一下`readonly`的单测。

```shell
# --silent=true 是禁用控制台打印，静默测试，主要是因为之前的set会触发console.warn
yarn test readonly --silent=true
```

<img src="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/202211110651096.png" width="666" alt="08_03_isReadonly单测结果"/>

### 三、代码重构

```ts
export function isReactive(value) {
  return !!value['is_reactive'];
}

export function isReadonly(value) {
  return !!value['is_readonly'];
}
```

这里就是一个优化的点，就像报错信息一样，报错可以用字典来维护起来，这里也是一样，可以用`ts`的`enum`枚举来维护。

```ts
// src/reactivity/reactive.ts

export const enum ReactiveFlags {
  IS_REACTIVE = '__v_isReactive',
  IS_READONLY = '__v_isReadonly'
}

// + 用到之前标识的地方，相应的进行修改
export function isReactive(value) {
  return !!value[ReactiveFlags.IS_REACTIVE];
}

export function isReadonly(value) {
  return !!value[ReactiveFlags.IS_READONLY];
}
```

```ts
// src/reactivity/baseHandlers.ts

import { track, trigger } from './effect';
import { ReactiveFlags } from './reactive';

function createGetter(isReadonly = false) {
  return function get(target, key) {
    const res = Reflect.get(target, key);

    // + 改成枚举的方式
    if (key === ReactiveFlags.IS_REACTIVE) {
      return !isReadonly;
    } else if (key === ReactiveFlags.IS_READONLY) {
      return isReadonly;
    }

    if (!isReadonly) {
      track(target, key);
    }

    return res;
  };
}
```

大概就这么多，重构完以后，依旧完整的跑一遍所有的单测。

```shell
# --watchAll可以进入watch模式, 下面有很多 usage 可供使用
# 这样做的好处是, 我们不需要每次都执行 yarn test, 在第一次执行之后, 进程会自动监听测试用例的变化, 如果测试用例代码发生了变化, 会自动执行
yarn test --watchAll --silent=true
```

<img src="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/202211110739444.png" width="666" alt="08_04_重构后的单测结果"/>
