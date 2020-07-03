---
title: Vue 与 React 的区别
date: 2020-06-29 10:19:26
tags: vue, react
---

> Vue 与 React 有什么区别？

这是前端开发同学面试时经常遇到的问题。

我也不例外，我最开始接触的是 React，对 Vue 的理解一直比较片面。感觉 Vue 要学很多 html 指令，很不习惯，也没觉得 Vue 比 React 有什么优势。

直到现在，使用了 Vue 一年之后，对 Vue 有了更多感受，也消除了一些刻板印象。

首先，这两个框架，其实是非常相似的，都有以下特性：
1. 响应式（Reactive）。两个框架都是一种类似 VM 的架构，将状态从视图层分离出来，开发者只需要关注业务逻辑，不需要直接操作 DOM 。当应用发生改变时，我们只需要更新状态即可，框架会自动帮我们重新渲染页面。
2. 组件化（Composable）。一个页面，可以拆分成一棵嵌套的组件树，我们只需要开发一个个组件即可，同一个组件可以在多个地方使用，这样就提升了代码的复用性和可维护性。
3. 使用了 Virtual DOM。框架在操作真实 DOM 之前，会先在内存中生成虚拟 DOM，最后再批量操作真实 DOM，以提高性能。

它们都是非常优秀的单页应用（SPA）开发框架，我个人理解，他们的差别主要是实现方式不同，主要体现在响应式和组件化上面。

## 响应式原理

Vue 的响应式，是使用观察者模式实现的。Vue 会遍历 data 状态对象，使用 Object.defineProperty() 将每个属性都转换为 getter/setter。

每个 Vue 组件实例都对应一个 watcher 实例，在组件渲染（render）时，会生成一颗虚拟 DOM 树，watcher 实例会记录哪些子组件用到了（getter）哪些数据 property。当数据发生改变时，会触发 setter 方法，watcher实例会通知所有用到了这个数据的子组件，调用该子组件的 updateComponent 方法。

![](https://cn.vuejs.org/images/data.png)

例如，一个 TodoList 的组件，代码结构如下：

```vue
<template>
  <div>
    <todo-items :items="items" />
    <add-todo v-model="text" />
  </div>
</template>

<script>
export default {
  data() {
    return {
	  text: '',
	  items: []
    };
  },
};
</script>  
```

在这个组件渲染过程中，watcher 会记录 TodoItems 子组件用到了 items 数据属性，AddTodo  子组件用到了 text 数据属性。

假设 AddTodo 组件里面是一个 input 元素，当在 input 输入内容是，会修改 text 数据属性，Vue 就会通知用到了 text 数据属性的所有子组件，这里只有 AddTodo 一个组件，所以 Vue 就只更新 AddTodo 这一个组件。