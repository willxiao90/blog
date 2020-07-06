---
title: Vue 与 React 的区别
date: 2020-06-29 10:19:26
tags: vue, react
---

> Vue 与 React 有什么区别？

这是前端开发同学面试时经常遇到的问题。

我也不例外，我最开始接触的是 React，对 Vue 的理解一直比较片面。感觉 Vue 要学很多 html 指令，很不习惯，也没觉得 Vue 比 React 有什么优势。

直到现在，使用了 Vue 一年之后，对 Vue 有了更多感受，也消除了一些刻板印象。

首先，这两个框架都是非常优秀的单页应用（SPA）开发框架，它们其实非常相似，都有以下特性：

1. 响应式（Reactive）。两个框架都是一种类似 VM 的架构，将状态从视图层分离出来，开发者只需要关注业务逻辑，不需要直接操作 DOM 。当应用发生改变时，我们只需要更新状态即可，框架会自动帮我们重新渲染页面。
2. 组件化（Composable）。一个页面，可以拆分成一棵嵌套的组件树，我们只需要开发一个个组件即可，同一个组件可以在多个地方使用，这样就提升了代码的复用性和可维护性。
3. 使用了 Virtual DOM。框架在操作真实 DOM 之前，会先在内存中生成虚拟 DOM，最后再批量操作真实 DOM，以提高性能。

我个人理解，它们最大的差别是响应式原理不同，组件的定义方式和 state 状态管理也有一些差别。

## 一、响应式原理不同

Vue 的响应式，是使用观察者模式实现的。Vue 会遍历 data 数据对象，使用 Object.defineProperty() 将每个属性都转换为 getter/setter。

每个 Vue 组件实例都对应一个  watcher  实例，在组件渲染（render）过程中时，watcher 实例会记录哪些子组件用到了（getter）哪些数据属性。当数据属性发生改变时，会触发 setter 方法，watcher 实例会通知所有用到了这个数据属性的子组件，调用该子组件的 updateComponent 方法更新组件。

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

React 的响应式，是使用 diff 算法实现的。React 在 state 或 props 改变时，会调用 render() 方法，生成一个虚拟 DOM 树，React 会将这棵树与上一次生成的树进行比较，找出其中的差异，并更新差异的部分。

比较两棵树，找出其中的差异，并生产一个做小操作数。这个算法的复杂度比较高，是 O(n 3 )，React 为了提高性能， 提出了一套复杂度为 O(n) 的优化算法， 这个算法有两个重要的假设：

1. 两个不同类型的元素会产生出不同的树。当根节点元素类型发生改变时，React 会销毁旧节点创建新节点。比如当一个元素从 `<Button>` 变成 `<div>`，或者 `<ComponentA>` 变成 `<ComponentB>`。当元素类型相同时，React 会保留 DOM 节点，仅比较并更新有改变的属性。如果元素有子节点，React 会递归比较  子节点。
2. 当元素类型相同时，开发者可以使用 key 属性来标识元素的唯一性。比如一个列表， 有多个相同类型的子节点，当子节点顺序发生改变是，如果没有一个唯一标识，就有可能产生比较多的操作数。React 引入 key 属性作为唯一标识，就是为了解决这个问题。

关于 React 的  diff 算法，官方文档写的很清楚 ，我就不多说了。 详情请看：[https://zh-hans.reactjs.org/docs/reconciliation.html](https://zh-hans.reactjs.org/docs/reconciliation.html)

## 二、Vue 可以直接修改状态，React 必须通过 setState() 更新状态

这个区别其实也是因为它们的响应式原理不同。

Vue 数据属性变更时，会自动通知所有用到了这个属性的子组件，调用它们的 updateComponent 方法更新组件，所以 Vue 直接修改数据属性组件也会正确更新。

React 调用 setState() 方法，组件会自动执行 render() 方法重新渲染。但是直接修改 state，不会调用 render() 方法，组件不会更新，除非调用 forceUpdate() 强制更新组件。

## 三、Vue 支持双向数据绑定，React 只允许单向数据流

Vue 提供了 v-model 指令，可以进行双向数据绑定。不过 v-model 其实只是一个语法糖，它的本质是  绑定 value 属性，并监听 input 事件，当 input 有新的  内容时更新 value 属性。

当 v-model  绑定一个对象时，Vue 允许子组件直接修改对象属性，得益于 Vue 的响应式原理，父组件中的用到对象属性的地方也会自动更新。

例如，有一个很长  的表单，字段  实在太多了，我们就需要把这个组件拆分为更小的子组件，这个时候，我们可以利用状态提升的技巧， 把状态都  放在父组件里面，然后把父组件的状态对象使用 v-model 指令绑定给子组件，在子组件中就可以直接修改父组件的对象  属性。

```html
<template>
  <form>
    <base-info v-model="form" />
    <company-info v-model="form.companyInfo" />
    <education-info v-model="form.educationInfo" />
  </form>
</template>

<script>
  export default {
    data() {
      return {
        form: {
          name: '',
          gender: 0,
          age: 18,
          email: '',
          phone: '',
          address1: '',
          address2: '',
          companyInfo: {
            name: '',
            tel: '',
            address: ''
          },
          educationInfo: {
            college: '',
            profession: '',
            graduateAt: ''
          }
          ...
        }
      }
    }
  }
</script>
```

---

React 只允许单向数据流，React 认为这样虽然麻烦一点儿，但是显式声明的方法更有助于人们理解程序的运作方式，不容易引入 bug。

上面的例子，如果使用 React 的写法，需要将修改父组件状态的方法传递给子组件，在子组件中调用。

```javascript
class Person extends Component {
  constructor(props) {
    super(props);

    this.state = {
      name: '',
      gender: 1,
      phone: '',
      email: '',
      address1: '',
      address2: '',
      companyInfo: {
        name: '',
        tel: '',
        address: ''
      },
      educationInfo: {
        college: '',
        profession: '',
        graduateAt: ''
      }
      ...
    };
  }

  updateBaseInfo(prop, value) {
    this.setState((state) => {
      ...state,
      {prop: value}
    });
  }

  updateCompanyInfo(prop, value) {
    this.setState((state) => {
      ...state,
      companyInfo: {
        ...state.companyInfo,
        {prop: value}
      }
    });
  }

  updateEducationInfo(prop, value) {
    this.setState((state) => {
      ...state,
      educationInfo: {
        ...state.educationInfo,
        {prop: value}
      }
    });
  }

  render() {
    const {form} = this.state;
    return (
      <form>
        <baseInfo data={form} update={updateBaseInfo} />
        <companyInfo data={form.companyInfo} update={updateCompanyInfo} />
        <educationInfo data={form.educationInfo} update={updateEducationInfo} />
      </form>
    )
  }
}
```

## 四、React 推荐使用不可变的数据

所谓不可变的数据，就是  当我们要改变一个  数据对象时，不要直接修改原数据对象，而是返回一个新的数据对象。比如使用 Object.assign() 方法修改数据属性:

```javascript
const data = {
  fontSize: 14,
  color: "black"
};

const newData = Object.assign({}, data, { color: "blue" });
```

之所以推荐使用不可变的数据，一个原因  是使用不可变的数据，可以更容易的实现“时间旅行 ”功能。但是更重要的一个  原因是可以更容易的实现 pure component。

 刚才  讲过 React 的响应式原理，当一个组件的状态发生改变时，React 会重新调用 render() 方法，比较生成的 VDOM 的差别。如果一个子组件的 proos 和 state 都没有改变，React  仍然需要  进行一次  对比，这个情况就  有点儿浪费了。所以 React 提供了 shouldComponentUpdate() 生命周期函数， 允许开发者判断什么时候应该更新组件，比如当组件的 props 和 state 都没有改变的时候，shouldComponentUpdate 就可以返回 false，那么 React 就不会再去比较 VDOM 的差异了。

React.PureComponent 类，实现了 shouldComponentUpdate 方法，会对 props 和 state 进行浅比较，如果没有变化，就返回 false 跳过组件更新。但是它只进行浅比较，所以如果直接修改了 props 或 state 的属性，shouldComponentUpdate 方法还是返回 false，就  漏掉了这次更新。所以这种情况下，推荐使用不可变的数据。

##  五、Vue 推荐使用 template 方式定义组件，React 推荐使用 JSX

Vue 推荐使用 template 的方式定义组件，因为这样更接近原生 html，可以在不破坏原有 html 代码的基础上引入 Vue 的能力。Vue 的组件也  参考了一些 Web Component 的规范，Vue 的组件可以很容易打包成 Web Component。

React 推荐使用 JSX，JSX 是使用 JS 的语法来编写 html 代码，所以一些流程控制，数据绑定也会更加方便些。JSX  不依赖模版，所以出现了 React Native 这样的  跨平台  框架。

事实上 Vue 也提供了 JSX 的支持，不过 Vue 更推荐 template 的方式。

## 六、React 对函数组件有更好的支持

React 发布之初，就支持函数组件，一个无状态的组件，可以使用一个函数表示， 函数组件接收一个 props 属性，并返回一个 React 元素。例如：

```javascript
function Welcome(props) {
  return <h1>Hello, {props.name}</h1>;
}
```

React 16.8 新增了 Hooks 特性，增强了函数组件的能力，使函数组件可以使用 state 以及其他 React 特性。

这样一来，我们就可以不使用 class 组件，只使用函数组件就可以开发复杂的 React 组件。 React 也更推荐使用函数组件， 因为函数组件更加简单，更加易于测试。相比较 class 组件，代码  逻辑分散在各个生命周期函数之中， 代码变得不好理解，也很难测试。

具体可以查看官方 [React Hooks 文档](https://zh-hans.reactjs.org/docs/hooks-intro.html)

---

Vue 其实也是支持函数组件的，但是我在实际项目中没有  用过，Vue 好像也指出了给 template 标记为 functional，在多人协作的时候可能会造成一些混乱。具体请看官网文档： [函数式组件](https://cn.vuejs.org/v2/guide/render-function.html#%E5%87%BD%E6%95%B0%E5%BC%8F%E7%BB%84%E4%BB%B6)

Vue 3.0 新增了 [Composition API](https://vue-composition-api-rfc.netlify.app/zh/)，也提供了类似 React Hooks 函数式编程的能力。 使得定义组件更加灵活、更加简单。
