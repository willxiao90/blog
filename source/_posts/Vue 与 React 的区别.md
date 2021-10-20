---
title: Vue 与 React 的区别
date: 2020-06-29 10:19:26
tags: [vue, react]
---

> Vue 与 React 有什么区别？

这是前端开发同学面试时经常遇到的问题。

我最开始接触的是 React，对 Vue 的理解一直比较片面，感觉 Vue 要学很多 html 指令，很不习惯，也没觉得 Vue 比 React 有什么优势。

直到现在，使用了 Vue 一年之后，对 Vue 有了更多感受，也消除了一些刻板印象。

首先，这两个框架都是非常优秀的单页应用（SPA）开发框架，它们其实非常相似，都有以下特性：

1. 响应式（Reactive）：两个框架都是一种类似 VM 的架构，将状态从视图层分离出来，开发者只需要关注业务逻辑，不需要直接操作 DOM 。当应用发生改变时，我们只需要更新状态即可，框架会自动帮我们重新渲染页面。
2. 组件化（Composable）：一个页面，可以拆分成一棵嵌套的组件树，我们只需要开发一个个组件即可，同一个组件可以在多个地方使用，这样就提升了代码的复用性和可维护性。
3. Virtual DOM：框架在操作真实 DOM 之前，会先在内存中生成虚拟 DOM，最后再批量操作真实 DOM，以提高性能。

至于它们的区别，我个人理解，最大的有以下三点：
1. 响应式原理不同；
2. Vue 推荐使用模版的方式定义组件，React 推荐使用 JSX；
3. React 推荐使用不可变的数据；

当然，它们肯定还有其他差别，比如代码实现上的差别、状态管理库的区别等等。但是上面几点是它们比较大的差别，是框架有意为之的。

## 一、响应式原理不同

Vue 使用观察者模式自动跟踪数据的变化，自动更新组件。

Vue 会遍历 data 数据对象，使用 Object.defineProperty() 将每个属性都转换为 getter/setter。每个 Vue 组件实例都有一个对应的  watcher  实例，在组件初次渲染（render）时，会记录组件用到了（调用 getter）哪些数据。当数据发生改变时，会触发 setter 方法，并通知所有依赖这个数据的 watcher 实例，然后 watcher 实例调用对应组件的 render 方法，生成一颗新的 vdom 树，Vue 会将新生成的 vdom 树与上一次生成的 vdom 树进行比较（diff），来决定具体要更新哪些 dom。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6af41181deba4e40b4dcd1667cce42fd~tplv-k3u1fbpfcp-zoom-1.image)

---

React 必须显式调用 setState() 方法更新状态，状态更新之后，组件也会重新渲染。

Vue 和 React 在状态更新之后，都会生成一颗新的虚拟 dom 树，与上一颗虚拟 dom 树进行比较，找出其中的差异，再更新真实 dom。这个过程里有一个虚拟 dom diff 算法，这个算法 Vue 与 React 差异其实并不大，思路应该是差不多的（这里我没有特别深入研究，如果有不同观点请留言），大家可以看看网上的文章。

## 二、Vue 推荐使用 template 定义组件，React 推荐使用 JSX

Vue 推荐使用 template 的方式定义组件，因为这样更接近原生 html，可以在不破坏原有 html 代码的基础上引入 Vue 的能力。Vue 的组件也参考了一些 Web Component 的规范，Vue 的组件可以很容易打包成 Web Component。

React 推荐使用 JSX，JSX 是使用 JS 的语法来编写 html 代码，所以一些流程控制，数据绑定也会更加方便。也不需要再学一套模板语法。

事实上 Vue 也提供了 JSX 的支持，不过 Vue 更推荐 template 的方式。

## 三、React 推荐使用不可变的数据

这一点对于从 Vue 转换到 React 的同学，需要特别注意。

所谓不可变的数据，就是当我们要改变一个数据对象时，不要直接修改原数据对象，而是返回一个新的数据对象。比如使用 Object.assign() 方法修改数据属性:

```javascript
const data = {
  fontSize: 14,
  color: "black"
};

const newData = Object.assign({}, data, { color: "blue" });
```

之所以推荐使用不可变的数据，一个原因是使用不可变的数据，可以更容易的实现“时间旅行”功能。但是更重要的一个原因是可以更容易的实现 pure component。

当一个组件的状态发生改变时，React 会重新调用 render() 方法，比较生成的 VDOM 的差别。如果一个子组件的 proos 和 state 都没有改变，React 仍然需要进行一次对比，这个情况就有点儿浪费了。所以 React 提供了 shouldComponentUpdate() 生命周期函数，允许开发者判断什么时候应该更新组件，比如当组件的 props 和 state 都没有改变的时候，shouldComponentUpdate 就可以返回 false，那么 React 就不会再去比较 VDOM 的差异了。

React.PureComponent 类，实现了 shouldComponentUpdate 方法，会对 props 和 state 进行浅比较，如果没有变化，就返回 false 跳过组件更新。但是它只进行浅比较，所以如果直接修改了 props 或 state 的属性，shouldComponentUpdate 方法还是返回 false，就漏掉了这次更新。所以这种情况下，推荐使用不可变的数据。

更多信息请看官方文档：[为什么不可变性在 React 中非常重要](https://zh-hans.reactjs.org/tutorial/tutorial.html#why-immutability-is-important)
