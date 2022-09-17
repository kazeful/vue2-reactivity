---
theme: seriph
background: https://source.unsplash.com/collection/94734566/1920x1080
class: text-center
highlighter: shiki
lineNumbers: false
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
drawings:
  persist: false
title: Vue.js 响应式系统
---

# Vue.js v2
<h4>响应式系统</h4>

<div class="pt-12">
  <span @click="$slidev.nav.next" class="px-2 py-1 rounded cursor-pointer" hover="bg-white bg-opacity-10">
   什么是响应式 <carbon:arrow-right class="inline"/>
  </span>
</div>

<div class="abs-bl m-6">
  <!-- <div class="mb-3 uppercase tracking-widest font-500"> Wind </div> -->
  <div class="text-md opacity-50">2022-07-11</div>
</div>

<div class="abs-br m-6 flex gap-2">
  <a href="https://github.com/slidevjs/slidev" target="_blank" alt="GitHub"
    class="text-xl icon-btn opacity-50 !border-none !hover:text-white">
    <carbon-logo-github />
  </a>
</div>

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

---

# 1. 什么是响应式
<div class="leading-normal">
响应性是 Vue的一个核心特性，用于监听视图中绑定的数据，当数据发生改变时视图自动更新。

只要状态发生改变，系统依赖部分发生自动更新就可以称为响应性。

在 web应用中，数据的变化如何响应到DOM中，就是Vue解决的问题。
</div>

<v-clicks>

<div>

```js
let title = 'hello'

```

</div>

<div>

```html
<span>hello</span>
```

</div>

<div>

```js
title = 'hello world'

```

</div>

<div>

```html
<span>hello world</span>
```

</div>

</v-clicks>

<style>
h1 {
  background-color: #2B90B6;
  background-image: linear-gradient(45deg, #4EC5D4 10%, #146b8c 20%);
  background-size: 100%;
  -webkit-background-clip: text;
  -moz-background-clip: text;
  -webkit-text-fill-color: transparent;
  -moz-text-fill-color: transparent;
}
</style>

---

# 1.1 举个例子
<div class="leading-normal">
假设我们有个需求，b永远等于a的十倍，如果使用命令式编程，可以很简单实现，

可以像下面这样实现，但是当我们把a设置成4时，b还是等于30
</div>

```js
let a = 3;
let b = a * 10；
console.log(b) // 30
a = 4
console.log(b) // 30 
```

<div class="leading-normal">
为了让b等于a的10倍，那我们需要重新设置b的值，像下面代码
</div>

```js
let a = 3;
let b = a * 10；
console.log(b) // 30
a = 4;
b = a * 10; // 新增代码
console.log(b) // 40 
```

---

# 1.2 神奇的函数
<div class="leading-normal">
假设我们有一个神奇函数叫onAchange，它接收一个函数并且当a改变时自动被执行，这时候可以对b重新赋值

那上面的问题就解决了，那这个函数如何实现是问题的关键
</div>

```js
onAchange(() => {
  b = a * 10
})
```
---

<div class="leading-normal">
再举个更贴合web开发的例子: 

下面代码同样有一个神奇函数onStateChange，它会在state改变的时候自动运行

那我们只要在函数中编写dom操作的代码，就可以实现dom的自动更新了
</div>

```html
<!-- DOM元素 -->
<span class="title"></span>
```

```js
// 神奇函数，当state值改变会自动重新运行
onStateChange(() => {
  document.querySelector('.title').textContent = state.title
})
```

如果你有react开发经验，会发现这和react修改数据调用方法是一样的

```js
onStateChanged(() => {
  view = render(state) // 这里抽象的视图渲染伪代码，可以简单的理解为在更新视图
})

setState({ a: 5 })
```

---

# 2. 深入响应式原理
<div class="leading-normal">
现在是时候深入一下了！Vue 最独特的特性之一，是其非侵入性的响应式系统。

数据模型仅仅是普通的 JavaScript 对象。而当你修改它们时，视图会进行更新。

这使得状态管理非常简单直接，不过理解其工作原理同样重要，这样你可以避开一些常见的问题。

现在开始实现一个 mini 版的响应系统，感受一下 Vue 响应式系统的底层实现的细节。

下面我们将分两步实现它

</div>

<style>
h1 {
  background-color: #2B90B6;
  background-image: linear-gradient(45deg, #4EC5D4 10%, #146b8c 20%);
  background-size: 100%;
  -webkit-background-clip: text;
  -moz-background-clip: text;
  -webkit-text-fill-color: transparent;
  -moz-text-fill-color: transparent;
}
</style>

---

# 2.1 getter和setter

在这之前，我们需要了解一个api，即ES5的 [Object.defineProperty](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty) ，它提供了监听属性变更的功能。

<div flex="~ gap-2">

<div>

```js
const obj = { foo: 123 }
convert(obj) 
obj.foo // 打印: 'getting key "foo": 123'
obj.foo = 234 // 打印: 'setting key "foo" to 234'
obj.foo // 打印: 'getting key "foo": 234'
```

<h5>
顺便一提，Object.defineProperty 是 ES5 中一个无法 shim 的特性，这也就是 Vue 不支持 IE8 以及更低版本浏览器的原因。</h5>

</div>

<div>

```js {all|2-16|4-15|all}
function convert (obj) {
  // Object.keys获取对象的所有key值，通过forEach对每个属性进行修改
  Object.keys(obj).forEach(key => {
    // 保存属性初始值
    let internalValue = obj[key]
    Object.defineProperty(obj, key, {
      get () {
        console.log(`getting key "${key}": ${internalValue}`)
        return internalValue
      },
      set (newValue) {
        console.log(`setting key "${key}" to: ${newValue}`)
        internalValue = newValue
      }
    })
  })
}
```
</div>
</div>

---

# 2.2 实现一个依赖跟踪 class Dep（订阅发布模式）
<div class="leading-normal">

类里有一个叫 depend 方法，该方法用于收集依赖；还有一个 notify 方法，该方法用于触发依赖项的执行。也就是说只要在之前使用 dep 方法收集的依赖项，当调用 notfiy 方法时会被触发执行。

下面是 class Dep 期望达到的效果：

调用dep.depend方法收集收集依赖，当调用dep.notify方法，控制台会再次输出updated语句
</div>

```js
const dep = new Dep()

autorun(() => {
  dep.depend()
  console.log('updated')
})
// 打印: "updated"

dep.notify()
// 打印: "updated"
```

autorun函数是接收一个函数，这个函数帮助我们创建一个响应区，当代码放在这个响应区内，就可以通过dep.depend方法注册依赖项

---

<div grid="~ cols-2 gap-4">
<div>

```js {all|4|9|14|all}
window.Dep = class Dep {
  constructor () {
    // 订阅任务队列，方式有相同的任务，用Set数据结构简单处理
    this.subscribers = new Set()
  }
	// 用于注册依赖项
  depend () {
    if (activeUpdate) {
      this.subscribers.add(activeUpdate)
    }
  }
	// 用于发布消息，触发依赖项重新执行
  notify () {
    this.subscribers.forEach(sub => sub())
  }
}
```

</div>
<div>

```js {all|1|9|4-8|all}
let activeUpdate = null

function autorun (update) {
  const wrappedUpdate = () => {
    activeUpdate = wrappedUpdate
    update()
    activeUpdate = null
  }
  wrappedUpdate()
}
```

</div>
</div>

---

# 2.3 实现迷你观察者
<div class="leading-normal">
我们将2.1和2.2的两个练习整合到一起，实现一个小型的观察者，通过在getter和setter中调用depend方法和notfiy方法，就可以实现自动更新数据的目的了，这也是Vue实现自动更新的核心原理。

期望实现的调用效果：
</div>

```js
const state = {
  count: 0
}

observe(state)

autorun(() => {
  console.log(state.count)
})
// 打印"count is: 0"

state.count++
// 打印"count is: 1"
```

---

<div grid="~ cols-2 gap-4">
<div>

```js
class Dep {
  constructor () {
    this.subscribers = new Set()
  }

  depend () {
    if (activeUpdate) {
      this.subscribers.add(activeUpdate)
    }
  }

  notify () {
    this.subscribers.forEach(sub => sub())
  }
}
```

```js
let activeUpdate = null

function autorun (update) {
  const wrappedUpdate = () => {
    activeUpdate = wrappedUpdate
    update()
    activeUpdate = null
  }
  wrappedUpdate()
}
```

</div>

<div>

```js {all|2-22|4-21|7-11|13-20|all}
function observe (obj) {
  Object.keys(obj).forEach(key => {
    let internalValue = obj[key]

    const dep = new Dep()
    Object.defineProperty(obj, key, {
      // 在getter收集依赖项，当触发notify时重新运行
      get () {
        dep.depend()
        return internalValue
      },

      // setter用于调用notify
      set (newVal) {
        const changed = internalValue !== newVal
        internalValue = newVal
        if (changed) {
          dep.notify()
        }
      }
    })
  })
  return obj
}
```
</div>
</div>

---

# 总结
<div class="leading-normal">

当你把一个普通的 JavaScript 对象传入 Vue 实例作为 data 选项，Vue 将遍历此对象所有的 property，并使用 Object.defineProperty 把这些 property 全部转为 getter/setter。

这些 getter/setter 对用户来说是不可见的，但是在内部它们让 Vue 能够追踪依赖，在 property 被访问和修改时通知变更。

每个组件实例都对应一个 watcher 实例，它会在组件渲染的过程中把“接触”过的数据 property 记录为依赖。

之后当依赖项的 setter 触发时，会通知 watcher，从而使它关联的组件重新渲染。
</div>

---

<img src="https://v2.cn.vuejs.org/images/data.png" class="w-200" />

---

# 参考文献
- [vue2官方文档](https://v2.cn.vuejs.org/v2/guide/reactivity.html)
- [尤雨溪教你写vue](https://www.bilibili.com/video/BV1d4411v7UX?p=2)
