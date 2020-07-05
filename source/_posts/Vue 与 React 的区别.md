---
title: Vue 与 React 的区别
date: 2020-06-29 10:19:26
tags: vue, react
---

> Vue 与 React 有什么区别？

这是前端开发同学面试时经常遇到的问题。

我也不例外，我最开始接触的是 React，对 Vue 的理解一直比较片面。感觉 Vue 要学很多 html 指令，很不习惯，也没觉得 Vue 比 React 有什么优势。

直到现在，使用了 Vue 一年之后，对 Vue 有了更多感受，也消除了一些刻板印象。

首先，这两个框架都是非常优秀的单页应用（SPA）开发框架，它们其实非常的相似，都有以下特性：

1. 响应式（Reactive）。两个框架都是一种类似 VM 的架构，将状态从视图层分离出来，开发者只需要关注业务逻辑，不需要直接操作 DOM 。当应用发生改变时，我们只需要更新状态即可，框架会自动帮我们重新渲染页面。
2. 组件化（Composable）。一个页面，可以拆分成一棵嵌套的组件树，我们只需要开发一个个组件即可，同一个组件可以在多个地方使用，这样就提升了代码的复用性和可维护性。
3. 使用了 Virtual DOM。框架在操作真实 DOM 之前，会先在内存中生成虚拟 DOM，最后再批量操作真实 DOM，以提高性能。

我个人理解，它们的差别主要是这些特性的实现方式不同，它们的响应式原理不同，定义组件的方式有也一些差别。Vue 推荐使用 template 的  方式定义组件，React 更喜欢 jsx，React 16 之后新增了 React Hooks 特性，对函数组件有了更好的支持。

## 响应式原理  不同

Vue 的响应式，是使用观察者模式实现的。Vue 会遍历 data 数据对象，使用 Object.defineProperty() 将每个属性都转换为 getter/setter。

每个 Vue 组件实例都对应一个  watcher  实例，在组件渲染（render）过程中时，watcher 实例会记录哪些子组件用到了（getter）哪些数据属性。当数据属性发生改变时，会触发 setter 方法，watcher 实例会通知所有用到了这个数据属性的子组件，调用该子组件的 updateComponent 方法。

![](https://cn.vuejs.org/images/data.png)

例如，一个 TodoList 组件，代码结构如下：

```html
<template>
  <div>
    <todo-items :items="items" />
    <add-todo v-model="text" :count="items.length + 1" />
  </div>
</template>
```

在这个组件渲染过程中，watcher 会记录 TodoItems 子组件用到了 items 数据属性，AddTodo 子组件用到了 text 和 items 数据属性。

当 text 数据属性发生改变时，watcher 就会通知用到了 text 数据属性的所有子组件，这里只有 AddTodo 一个组件，所以 Vue 就只更新 AddTodo 这一个组件。

同理，当 items 数据属性发生改变时，Vue 就会更新 TodoItems 和 AddTodo 两个组件。

---

React 的响应式，是  使用 diff 算法实现的。React 在 state 或 props 改变时，会调用 render() 方法，生成一个虚拟 DOM 树，React 会将这棵树与上一次生成的树进行比较，找出其中的差异，并更新差异的部分。

比较两棵树，找出其中的差异，并生产一个做小操作数。这个算法的复杂度比较高，是 O(n 3 )，React 为了提高性能， 提出了一种复杂度为 O(n) 的优化算法， 这个算法有两个重要的假设：

1. 两个不同类型的  元素会产生出不同的树。当根节点元素类型发生改变时，React 会销毁旧节点创建新节点。比如当一个元素从 `<Button>` 变成 `<div>`，或者 `<ComponentA>` 变成 `<ComponentB>`。当元素类型相同时，React 会保留 DOM 节点，仅比较并更新有改变的属性。如果有子节点，React 会递归比较  子节点。
2. 当元素类型相同时，开发者可以使用 key 属性来标识元素的唯一性。比如一个列表， 有多个相同类型的子节点，当子节点顺序发生改变是，如果没有一个唯一标识，就有可能产生比较多的操作数。React 引入 key 属性作为唯一标识，就是为了解决这个问题。

关于 React 的  diff 算法，官方文档写的很清楚 ，我就不多说了。 详情请看：[https://zh-hans.reactjs.org/docs/reconciliation.html](https://zh-hans.reactjs.org/docs/reconciliation.html)

## Vue 可以直接修改状态，React 不可以

Vue 可以直接修改 data 数据  属性，React 必须通过 setState() 方法更新状态。

这个区别其实也是因为它们的响应式原理不同。

Vue 数据属性变更时，会自动通知  所有依赖这个属性的子组件，调用  它们的 updateComponent 方法更新组件，所以 Vue 可以直接修改数据属性。

React 调用 setState() 方法，组件  的 render() 方法会自动执行，重新渲染页面。但是直接修改 state，不会调用 render() 方法，所以组件就不会正确更新，这个时候可以调用 forceUpdate() 强制更新组件。但是尽量不要用 forceUpdate() 方法，因为 setState() 更新组件是异步的，会自动将一个时间循环里的多个 setState() 状态合并，以优化性能，所以正常情况  应该使用 setState() 更新状态。

## React 推荐使用不可变的数据

所谓不可变的数据，就是  当我们要改变一个  数据对象时，不要直接修改原数据对象，而是返回一个新的数据对象。比如使用 Object.assign() 方法修改数据属性:

```javascript
const data = {
  fontSize: 14,
  color: "black"
};

const newData = Object.assign({}, data, { color: "blue" });
```

之所以推荐使用不可变的数据，一个原因  是使用不可变的数据，可以更容易的实现“时间旅行 ”功能。但是更重要的一个  原因是可以更容易的实现 pure component。
