#  Vue3 响应式

只要问到 `Vue` 相关的内容，似乎总绕不过 **响应式原理** 的话题，随之而来的回答必然是围绕着 `Object.defineProperty` 和 `Proxy` 来展开（即 `Vue2` 和 `Vue3`），但若继续追问某些具体实现是不是就仓促结束回答了（~~**`你跑我追，你不跑我还追`**~~）。

本文就不再过多介绍 `Vue2` 中响应式的处理，感兴趣可以参考 ***\*从 vue 源码看问题 —— 如何理解 `Vue` 响应式？\****[2]，但是会有简单提及，下面就来看看 `Vue3` 中是如何处理 **原始值、Object、Array、Set、Map** 等数据类型的响应式。

# 从 `Object.defineProperty` 到 `Proxy`

## Object.defineProperty

**`Object.defineProperty(obj, prop, descriptor)`** 方法会直接在一个对象上定义一个 **新属性**，或修改一个 **对象** 的 **现有属性**，并返回此对象，其参数具体为：

- `obj`：要定义属性的对象
- `prop`：要定义或修改的 **属性名称** 或 ***\*`Symbol`\****[3]
- `descriptor`：要定义或修改的 **属性描述符**

从以上的描述就可以看出一些限制，比如：

- 目标是 **对象属性**，不是 **整个对象**

- 一次只能 **定义或修改一个属性**

- - 当然有对应的一次处理多个属性的方法 **`Object.defineProperties()`**[4]，但在 `vue` 中并不适用，因为 `vue` 不能提前知道用户传入的对象都有什么属性，因此还是得经过类似 `Object.keys() + for` 循环的方式获取所有的 `key -> value`，而这其实是没有必要使用 **`Object.defineProperties()`**[5]

### 在 Vue2 中的缺陷

`Object.defineProperty()` 实际是通过 **定义** 或 **修改** **`对象属性`** 的描述符来实现 **数据劫持**，其对应的缺点也是没法被忽略的：

- 只能拦截对象属性的 `get` 和 `set` 操作，比如无法拦截 `delete`、`in`、`方法调用` 等操作

- 动态添加新属性（响应式丢失）

- - 保证后续使用的属性要在初始化声明 `data` 时进行定义
  - 使用 `this.$set()` 设置新属性

- 通过 `delete` 删除属性（响应式丢失）

- - 使用 `this.$delete()` 删除属性

- 使用数组索引 **替换/新增** 元素（响应式丢失）

- - 使用 `this.$set()` 设置新元素

- 使用数组 `push、pop、shift、unshift、splice、sort、reverse` 等 **原生方法** 改变原数组时（响应式丢失）

- - 使用 **重写/增强** 后的 `push、pop、shift、unshift、splice、sort、reverse` 方法

- 一次只能对一个属性实现 **数据劫持**，需要遍历对所有属性进行劫持

- 数据结构复杂时（属性值为 **引用类型数据**），需要通过 **递归** 进行处理

### **【扩展】`Object.defineProperty` 和 `Array` ？**

它们有啥关系，其实没有啥关系，只是大家习惯性的会回答 `Object.defineProperty` 不能拦截 `Array` 的操作，这句话说得对但也不对

> **使用 Object.defineProperty 拦截 Array**

`Object.defineProperty` 可用于实现对象属性的 `get` 和 `set` 拦截，而数组其实也是对象，那自然是可以实现对应的拦截操作，如下：

![图片](https://mmbiz.qpic.cn/mmbiz/pfCCZhlbMQTn1pZ8MnehWiba2cp9UdjByJcgPicbrwnl52IX5kJMz4yAG46Lxrj7QIk7Lnt2T1ArTPjJNibbicZKMQ/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)![图片](https://mmbiz.qpic.cn/mmbiz/pfCCZhlbMQTn1pZ8MnehWiba2cp9UdjByT6ZmicicmESEib1fFabRmy7QD2yicY1NEMNvSOjjHYAdozIribb3eROBTUg/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

> **Vue2 为什么不使用 Object.defineProperty 拦截 Array？**

尤大在曾在 `GitHub` 的 `Issue` 中做过如下回复：

![图片](https://mmbiz.qpic.cn/mmbiz/pfCCZhlbMQTn1pZ8MnehWiba2cp9UdjBy3wFPibHjNKhp6ViaE0FYGcJjtSw926iaJUMcsY5LEMFH5sYKIyXTc8clA/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

说实话性能问题到底指的是什么呢？

**数组 和 普通对象 在使用场景下有区别**，在项目中使用数组的目的大多是为了 **遍历**，即比较少会使用 `array[index] = xxx` 的形式，更多的是使用数组的 `Api` 的方式

- **数组长度是多变的，不可能像普通对象一样先在 `data` 选项中提前声明好所有元素**，比如通过 `array[index] = xxx` 方式赋值时，一旦 `index` 的值超过了现有的最大索引值，那么当前的添加的新元素也不会具有响应式
- **数组存储的元素比较多，不可能为每个数组元素都设置 `getter/setter`**
- **无法拦截数组原生方法如 `push、pop、shift、unshift` 等的调用**，最终仍需 **重写/增强** 原生方法

## Proxy & Reflect

由于在 `Vue2` 中使用 `Object.defineProperty` 带来的缺陷，导致在 `Vue2` 中不得不提供了一些额外的方法（如：`Vue.set、Vue.delete()`）解决问题，而在 `Vue3` 中使用了 `Proxy` 的方式来实现 **数据劫持**，而上述的问题在 `Proxy` 中都可以得到解决。

### Proxy

***\*`Proxy`\****[6] 主要用于创建一个 **对象的代理**，从而实现基本操作的拦截和自定义（如属性查找、赋值、枚举、函数调用等），本质上是通过拦截对象 **内部方法** 的执行实现代理，而对象本身根据规范定义的不同又会区分为 **常规对象** 和 **异质对象**（这不是重点，可自行了解）。

- **`new Proxy(target, handler)`** 是针对整个对象进行的代理，不是某个属性

- 代理对象属性拥有 **读取、修改、删除、新增、是否存在属性** 等操作相应的捕捉器，***\*更多可见\****[7]

- - **`get()`**[8] 属性 **读取** 操作的捕捉器
  - **`set()`**[9] 属性 **设置** 操作的捕捉器
  - **`deleteProperty()`**[10] 是 **`delete`**[11] 操作符的捕捉器
  - **`ownKeys()`**[12] 是 **`Object.getOwnPropertyNames`**[13] 方法和 **`Object.getOwnPropertySymbols`**[14] 方法的捕捉器
  - **`has()`**[15] 是 **`in`**[16] 操作符的捕捉器

### Reflect

***\*`Reflect`\****[17] 是一个内置的对象，它提供拦截 `JavaScript` 操作的方法，这些方法与 ***\*`Proxy handlers`\****[18] 提供的的方法是一一对应的，且 `Reflect` 不是一个函数对象，即不能进行实例化，其所有属性和方法都是静态的。

- **`Reflect.get(target, propertyKey[, receiver])`**[19] 获取对象身上某个属性的值，类似于 `target[name]`
- **`Reflect.set(target, propertyKey, value[, receiver])`**[20] 将值分配给属性的函数。返回一个**`Boolean`**[21]，如果更新成功，则返回`true`
- **`Reflect.deleteProperty(target, propertyKey)`**[22] 作为函数的**`delete`**[23]操作符，相当于执行 `delete target[name]`
- **`Reflect.ownKeys(target)`**[24] 返回一个包含所有自身属性（不包含继承属性）的数组。(类似于 **`Object.keys()`**[25], 但不会受`enumerable` 影响)
- **`Reflect.has(target, propertyKey)`**[26] 判断一个对象是否存在某个属性，和 **`in` 运算符**[27] 的功能完全相同

***\*更多方法点此可见\****[28]

### Proxy 为什么需要 Reflect 呢？

在 **`Proxy`** 的 **`get(target, key, receiver)、set(target, key, newVal, receiver)`** 的捕获器中都能接到前面所列举的参数：

- **`target`** 指的是 **原始数据对象**
- **`key`** 指的是当前操作的 **属性名**
- **`newVal`** 指的是当前操作接收到的 **最新值**
- **`receiver`** 指向的是当前操作 **正确的上下文**

#### 怎么理解 `Proxy handler` 中 `receiver` 指向的是当前操作正确上的下文呢？

- 正常情况下，**`receiver`** 指向的是 **当前的代理对象**

  ![图片](https://mmbiz.qpic.cn/mmbiz/pfCCZhlbMQTn1pZ8MnehWiba2cp9UdjByS3jCtYHM2R0KXavnhV2otSnaa9IXQAyDicPjDpkLQL2m9sibpNPnW4qA/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)image.png

- 特殊情况下，**`receiver`** 指向的是 **引发当前操作的对象**

  ![图片](https://mmbiz.qpic.cn/mmbiz/pfCCZhlbMQTn1pZ8MnehWiba2cp9UdjByckW6HKDcqlQWqVoG8bN7DntqGyjmfqeCRw2V8sJ6XK81CmLcOiarGcw/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)image.png

- - 通过 `Object.setPrototypeOf()` 方法将代理对象 `proxy` 设置为普通对象 `obj` 的原型
  - 通过 `obj.name` 访问其不存在的 `name` 属性，由于原型链的存在，最终会访问到 `proxy.name` 上，即触发 `get` 捕获器

在 **`Reflect`** 的方法中通常只需要传递 `target、key、newVal` 等，但为了能够处理上述提到的特殊情况，一般也需要传递 `receiver` 参数，因为 **Reflect 方法中传递的 receiver 参数代表执行原始操作时的 `this` 指向**，比如：`Reflect.get(target, key , receiver)`、`Reflect.set(target, key, newVal, receiver)`。

**总结**：**`Reflect`** 是为了在执行对应的拦截操作的方法时能 **传递正确的 `this` 上下文**。

# Vue3 如何使用 Proxy 实现数据劫持？

**`Vue3`** 中提供了 **`reactive()`** 和 **`ref()`** 两个方法用来将 **目标数据** 变成 **响应式数据**，而通过 `Proxy` 来实现 **数据劫持（或代理）** 的具体实现就在其中，

## reactive 函数

从源码来看，其核心其实就是 `createReactiveObject(...)` 函数，那么继续往下查看对应的内容

**源码位置：`packages\reactivity\src\reactive.ts`**

```
export function reactive(target: object) {
  // if trying to observe a readonly proxy, return the readonly version.
  // 若目标对象是响应式的只读数据，则直接返回
  if (isReadonly(target)) {
    return target
  }

  // 否则将目标数据尝试变成响应式数据
  return createReactiveObject(
    target,
    false,
    mutableHandlers, // 对象类型的 handlers
    mutableCollectionHandlers, // 集合类型的 handlers
    reactiveMap
  )
}
复制代码
```

### createReactiveObject() 函数

源码的体现也是非常简单，无非就是做一些前置判断处理：

- 若目标数据是 **原始值类型**，直接向返回 **原数据**

- 若目标数据的 `__v_raw` 属性为 `true`，且是【非响应式数据】或 不是通过调用 `readonly()` 方法，则直接返回 **原数据**

- 若目标数据已存在相应的 `proxy` 代理对象，则直接返回 **对应的代理对象**

- 若目标数据不存在对应的 **白名单数据类型** 中，则直接返回原数据，支持响应式的数据类型如下：

- - **可扩展的对象**，即是否可以在它上面添加新的属性
  - **__v_skip 属性不存在或值为 false 的对象**
  - **数据类型为 `Object、Array、Map、Set、WeakMap、WeakSet` 的对象**
  - 其他数据都统一被认为是 **无效的响应式数据对象**

- 通过 **`Proxy`** 创建代理对象，根据目标数据类型选择不同的 **`Proxy handlers`**

看来具体的实现又在不同数据类型的 **捕获器** 中，即下面源码的 **`collectionHandlers`** 和 **`baseHandlers`** ，而它们则对应的是在上述 `reactive()` 函数中为 `createReactiveObject()` 函数传递的 `mutableCollectionHandlers` 和 `mutableHandlers` 参数。

**源码位置：`packages\reactivity\src\reactive.ts`**

```
function createReactiveObject(
  target: Target,
  isReadonly: boolean,
  baseHandlers: ProxyHandler<any>,
  collectionHandlers: ProxyHandler<any>,
  proxyMap: WeakMap<Target, any>
) {

  // 非对象类型直接返回
  if (!isObject(target)) {
    if (__DEV__) {
      console.warn(`value cannot be made reactive: ${String(target)}`)
    }
    return target
  }

  // 目标数据的 __v_raw 属性若为 true，且是【非响应式数据】或 不是通过调用 readonly() 方法，则直接返回
  if (
    target[ReactiveFlags.RAW] &&
    !(isReadonly && target[ReactiveFlags.IS_REACTIVE])
  ) {
    return target
  }

  // 目标对象已存在相应的 proxy 代理对象，则直接返回
  const existingProxy = proxyMap.get(target)
  if (existingProxy) {
    return existingProxy
  }

  // 只有在白名单中的值类型才可以被代理监测，否则直接返回
  const targetType = getTargetType(target)
  if (targetType === TargetType.INVALID) {
    return target     
  }

  // 创建代理对象
  const proxy = new Proxy(
    target,
    // 若目标对象是集合类型（Set、Map）则使用集合类型对应的捕获器，否则使用基础捕获器
    targetType === TargetType.COLLECTION ? collectionHandlers : baseHandlers 
  )

  // 将对应的代理对象存储在 proxyMap 中
  proxyMap.set(target, proxy)

  return proxy
}
复制代码
```

## 捕获器 Handlers

### 对象类型的捕获器 — `mutableHandlers`

这里的对象类型指的是 **数组** 和 **普通对象**

**源码位置：`packages\reactivity\src\baseHandlers.ts`**

```
export const mutableHandlers: ProxyHandler<object> = {
  get,
  set,
  deleteProperty,
  has,
  ownKeys
}
复制代码
```

以上这些捕获器其实就是我们在上述 `Proxy` 部分列举出来的捕获器，显然可以拦截对普通对象的如下操作：

- **读取**，如 **`obj.name`**
- **设置**，如 **`obj.name = 'zs'`**
- **删除属性**，如 **`delete obj.name`**
- **判断是否存在对应属性**，如 **`name in obj`**
- **获取对象自身的属性值**，如 **`obj.getOwnPropertyNames()` 和 `obj.getOwnPropertySymbols()`**

#### `get` 捕获器

具体信息在下面的注释中，这里只列举核心内容：

- 若当前数据对象是 **数组**，则 **重写/增强** 数组对应的方法
- 数组元素的 **查找方法**：`includes、indexOf、lastIndexOf`
- **修改原数组** 的方法：`push、pop、unshift、shift、splice`
- 若当前数据对象是 **普通对象**，且非 **只读** 的则通过 `track(target, TrackOpTypes.GET, key)` 进行 **依赖收集**
- 若当前数据对象是 **浅层响应** 的，则直接返回其对应属性值
- 若当前数据对象是 **ref** 类型的，则会进行 **自动脱 ref**
- 若当前数据对象的属性值是 **对象类型**
- 若当前属性值属于 **只读的**，则通过 `readonly(res)` 向外返回其结果
- 否则会将当前属性值以 `reactive(res)` 向外返回 **proxy 代理对象**
- 否则直接向外返回对应的 **属性值**

```
function createGetter(isReadonly = false, shallow = false) {
  return function get(target: Target, key: string | symbol, receiver: object) {
    // 当直接通过指定 key 访问 vue 内置自定义的对象属性时，返回其对应的值
    if (key === ReactiveFlags.IS_REACTIVE) {
      return !isReadonly
    } else if (key === ReactiveFlags.IS_READONLY) {
      return isReadonly
    } else if (key === ReactiveFlags.IS_SHALLOW) {
      return shallow
    } else if (
      key === ReactiveFlags.RAW &&
      receiver ===
        (isReadonly
          ? shallow
            ? shallowReadonlyMap
            : readonlyMap
          : shallow
          ? shallowReactiveMap
          : reactiveMap
        ).get(target)
    ) {
      return target
    }

    // 判断是否为数组类型
    const targetIsArray = isArray(target)

    // 数组对象
    if (!isReadonly && targetIsArray && hasOwn(arrayInstrumentations, key)) {
      // 重写/增强数组的方法： 
      //  - 查找方法：includes、indexOf、lastIndexOf
      //  - 修改原数组的方法：push、pop、unshift、shift、splice
      return Reflect.get(arrayInstrumentations, key, receiver)
    }

    // 获取对应属性值
    const res = Reflect.get(target, key, receiver)

    if (isSymbol(key) ? builtInSymbols.has(key) : isNonTrackableKeys(key)) {
      return res
    }

    // 依赖收集
    if (!isReadonly) {
      track(target, TrackOpTypes.GET, key)
    }

    // 浅层响应
    if (shallow) {
      return res
    }

    // 若是 ref 类型响应式数据，会进行【自动脱 ref】，但不支持【数组】+【索引】的访问方式
    if (isRef(res)) {
      const shouldUnwrap = !targetIsArray || !isIntegerKey(key)
      return shouldUnwrap ? res.value : res
    }

    // 属性值是对象类型：
    //  - 是只读属性，则通过 readonly() 返回结果，
    //  - 且是非只读属性，则递归调用 reactive 向外返回 proxy 代理对象
    if (isObject(res)) {
      return isReadonly ? readonly(res) : reactive(res)
    }

    return res
  }
}
复制代码
```

#### `set` 捕获器

除去额外的边界处理，其实核心还是 **更新属性值**，并通过 `trigger(...)` 触发依赖更新

```
function createSetter(shallow = false) {
  return function set(
    target: object,
    key: string | symbol,
    value: unknown,
    receiver: object
  ): boolean {
    // 保存旧的数据
    let oldValue = (target as any)[key]

    // 若原数据值属于 只读 且 ref 类型，并且新数据值不属于 ref 类型，则意味着修改失败
    if (isReadonly(oldValue) && isRef(oldValue) && !isRef(value)) {
      return false
    }

    if (!shallow && !isReadonly(value)) {
      if (!isShallow(value)) {
        value = toRaw(value)
        oldValue = toRaw(oldValue)
      }
      if (!isArray(target) && isRef(oldValue) && !isRef(value)) {
        oldValue.value = value
        return true
      }
    } else {
      // in shallow mode, objects are set as-is regardless of reactive or not
    }

    // 是否存在对应的 key
    const hadKey =
      isArray(target) && isIntegerKey(key)
        ? Number(key) < target.length
        : hasOwn(target, key)

    // 设置对应值
    const result = Reflect.set(target, key, value, receiver)

    // 若目标对象是原始原型链上的内容（非自定义添加），则不触发依赖更新
    if (target === toRaw(receiver)) {
      if (!hadKey) {
        // 目标对象不存在对应的 key，则为新增操作
        trigger(target, TriggerOpTypes.ADD, key, value)
      } else if (hasChanged(value, oldValue)) {
        // 目标对象存在对应的值，则为修改操作
        trigger(target, TriggerOpTypes.SET, key, value, oldValue)
      }
    }

    // 返回修改结果
    return result
  }
}
复制代码
```

#### `deleteProperty` & `has` & `ownKeys` 捕获器

这三个捕获器内容非常简洁，其中 `has` 和 `ownKeys` 本质也属于 **读取操作**，因此需要通过 `track()` 进行依赖收集，而 `deleteProperty` 相当于修改操作，因此需要 `trigger()` 触发更新

```
function deleteProperty(target: object, key: string | symbol): boolean {
  const hadKey = hasOwn(target, key)
  const oldValue = (target as any)[key]
  const result = Reflect.deleteProperty(target, key)
  // 目标对象上存在对应的 key ，并且能成功删除，才会触发依赖更新
  if (result && hadKey) {
    trigger(target, TriggerOpTypes.DELETE, key, undefined, oldValue)
  }
  return result
}

function has(target: object, key: string | symbol): boolean {
  const result = Reflect.has(target, key)
  if (!isSymbol(key) || !builtInSymbols.has(key)) {
    track(target, TrackOpTypes.HAS, key)
  }
  return result
}

function ownKeys(target: object): (string | symbol)[] {
  track(target, TrackOpTypes.ITERATE, isArray(target) ? 'length' : ITERATE_KEY)
  return Reflect.ownKeys(target)
}
复制代码
```

### 数组类型捕获器 —— `arrayInstrumentations`

**数组类型** 和 **对象类型** 的大部分操作是可以共用的，比如 `obj.name` 和 `arr[index]` 等，但数组类型的操作还是会比对象类型更丰富一些，而这些就需要特殊处理。

> **源码位置：`packages\reactivity\src\collectionHandlers.ts`**

#### 处理数组索引 `index` 和 `length`

数组的 `index` 和 `length` 是会相互影响的，比如存在数组 `const arr = [1]` ：

- `arr[1] = 2` 的操作会隐式修改 `length` 的属性值
- `arr.length = 0` 的操作会导致原索引位的值发生变更

为了能够合理触发和 `length` 相关副作用函数的执行，在 `set()` 捕获器中会判断当前操作的类型：

- 当 `Number(key) < target.length` 证明是修改操作，对应 `TriggerOpTypes.SET` 类型，即当前操作不会改变 `length` 的值，**不需要** 触发和 `length` 相关副作用函数的执行
- 当 `Number(key) >= target.length` 证明是新增操作，`TriggerOpTypes.ADD` 类型，即当前操作会改变 `length` 的值，**需要** 触发和 `length` 相关副作用函数的执行

```
function createSetter(shallow = false) {
  return function set(
    target: object,
    key: string | symbol,
    value: unknown,
    receiver: object
  ): boolean {
  
   省略其他代码
   
    const hadKey =
      isArray(target) && isIntegerKey(key)
        ? Number(key) < target.length
        : hasOwn(target, key)
        
    const result = Reflect.set(target, key, value, receiver)
    // don't trigger if target is something up in the prototype chain of original
    
    if (target === toRaw(receiver)) {
      if (!hadKey) {
        trigger(target, TriggerOpTypes.ADD, key, value)
      } else if (hasChanged(value, oldValue)) {
        trigger(target, TriggerOpTypes.SET, key, value, oldValue)
      }
    }
    return result
  }
}
复制代码
```

#### 处理数组的查找方法

数组的查找方法包括 `includes`、`indexOf`、`lastIndexOf`，这些方法通常情况下是能够按预期进行工作，但还是需要对某些特殊情况进行处理：

- **当查找的目标数据是响应式数据本身时**，得到的就不是预期结果

  ```
  const obj = {}
  const proxy = reactive([obj])
  console.log(proxy.includs(proxy[0])) // false
  复制代码
  ```

- - 【**产生原因**】首先这里涉及到了两次读取操作，**第一次** 是 `proxy[0]` 此时会触发 `get` 捕获器并为 `obj` 生成对应代理对象并返回，**第二次** 是 `proxy.includs()` 的调用，它会遍历数组的每个元素，即会触发 `get` 捕获器，并又生成一个新的代理对象并返回，而这两次生成的代理对象不是同一个，因此返回 `false`
  - 【**解决方案**】源码中会在 `get` 中设置一个名为 `proxyMap` 的 `WeakMap` 集合用于存储每个响应式对象，在触发 `get` 时优先返回 `proxyMap` 存在的响应式对象，这样不管触发多少次 `get` 都能返回相同的响应式数据

- **当在响应式对象中查找原始数据时**，得到的就不是预期结果

  ```
  const obj = {}
  const proxy = reactive([obj])
  console.log(proxy.includs(obj)) // false
  复制代码
  ```

- - 在 重写/增强 的 `includes`、`indexOf`、`lastIndexOf` 等方法中，会将当前方法内部访问到的响应式数据转换为原始数据，然后调用数组对应的原始方法进行查找，若查找结果为 `true` 则直接返回结果
  - 若以上操作没有查找到，则通过将当前方法传入的参数转换为原始数据，在调用数组的原始方法，此时直接将对应的结果向外进行返回
  - 【**产生原因**】`proxy.includes()` 会触发 `get` 捕获器并为 `obj` 生成对应代理对象并返回，而 `includes` 方法的参数传递的是 **原始数据**，相当于此时是 **响应式对象** 和 **原始数据对象** 进行比较，因此对应的结果一定是为 `false`
  - 【**解决方案**】核心就是将它们的数据类型统一，即统一都使用 **原始值数据对比** 或 **响应式数据对比**，由于 `includes()` 的方法本身并不支持对传入参数或内部响应式数据的处理，因此需要自定义以上对应的数组查找方法

**源码位置：`packages\reactivity\src\baseHandlers.ts`**

```
;(['includes', 'indexOf', 'lastIndexOf'] as const).forEach(key => {
    instrumentations[key] = function (this: unknown[], ...args: unknown[]) {
      // 外部调用上述方法，默认其内的 this 指向的是代理数组对象，
      // 但实际上是需要通过原始数组中进行遍历查找
      const arr = toRaw(this) as any
      for (let i = 0, l = this.length; i < l; i++) {
        track(arr, TrackOpTypes.GET, i + '')
      }
      // we run the method using the original args first (which may be reactive)
      const res = arr[key](...args)
      if (res === -1 || res === false) {
        // if that didn't work, run it again using raw values.
        return arr[key](...args.map(toRaw))
      } else {
        return res
      }
    }
  })
复制代码
```

#### 处理数组影响 `length` 的方法

隐式修改数组长度的原型方法包括 **`push`、`pop`、`shift`、`unshift`、`splice`** 等，在调用这些方法的同时会间接的读取数组的 `length` 属性，又因为这些方法具有修改数组长度的能力，即相当于 `length` 的设置操作，若不进行特殊处理，会导致与 `length` 属性相关的副作用函数被重复执行，即 **栈溢出**，比如：

```
const proxy = reactive([])

// 第一个副作用函数
effect(() => {
  proxy.push(1) // 读取 + 设置 操作
})

// 第二个副作用函数
effect(() => {
  proxy.push(2) // 读取 + 设置 操作（此时进行 trigger 时，会触发包括第一个副作用函数的内容，然后循环导致栈溢出）
})
复制代码
```

在源码中还是通过 **重写/增强** 上述对应数组方法的形式实现自定义的逻辑处理：

- 在调用真正的数组原型方法前，会通过设置 `pauseTracking()` 方法来禁止 `track` 依赖收集
- 在调用数组原生方法后，在通过 `resetTracking()` 方法恢复 `track` 进行依赖收集
- 实际上以上的两个方法就是通过控制 `shouldTrack` 变量为 `true` 或 `false`，使得在 `track` 函数执行时是否要执行原来的依赖收集逻辑

**源码位置：`packages\reactivity\src\baseHandlers.ts`**

```
;(['push', 'pop', 'shift', 'unshift', 'splice'] as const).forEach(key => {
    instrumentations[key] = function (this: unknown[], ...args: unknown[]) {
      pauseTracking()
      const res = (toRaw(this) as any)[key].apply(this, args)
      resetTracking()
      return res
    }
  })
复制代码
```

### 集合类型的捕获器 — `mutableCollectionHandlers`

**集合类型** 包括 `Map`、`WeakMap`、`Set`、`WeakSet` 等，而对 **集合类型** 的 **代理模式** 和 **对象类型** 需要有所不同，因为 **集合类型** 和 **对象类型** 的操作方法是不同的，比如：

> **`Map` 类型** 的原型 **属性** 和 **方法** 如下，***\*详情可见\****[29]：

- size
- clear()
- delete(key)
- has(key)
- get(key)
- set(key)
- keys()
- values()
- entries()
- forEach(cb)

> **`Set` 类型** 的原型 **属性** 和 **方法** 如下，***\*详情可见\****[30]：

- size
- add(value)
- clear()
- delete(value)
- has(value)
- keys()
- values()
- entries()
- forEach(cb)

> **源码位置：`packages\reactivity\src\collectionHandlers.ts`**

#### 解决 `代理对象` 无法访问 `集合类型` 对应的 `属性` 和 `方法`

代理集合类型的第一个问题，就是代理对象没法获取到集合类型的属性和方法，比如：

![图片](https://mmbiz.qpic.cn/mmbiz/pfCCZhlbMQTn1pZ8MnehWiba2cp9UdjByicgpp3qlH1NroeHhadXrP9M7oR2syKw1zhI9B24ao0I0qfzZ2Ig1QaA/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

从报错信息可以看出 `size` 属性是一个访问器属性，所以它被作为方法调用了，而主要错误原因就是在这个访问器中的 `this` 指向的是 **代理对象**，在源码中就是通过为这些特定的 **属性** 和 **方法** 定义对应的 **key** 的 **mutableInstrumentations** 对象，并且在其对应的 **属性** 和 **方法** 中将 `this`指向为 **原对象**.

```
function has(this: CollectionTypes, key: unknown, isReadonly = false): boolean {
  const target = (this as any)[ReactiveFlags.RAW]
  const rawTarget = toRaw(target)
  const rawKey = toRaw(key)
  if (key !== rawKey) {
    !isReadonly && track(rawTarget, TrackOpTypes.HAS, key)
  }
  !isReadonly && track(rawTarget, TrackOpTypes.HAS, rawKey)
  return key === rawKey
    ? target.has(key)
    : target.has(key) || target.has(rawKey)
}

function size(target: IterableCollections, isReadonly = false) {
  target = (target as any)[ReactiveFlags.RAW]
  !isReadonly && track(toRaw(target), TrackOpTypes.ITERATE, ITERATE_KEY)
  return Reflect.get(target, 'size', target)
}

省略其他代码
复制代码
```

#### 处理集合类型的响应式

集合建立响应式核心还是 `track` 和 `trigger`，转而思考的问题就变成，什么时候需要 `track`、什么时候需要 `trigger`:

- `track` 时机：`get()、get size()、has()、forEach()`
- `trigger` 时机：`add()、set()、delete()、clear()`

这里涉及一些优化的内容，比如：

- 在 `add()` 中通过 `has()` 判断当前添加的元素是否已经存在于 `Set` 集合中时，若已存在就不需要进行 `trigger()` 操作，因为 `Set` 集合本身的一个特性就是 **去重**
- 在 `delete()` 中通过 `has()` 判断当前删除的元素或属性是否存在，若不存在就不需要进行 `trigger()` 操作，因为此时的删除操作是 **无效的**

```
function createInstrumentations() {
  const mutableInstrumentations: Record<string, Function> = {
    get(this: MapTypes, key: unknown) {// track
      return get(this, key)
    },
    get size() {// track
      return size(this as unknown as IterableCollections)
    },
    has,// track
    add,// trigger
    set,// trigger
    delete: deleteEntry,// trigger
    clear,// trigger
    forEach: createForEach(false, false) // track
  }
  省略其他代码
}
复制代码
```

#### 避免污染原始数据

通过重写集合类型的方法并手动指定其中的 `this` 指向为 **原始对象** 的方式，解决 **代理对象** 无法访问 **集合类型** 对应的 **属性** 和 **方法** 的问题，但这样的实现方式也带来了另一个问题：**`原始数据被污染`** 。

简单来说，我们只希望 **代理对象（`响应式对象`）** 才具备 **依赖收集(`track`)** 和 **依赖更新(`trigger`)** 的能力，而通过 **原始数据** 进行的操作不应该具有响应式的能力。

如果只是单纯的把所有操作直接作用到 **原始对象** 上就不能保证这个结果，比如 

```
 // 原数数据 originalData1
  const originalData1 = new Map({});
  // 代理对象 proxyData1
  const proxyData1 = reactive(originalData1);

  // 另一个代理对象 proxyData2
  const proxyData2 = reactive(new Map({}));

  // 将 proxyData2 做为 proxyData1 一个键值
  // 【注意】此时的 set() 经过重写，其内部 this 已经指向 原始对象（originalData1），等价于 原始对象 originalData1 上存储了一个 响应式对象 proxyData2
  proxyData1.set("proxyData2", proxyData2);

  // 若不做额外处理，如下基于 原始数据的操作 就会触发 track 和 trigger
  originalData1.get("proxyData2").set("name", "zs");
复制代码
```

在源码中的解决方案也是很简单，直接通过 `value = toRaw(value)` 获取当前设置值对应的 **原始数据**，这样旧可以避免 **响应式数据对原始数据的污染**。

#### 处理 `forEach` 回调参数

首先 **`Map.prototype.forEach(callbackFn [, thisArg])`**[31] 其中 `callbackFn` 回调函数会接收三个参数：

- 当前的 **值 `value`**
- 当前的 **键 `key`**
- 正在被遍历的 **`Map` 对象**（原始对象）

**遍历操作** 等价于 **读取操作**，在处理 **普通对象** 的 `get()` 捕获器中有一个处理，如果当前访问的属性值是 **对象类型** 那么就会向外返回其对应的 **代理对象**，目的是实现 **惰性响应** 和 **深层响应**，这个处理也同样适用于 **集合类型**。

因此，在源码中通过 `callback.call(thisArg, wrap(value), wrap(key), observed)` 的方式将 `Map` 类型的 **键** 和 **值** 进行响应式处理，以及进行 `track` 操作，因为 `Map` 类型关注的就是 **键** 和 **值**。

```
function createForEach(isReadonly: boolean, isShallow: boolean) {
  return function forEach(
    this: IterableCollections,
    callback: Function,
    thisArg?: unknown
  ) {
    const observed = this as any
    const target = observed[ReactiveFlags.RAW]
    const rawTarget = toRaw(target)
    const wrap = isShallow ? toShallow : isReadonly ? toReadonly : toReactive
    !isReadonly && track(rawTarget, TrackOpTypes.ITERATE, ITERATE_KEY)
    return target.forEach((value: unknown, key: unknown) => {
      // important: make sure the callback is
      // 1. invoked with the reactive map as `this` and 3rd arg
      // 2. the value received should be a corresponding reactive/readonly.
      return callback.call(thisArg, wrap(value), wrap(key), observed)
    })
  }
复制代码
```

#### 处理迭代器

集合类型的迭代器方法：

- `entries()`
- `keys()`
- `values()`

`Map` 和 `Set` 都实现了 **可迭代协议**（即 `Symbol.iterator` 方法，而 **迭代器协议** 是指 一个对象实现了 `next` 方法），因此它们还可以通过 `for...of` 的方式进行遍历。

根据对 `forEach` 的处理，不难知道涉及遍历的方法，终究还是得将其对应的遍历的 **键、值** 进行响应式包裹的处理，以及进行 `track` 操作，而原本的的迭代器方法没办法实现，因此需要内部自定义迭代器协议。

```
const iteratorMethods = ['keys', 'values', 'entries', Symbol.iterator]
  iteratorMethods.forEach(method => {
    mutableInstrumentations[method as string] = createIterableMethod(
      method,
      false,
      false
    )
    省略其他代码
  })
复制代码
```

这一部分的源码涉及的内容比较多，以上只是简单的总结一下，更详细的内容可查看对应的源码内容。

## ref 函数 — 原始值的响应式

原始值指的是 **`Boolean、Number、BigInt、String、Symbol、undefined、null`** 等类型的值，我们知道用 **`Object.defineProperty`** 肯定是不支持，因为它拦截的就是对象属性的操作，都说 **`Proxy`** 比 **`Object.defineProperty`** 强，那么它能不能直接支持呢？

直接支持是肯定不能的，别忘了 `Proxy` 代理的目标也还是对象类型呀，它的强是在自己的所属领域，跨领域也是遭不住的。

因此在 `Vue3` 的 `ref` 函数中对原始值的处理方式是通过为 **原始值类型** 提供一个通过 `new RefImpl(rawValue, shallow)` 实例化得到的 **包裹对象**，说白了还是将原始值类型变成对象类型，但 `ref` 函数的参数并 **不限制数据类型**：

- **原始值类型**，`ref` 函数中会为原始值类型数据创建 `RefImpl` 实例对象（必须通过 `.value` 的方式访问数据），并且实现自定义的 `get、set` 用于分别进行 **依赖收集** 和 **依赖更新**，注意的是这里并不会通过 `Proxy` 为原始值类型创建代理对象，准确的说在 `RefImpl` 内部自定义实现的 `get、set` 就实现了对原始值类型的拦截操作，**`因为原始值类型不需要向对象类型设置那么多的捕获器`**

- **对象类型**，`ref` 函数中除了为 **对象类型** 数据创建 `RefImpl` 实例对象之外，还会通过 `reactive` 函数将其转换为响应式数据，其实主要还是为了支持类似如下的操作

  ```
  const refProxy = ref({name: 'zs'})
  refProxy.value.name = 'ls'
  复制代码
  ```

- **依赖容器 dep**，在 `ref` 类型中依赖存储的位置就是每个 `ref` 实例对象上的 `dep` 属性，它本质就是一个 `Set` 实例，触发 `get` 时往 `dep` 中添加副作用函数（依赖），触发 `set` 时从 `dep` 中依次取出副作用函数执行

**源码位置：`packages\reactivity\src\ref.ts`**

```
export function ref(value?: unknown) {
  return createRef(value, false)
}

function createRef(rawValue: unknown, shallow: boolean) {
  if (isRef(rawValue)) {
    return rawValue
  }
  return new RefImpl(rawValue, shallow)
}

class RefImpl<T> {
  private _value: T
  private _rawValue: T

  public dep?: Dep = undefined
  public readonly __v_isRef = true

  constructor(value: T, public readonly __v_isShallow: boolean) {
    this._rawValue = __v_isShallow ? value : toRaw(value)
    this._value = __v_isShallow ? value : toReactive(value)
  }

  get value() {
    // 将依赖收集到 dep 中，实际上就是一个 Set 类型
    trackRefValue(this)
    return this._value
  }

  set value(newVal) {
    // 获取原始数据
    newVal = this.__v_isShallow ? newVal : toRaw(newVal)

    // 通过 Object.is(value, oldValue) 判断新旧值是否一致，若不一致才需要进行更新
    if (hasChanged(newVal, this._rawValue)) {
      // 保存原始值
      this._rawValue = newVal
      // 更新为新的 value 值
      this._value = this.__v_isShallow ? newVal : toReactive(newVal)
      // 依赖更新，从 dep 中取出对应的 effect 函数依次遍历执行
      triggerRefValue(this, newVal)
    }
  }
}

// 若当前 value 是 对象类型，才会通过 reactive 转换为响应式数据
export const toReactive = <T extends unknown>(value: T): T =>
  isObject(value) ? reactive(value) : value
复制代码
```

# Vue3 如何进行依赖收集？

在 `Vue2` 中依赖的收集方式是通过 `Dep` 和 `Watcher` 的 **观察者模式** 来实现的，是不是还能想起初次了解 `Dep` 和 `Watcher` 之间的这种 **`剪不断理还乱`** 的关系时的心情 ......

> 关于 **设计模式** 部分感兴趣可查看 ***\*常见 JavaScript 设计模式 — 原来这么简单\****[32] 一文，里面主要围绕着 `Vue` 中对应的设计模式来进行介绍，相信会有一定的帮助

**依赖收集** 其实说的就是 `track` 函数需要处理的内容：

- 声明 `targetMap` 作为一个容器，用于保存和当前响应式对象相关的依赖内容，本身是一个 `WeakMap` 类型

- - 选择 `WeakMap` 类型作为容器，是因为 `WeakMap` 对 **键**（对象类型）的引用是 **弱类型** 的，一旦外部没有对该 **键**（对象类型）保持引用时，`WeakMap` 就会自动将其删除，即 **能够保证该对象能够正常被垃圾回收**
  - 而 `Map` 类型对 **键** 的引用则是 **强引用** ，即便外部没有对该对象保持引用，但至少还存在 `Map` 本身对该对象的引用关系，因此会导致该对象不能及时的被垃圾回收

- 将对应的 **响应式数据对象** 作为 `targetMap` 的 **键**，存储和当前响应式数据对象相关的依赖关系 `depsMap`（属于 `Map` 实例），即 `depsMap` 存储的就是和当前响应式对象的每一个 `key` 对应的具体依赖

- 将 `deps`（属于 `Set` 实例）作为 `depsMap` 每个 `key` 对应的依赖集合，因为每个响应式数据可能在多个副作用函数中被使用，并且 `Set` 类型用于自动去重的能力

**可视化结构如下：**

![图片](https://mmbiz.qpic.cn/mmbiz/pfCCZhlbMQTn1pZ8MnehWiba2cp9UdjBy6GqUl3aRXb39HSY558YYOicGDJJHnQqwo9Cm27e1gf7icuKcdCKYMiaAA/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

**源码位置：`packages\reactivity\src\effect.ts`**

```
const targetMap = new WeakMap<any, KeyToDepMap>()

export function track(target: object, type: TrackOpTypes, key: unknown) {
  // 当前应该进行依赖收集 且 有对应的副作用函数时，才会进行依赖收集
  if (shouldTrack && activeEffect) {
    // 从容器中取出【对应响应式数据对象】的依赖关系
    let depsMap = targetMap.get(target)
    if (!depsMap) {
      // 若不存在，则进行初始化
      targetMap.set(target, (depsMap = new Map()))
    }

    // 获取和对【应响应式数据对象 key】相匹配的依赖
    let dep = depsMap.get(key)
    if (!dep) {
      // 若不存在，则进行初始化 dep 为 Set 实例
      depsMap.set(key, (dep = createDep()))
    }

    const eventInfo = __DEV__
      ? { effect: activeEffect, target, type, key }
      : undefined

    // 往 dep 集合中添加 effect 依赖
    trackEffects(dep, eventInfo)
  }
}

export const createDep = (effects?: ReactiveEffect[]): Dep => {
  const dep = new Set<ReactiveEffect>(effects) as Dep
  dep.w = 0
  dep.n = 0
  return dep
}
复制代码
```

# 最后

以上就是针对 `Vue3` 中对不同数据类型的处理的内容，无论是 `Vue2` 还是 `Vue3` 响应式的核心都是 **数据劫持/代理、依赖收集、依赖更新**，只不过由于实现数据劫持方式的差异从而导致具体实现的差异，在 `Vue3` 中值得注意的是：

- **普通对象类型** 可以直接配合 `Proxy` 提供的捕获器实现响应式

- **数组类型** 也可以直接复用大部分和 **普通对象类型** 的捕获器，但其对应的查找方法和隐式修改 `length` 的方法仍然需要被 **重写/增强**

- 为了支持 **集合类型** 的响应式，也对其对应的方法进行了 **重写/增强**

- **原始值数据类型** 主要通过 `ref` 函数来进行响应式处理，不过内容不会对 **原始值类型** 使用 `reactive（或 Proxy）` 函数来处理，而是在内部自定义 `get value(){}` 和 `set value(){}` 的方式实现响应式，毕竟原始值类型的操作无非就是 **读取** 或 **设置**，核心还是将 **原始值类型** 转变为了 **普通对象类型**

- - `ref` 函数可实现原始值类型转换为 **响应式数据**，但 `ref` 接收的值类型并没只限定为原始值类型，若接收到的是引用类型，还是会将其通过 `reactive` 函数的方式转换为响应式数据

 