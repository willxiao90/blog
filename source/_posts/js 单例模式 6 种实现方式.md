---
title: js 单例模式 6 种实现方式
date: 2024-12-09 21:23:13
tags: [设计模式, 单例模式]
---

JavaScript 中的单例模式确保一个类只有一个实例，并提供全局访问点。以下是几种常见的实现方式：

## 1.对象字面量（最简单的方式）

```javascript
const Singleton = {
  property: 'value',
  method() {
    // 使用 this（在方法被解构调用时会丢失上下文）
    // console.log(this.property);
    
    // 使用 Singleton（更安全）
    console.log(Singleton.property);
  }
};

// 使用
Singleton.method();
```

`javascript` 可以使用对象字面量快速创建一个对象，创建的对象本身就是单例。这种方式最为简单，但是需要注意 `this` 的引用可能出问题。

为了解决 `this` 的问题，可以直接引用单例对象本身。

## 2.闭包实现

```javascript
const Singleton = (function() {
  let instance;
  
  function createInstance() {
    const object = new Object('I am the instance');
    return object;
  }
  
  return {
    getInstance: function() {
      if (!instance) {
        instance = createInstance();
      }
      return instance;
    }
  };
})();

// 使用
const instance1 = Singleton.getInstance();
```

这里利用了闭包的特性实现了模块的封装和单例对象的引用，返回一个 `getInstance` 方法用于获取实例对象。

## 3.ES6 Class 实现

```javascript
class Singleton {
  constructor() {
    if (Singleton.instance) {
      return Singleton.instance;
    }
    
    this.data = 'Singleton Data';
    Singleton.instance = this;
  }
  
  getData() {
    return this.data;
  }
  
  setData(data) {
    this.data = data;
  }
}

// 使用
const s1 = new Singleton();
const s2 = new Singleton();
console.log(s1 === s2); // true
s1.setData('New Data');
console.log(s2.getData()); // 'New Data'
```

这里的 `instance` 是一个静态变量，这种方式在 `constructor` 的最后将 `this` 赋值给 `instance`。

## 4.改进的 class 实现（typescript 版本）

```typescript
class Singleton {
  private static instance?: Singleton;

  private constructor() {}

  static getInstance() {
    if (!Singleton.instance) {
      Singleton.instance = new Singleton();
    }
    return Singleton.instance;
  }
}

// 使用
const s1 = Singleton.getInstance();
const s2 = Singleton.getInstance();
console.log(s1 === s2); // true
```

`typescript` 版本抽取 `getInstance` 方法，让代码更可读。使用 `private` 关键字实现私有属性和方法，保证代码不会被随意篡改。

## 5.ES6 模块模式的单例

```javascript
// 在模块文件中
let instance = null;

class Database {
  constructor(config) {
    if (instance) {
      return instance;
    }
    
    this.connection = this.connect(config);
    instance = this;
  }
  
  connect(config) {
    return { connected: true, config };
  }
}

// 导出一个获取实例的函数
export const getInstance= (() => {
  let instance = null;
  return (config) => {
    if (!instance) {
      instance = new Database(config);
    }
    return instance;
  };
})();
```

这种方式利用 ES6 模块变量的引用共享特性，保证 `instance` 唯一。导出一个 `getInstance` 方法。

## 6.ES6 模块本身就是单例

```javascript
class Database {
  constructor(config) {
    this.connection = this.connect(config);
  }
  
  connect(config) {
    return { connected: true, config };
  }
}

// 直接导出一个实例
export default new Database({ host: 'localhost' });
```

实测证明，导出的实例在多个模块间是共享的。

## 总结

JavaScript 实现单例模式的方式很多，这里介绍常用的6种，主要分4大类：对象字面量、闭包实现、 ES6 class 实现、ES6 模块模式实现。

选择哪种实现方式取决于具体需求，简单场景可以使用对象字面量，复杂场景建议使用 ES6 Class 或 ES6 模块模式实现。
