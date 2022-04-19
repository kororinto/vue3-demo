> vue3核心源码地址：[https://github.com/vuejs/core](https://github.com/vuejs/core)  
此次源码分析对应版本为3.2.25  日期为2022.1.23
## 一、reactive
源码文件路径：packages\reactivity\src\reactive.ts  
过程：  
1. 如果被readonly处理过，直接返回该数据，因为被readonly处理过也是Proxy对象  
否则对该数据创建响应式对象  
2. 如果该数据不是对象类型 直接返回该数据  
3. 如果该数据是响应式对象 除非是用readonly处理过的响应式对象 直接返回该对象，这里说明响应式对象也可以用readonly方法再次处理  
4. 如果可以在Map中找到该对象（被处理过的响应式对象会存入一个Map中） 直接返回该对象  
5. 如果该对象不可扩展或被标记为跳过 直接返回该对象，说明被preventExtensions、seal、freeze处理过的对象不能成为响应式
6. 使用 new Proxy 创建响应式对象
7. 原对象作为key，响应式对象作为value加入到对应Map中保存

#### 一些需要注意的点

##### 1. [WeakMap](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/WeakMap) 和 [Map]((https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Map)) 的区别
WeakMap 的 key 只能是对象且不可枚举，key对value是弱引用，可正确的被垃圾回收机制回收  
Map 的 key 可以是任意类型，除非使用clear，否则key会一直作为被引用的存在  
##### 2. [isExtensible](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/isExtensible) 函数判断对象是否可扩展
preventExtensions、seal、freeze 方法都可以标记一个对象为不可扩展  
[preventExtensions](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/preventExtensions)：使对象不能添加新属性  
[seal](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/seal)：使对象不能添加新属性且不能删除已有属性  
[freeze](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/freeze)：使对象不能添加新属性且不能删除已有属性且不能修改已有属性值
##### 3. [toString](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/toString) 检测类型
源码文件路径：packages\shared\src\index.ts  
需要用call或apply调用，返回 [object RawType]  
```javascript
export const toTypeString = (value: unknown): string =>
  objectToString.call(value)

export const toRawType = (value: unknown): string => {
  // extract "RawType" from strings like "[object RawType]"
  return toTypeString(value).slice(8, -1)
}
```
## 二、Proxy处理对象
当判断该数据可以被[Proxy]((https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy))处理后，传入一个handler对其处理  
注意：下面是非集合（非Set、Map）类型对象的handler中get、set方法详解
#### ProxyHandler.get
源码文件路径：packages\reactivity\src\baseHandlers.ts  
过程：  
1. 如果访问的属性key是否一些标记（与isReactive、isReadonly、isShallow、toRaw有关）  
则返回对应结果，这些结果不需要关心，是vue中的API所用到的，我们使用对应API即可，下面会提到
2. 如果是数组，会将一些数组方法做对应处理，保证用户对Proxy对象进行数组操作时跟操作数组一样，并对部分方法加入一些响应式更新视图的逻辑，让用户操作数组时也会响应式更新视图
3. 使用Reflect.get获取到对应value
4. 如果是一些内置的Symbol或者__proto__、__v_isRef、__isVue这些类似内置的属性 直接返回对应value
5. 如果是浅监视（shallow） 直接返回对应value
6. 如果不是被readonly处理过的 则添加响应式更新视图逻辑，因为只读数据不能修改，也就没有必要响应式更新视图
7. 如果是该value是ref且原对象不是数组，会直接返回value.value，而不是ref对象，因为ref对象要通过 .value取值  
但如果是数组的话，通过下标访问数组元素，还是返回的还是ref对象
```javascript
const reactiveObj = reactive({ age: ref(0) })
const reactiveArr = reactive([ref(0)])
console.log(reactiveObj.age)      // 0
console.log(reactiveArr[0].value) // 0
```
8. 如果该value是对象且原对象不是readonly的，则执行 ```reactive(value)``` 再次对其进行Proxy处理

我们来验证一下上述过程中的第一条，有关源码如下：  
```javascript
/*
  vue用到的一些key，应避免使用这些key作为reactive对象的属性
  否则会使isReactive、isReadonly、toRaw 等方法失效
*/
export const enum ReactiveFlags {
  SKIP = '__v_skip',
  IS_REACTIVE = '__v_isReactive',
  IS_READONLY = '__v_isReadonly',
  IS_SHALLOW = '__v_isShallow',
  RAW = '__v_raw'
}

/*
 以非集合响应式对象为例
 proxy的handler的get中第一个过程：
*/

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
```
为了验证上述四种 ```if``` 分支，可以像如下这样访问  
注意：在ts中这种写法会在编译前报错，因为ts会检测reactive函数的返回值类型，不允许访问类型中没有的属性，但依然可以运行
```javascript
<script setup lang="ts">
import { reactive } from 'vue'

interface Person {
  name: string
  age: number
}

// debugger
const state = reactive<Person>({ name: 'dx', age: 23 })
console.log(
  state.name            // 'dx'
  state.__v_isReactive, // ture
  state.__v_isReadonly, // false
  state.__v_isShallow,  // false
  state.__v_raw         // { name: 'dx', age: 23 }
)
</script>
``` 
这些访问方法vue也暴露了出来，应正确使用：  
```javascript
console.log(
  isReactive(state), // true
  isReadonly(state), // false
  isShallow(state),  // false
  toRaw(state)       // { name: 'dx', age: 23 }
)
```
#### ProxyHandler.set
源码文件路径：packages\reactivity\src\baseHandlers.ts  
过程：  
1. 通过key获取oldValue
2. 如果不是数组，oldValue是ref对象，value不是ref对象，会对oldValue.value赋值（也就是尤雨溪说的包装和解包装），不需要我们太麻烦的赋值，否则代码多可能会混乱，记不清哪个属性用 .value
```javascript
const reactiveObj = reactive({ age: ref(0) })
reactiveObj.age = 1
```
3. 使用Reflect.set设置对应属性值
4. 如果对象是原型链上的，不触发更新视图的操作 否则更新视图（暂时没有找到合适场景的例子）
## 三、ref
源码文件路径：packages\reactivity\src\ref.ts  
过程：
1. 如果已经是ref对象 直接返回对应value
```javascript
// 判断方法用到的key在reactive也有用到
export function isRef<T>(r: Ref<T> | unknown): r is Ref<T>;
export function isRef(r: any): r is Ref {
  return Boolean(r && r.__v_isRef === true)
}
```
2. new生成RefImpl类的实例并将其返回  
#### RefImpl
构造函数中将通过toRaw获取到原数据、toReactive获取到响应式数据（如果是对象的话） 并保存起来
```javascript
constructor(value: T, public readonly __v_isShallow: boolean) {
  this._rawValue = __v_isShallow ? value : toRaw(value)
  this._value = __v_isShallow ? value : toReactive(value)
}
```
起初我心想，当我们使用ref函数创建声明变量时，为什么要对其进行toRaw呢
```javascript
const countRef = ref({})
console.log(countRef) // RefImpl {...}
// 此时进入到构造函数中的value并不是响应式对象 只是普通的 {}
```  
后来发现，因为有可能将ref对象作为reactive对象中的属性去用，此时就会被Proxy处理
```javascript
const reactiveData = reactive({ count: countRef })
console.log(reactiveData)       // Proxy {count: RefImpl}
console.log(reactiveData.count) // Proxy {}
```  
当通过 .value取值时进入的是类中的get方法  
返回构造函数中通过toReactive获取到响应式数据，并进行追踪追踪  
说明当我们声明一个简单类型的ref对象时，并不会被vue追踪，而且被使用时才会被追踪
```javascript
get value() {
  trackRefValue(this)
  return this._value
}
```
当对 .value进行赋值操作时进入到set方法，set中的toRaw也是一个道理  

```javascript
set value(newVal) {
  newVal = this.__v_isShallow ? newVal : toRaw(newVal)
  if (hasChanged(newVal, this._rawValue)) {
    this._rawValue = newVal
    this._value = this.__v_isShallow ? newVal : toReactive(newVal)
    triggerRefValue(this, newVal)
  }
}
```
需要注意的是，```this._rawValue``` 是存储原数据，在set中通过对比newVal来判断是否修改，如果值没有变化就不会再进行操作  
如果是浅监视 ```this.__v_isShallow``` 为ture, 就不需要这些处理，只需追踪这一个值的变化
## 四、computed
源码文件路径：packages\reactivity\src\computed.ts  
过程：
1. 判断第一个是否为函数 是则作为getter 否则就是对象options，并将options对象中的get、set作为getter、setter  
对应computed的两种用法，第一个参数传一个函数作为getter或者第一个参数传一个对象，对象里的get、set作为getter、setter
2. new生成ComputedRefImpl类的实例
3. 如果传了第二个参数，则在开发环境中把第二个参数中的onTrack、onTrigger挂载到该计算属性上
```javascript
if (__DEV__ && debugOptions && !isSSR) {
  cRef.effect.onTrack = debugOptions.onTrack
  cRef.effect.onTrigger = debugOptions.onTrigger
}
```
注意：computed的第二个参数是3.2.x新增的debuggerOptions，该参数类型如下
```javascript
export interface DebuggerOptions {
  onTrack?: (event: DebuggerEvent) => void
  onTrigger?: (event: DebuggerEvent) => void
}
```
#### ComputedRefImpl
构造函数中：
1. 判断传入的第三个参数决定该计算属性是否是只读的，第三个参数就是有没有setter  
2. 第四个参数isSSR跟服务端渲染以及缓存有关
```javascript
const cRef = new ComputedRefImpl(getter, setter, onlyGetter || !setter, isSSR)
```
3. 使用ReactiveEffect，使获得的计算属性具有响应式  
  
类中一些成员变量的作用  
 ```public readonly __v_isRef = true```，说明计算属性也是ref对象  
 ```public _dirty = true```，这个值在调用get后置为false，在这个值为true时才会更新，说明当computed依赖的变量发生变化时，并不会马上更新computed值，而是调用get时更新，非常节约性能！
## 五、watch和watchEffect
源码文件路径：packages\runtime-core\src\apiWatch.ts  
watch和watchEffect在对参数做一些判断处理后都是调用的同一个叫doWatch的方法，所以把他们放在一起讲  
#### watch
watch中一共重载了四个类型定义：侦听多个数据源、侦听多个只读数据源、侦听单个数据源、侦听响应式对象。在开发环境中如果第二个参数不是函数，控制台会warn提示，传入对应参数调用doWatch
#### watchEffect
watchEffect中直接调用了doWatch，需要注意的是第二个参数options可以设置flush为```'post'```、```'sync'```、```'re'```
```javascript
export interface WatchOptionsBase extends DebuggerOptions {
  flush?: 'pre' | 'post' | 'sync'
}
```
在3.2+中新增了两个API，watchPostEffect和watchSyncEffect，对应就是watchEffect带了```{flush: 'post'}```和```{flush: 'sync'}```参数
#### doWatch
过程：
1. 分别对第一个参数是ref对象、reactive对象、Array、Function做不同的处理来获取数据源：  
ref对象时getter返回source.value  
reactive对象时返回source  
Array时getter返回被单个情况处理后的数组
Function时，分别对watch和watchEffect进入的doWatch做不同处理
2. 如果有第二个参数cb，说明是从watch进入的doWatch，判断options中的deep为true时会遍历所有对象中的响应式对象
3. 对options中的flush参数三个值分别做处理，通过一个调度函数让watch/watchEffect在对应的时机去执行回调
4. 初次执行doWatch，对于watchEffect会根据flush参数来执行追踪，对于watch根据immediate参数是否执行追踪
5. 返回一个函数，函数体中会停止对这些数据源追踪，说明我们可以通过调用watch/watchEffect函数返回值去停止侦听
```javascript
// vue源码
return () => {
  effect.stop()
  if (instance && instance.scope) {
    remove(instance.scope.effects!, effect)
  }
}

// 使用
const stopWatch = watch(source, callback, options)
stopWatch()
```
其实涉及到很多重要且复杂的函数：
1. ```job: SchedulerJob```，与watch中获取数据源相关、watchEffect中执行回调有关
2. ```scheduler: EffectScheduler```，调用函数，让job在合适的时机去执行
3. ```ReactiveEffect```，侦听函数用到的一系列响应式操作对象的类，通过他可以巧妙的和响应式数据联系到一起

三者关系为：job函数被scheduler调度函数包装后通过```new ReactiveEffect()```生成一个effect，effect对象上的函数对应获取数据源、执行传入的回调、停止侦听

##### job: SchedulerJob
函数体流程：
1. 如果当前watch已被停止则直接返回
2. 当有cb参数时（watch进入），执行传入的回调获得数据源并赋值给newValue  
如果deep为true、数据源为reactive对象、简单类型ref对象、数组类型中有元素发生变化的情况下，会异步的循环执行传入的回调函数  
3. 没有cb参数时，是watchEffect，直接执行回调函数

##### scheduler: EffectScheduler
1. flush为'sync'时，```schedubler = job```
2. flush为'post'时，把job加入到postFlush执行队列，如果组件suspense，会把组件异步执行的操作一并加入到队列中
3. flush为'pre'时，如果此时组件未挂载或mouted完毕，把job加入到pre执行队列，使job在组件下一次beforeUpdate之前调用，否则直接执行一次job

##### ReactiveEffect
这个类做了很多关于effect的deps（数据执行保护）  
构造函数：把实例effect记录到当前响应式链的中  
run:
1. 如果effect被停止过，则直接执行传入的回调并返回
2. 如果effect栈为空或没有这个effect，则会执行传入的回调并返回，并对effect栈做一些限制处理，防止追踪链太深影响整个项目
stop：停止追踪并在deps中清除这个effect，将active属性置为false，标记为被停止过
***
<strong>注意：当job执行到我们包含响应式对象的语句时，就会进入到Proxy代理后的get方法中，就会进入到track函数并触发trigger函数，watch/watchEffec就是这样把响应式连接起来，下面讲解追踪原理（前面几个响应式对象中也包括的有，之前略过了）</strong>

## 六、track
源码文件路径：packages\reactivity\src\effect.ts
