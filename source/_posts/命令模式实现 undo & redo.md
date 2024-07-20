---
title: 命令模式实现 undo & redo
date: 2024-7-20 20:14:15
tags: [设计模式, 命令模式]
---

前端 undo & redo 功能是非常常见的，通常会使用命令模式来实现。

下面以一个低代码编辑器的例子，来介绍 JavaScript 是如何使用命令模式来实现 undo & redo 功能的。

首先，我们来看一下命令模式的结构示意图。

![](https://img-blog.csdnimg.cn/cc2f4b0283e74a8fa5cdaeb85c2837d1.png)

在命令模式中，关键是定义了一个 Command 接口，它有 execute 和 undo 两个方法，具体的命令类都需要实现这两个方法。调用者（Invoker）在调用命令的时候，只需要执行命令对象的 execute 和 undo 方法即可，而不用关心这两个方法具体做了什么。实际上这两方法的具体实现，通常都是在接收者（Receiver）中，命令类中通常有一个接收者实例，命令类只需要调用接收者实例方法即可。

OK，我们来看一下，我们的低代码编辑器的状态库（简化版的）。它是使用 zustand 定义的，它有一个组件列表 componentList，以及相关的3个方法。

```javascript
import { createStore } from "zustand/vanilla";

const store = createStore((set) => ({
  componentList: [], // 组件列表
  // 添加组件
  addComponent: (comp) =>
    set((state) => ({ componentList: [...state.componentList, comp] })),
  // 删除组件
  removeComponent: (comp) =>
    set((state) => ({
      componentList: state.componentList.filter((v) => v.id !== comp.id),
    })),
  // 更新组件属性
  updateComponentProps: (comp, newProps) =>
    set((state) => {
      const index = state.componentList.findIndex((v) => v.id === comp.id);
      if (index > -1) {
        const list = [...state.componentList];
        return {
          componentList: [
            ...list.slice(0, index),
            { ...comp, props: newProps },
            ...list.slice(index + 1),
          ],
        };
      }
    }),
}));
// const { getState, setState, subscribe, getInitialState } = store;

export default store;
```

接下来，我们看一下相关命令类的实现：

```javascript
// 命令基类
class Command {
  constructor() {}

  execute() {
    throw new Error("未重写 execute 方法！");
  }

  undo() {
    throw new Error("未重写 undo 方法！");
  }
}

export class AddComponentCommand extends Command {
  editorStore; // 状态库（它充当 Receiver）
  comp;

  constructor(editorStore, comp) {
    super();
    this.editorStore = editorStore;
    this.comp = comp;
  }

  execute(comp) {
    this.editorStore.getState().addComponent(this.comp);
  }

  undo() {
    this.editorStore.getState().removeComponent(this.comp);
  }
}

export class RemoveComponentCommand extends Command {
  editorStore;
  comp;

  constructor(editorStore, comp) {
    super();
    this.editorStore = editorStore;
    this.comp = comp;
  }

  execute() {
    this.editorStore.getState().removeComponent(this.comp);
  }

  undo() {
    this.editorStore.getState().addComponent(this.comp);
  }
}

export class UpdateComponentPropsCommand extends Command {
  editorStore;
  comp;
  newProps;
  prevProps; // 保存之前的属性

  constructor(editorStore, comp, newProps) {
    super();
    this.editorStore = editorStore;
    this.comp = comp;
    this.newProps = newProps;
  }

  execute() {
    const { updateComponentProps, componentList } = this.editorStore.getState();
    this.prevProps = componentList.find((v) => v.id === this.comp.id)?.props;
    updateComponentProps(this.comp, this.newProps);
  }

  undo() {
    const { updateComponentProps } = this.editorStore.getState();
    updateComponentProps(this.comp, this.prevProps);
  }
}
```

我们实现了 AddComponentCommand、RemoveComponentCommand 和 UpdateComponentPropsCommand 3个命令类，在我们的命令类中都有一个 editorStore 属性，它在这里充当了 Receiver 接收者，因为编辑器相关操作我们都定义在状态库中。

其中 AddComponentCommand 和 RemoveComponentCommand 相对比较简单，有直接的操作可以实现撤销。UpdateComponentPropsCommand 就稍微复杂一点，我们更新了属性之后，没有一个直接的操作可以撤销修改，这种情况我们通常需要增加一个属性，记录修改之前的状态，用于实现撤销功能，在 UpdateComponentPropsCommand 中就是 prevProps。

到这里，我们的命令类都已经实现了，要实现 undo 和 redo 功能，通常我们还需要实现一个命令管理类，它需要实现 execute、undo 和 redo 三个方法。它的具体实现多种方法，我们这里使用两个栈（Stack）来实现，具体代码如下：

```javascript
class CommandManager {
  undoStack = []; // 撤销栈
  redoStack = []; // 重做栈

  execute(command) {
    command.execute();
    this.undoStack.push(command);
    this.redoStack = [];
  }

  undo() {
    const command = this.undoStack.pop();
    if (command) {
      command.undo();
      this.redoStack.push(command);
    }
  }

  redo() {
    const command = this.redoStack.pop();
    if (command) {
      command.execute();
      this.undoStack.push(command);
    }
  }
}

export default new CommandManager();
```

有了这些，接下来我们可以进入测试环节了，下面是我们的测试代码：

```javascript
import store from "./store/editorStore";
import cmdManager from "./commands/cmdManager";

// 实时打印组件列表
store.subscribe((state) =>
  console.log(JSON.stringify(state.componentList))
);

const comp1 = {
  id: 101,
  componentName: "Comp1",
  props: {},
  children: null,
};
const comp2 = {
  id: 102,
  componentName: "Comp2",
  props: {},
  children: null,
};

cmdManager.execute(new AddComponentCommand(store, comp1));
cmdManager.execute(new AddComponentCommand(store, comp2));
cmdManager.undo();
cmdManager.redo();

cmdManager.execute(new RemoveComponentCommand(store, comp1));
cmdManager.undo();

cmdManager.execute(
  new UpdateComponentPropsCommand(store, comp1, { visible: true })
);
cmdManager.undo();
```

测试结果如下，说明我们的代码正常工作了。

```javascript
// [{"id":101,"componentName":"Comp1","props":{},"children":null}]
// [{"id":101,"componentName":"Comp1","props":{},"children":null},{"id":102,"componentName":"Comp2","props":{},"children":null}]
// [{"id":101,"componentName":"Comp1","props":{},"children":null}]
// [{"id":101,"componentName":"Comp1","props":{},"children":null},{"id":102,"componentName":"Comp2","props":{},"children":null}]
// [{"id":102,"componentName":"Comp2","props":{},"children":null}]
// [{"id":102,"componentName":"Comp2","props":{},"children":null},{"id":101,"componentName":"Comp1","props":{},"children":null}]
// [{"id":102,"componentName":"Comp2","props":{},"children":null},{"id":101,"componentName":"Comp1","props":{"visible":true},"children":null}]
// [{"id":102,"componentName":"Comp2","props":{},"children":null},{"id":101,"componentName":"Comp1","props":{},"children":null}]
```

&#x20;至此，我们已经完成了完整的第一个版本了。但是代码还有优化的空间，我们继续改进一下。

第一点，执行命令的地方，要手动 new 命令类，传入 store 状态库，有较多的模板代码。

```javascript
cmdManager.execute(new AddComponentCommand(store, comp1));
cmdManager.execute(new AddComponentCommand(store, comp2));
cmdManager.undo();
cmdManager.redo();
```

我们可以参考 js 原生方法 [document.execCommand](https://developer.mozilla.org/zh-CN/docs/Web/API/Document/execCommand) 实现一个 executeCommand () 方法，这样执行命令就变成了 executeCommand(commandName, ...args) 这样，更为方便。

```javascript
import cmdManager from "./cmdManager";
import {
  AddComponentCommand,
  RemoveComponentCommand,
  UpdateComponentPropsCommand,
} from "./index";
import store from "../store/editorStore";

const commondActions = {
  addComponent(...args) {
    const cmd = new AddComponentCommand(store, ...args);
    cmdManager.execute(cmd);
  },

  removeComponent(...args) {
    const cmd = new RemoveComponentCommand(store, ...args);
    cmdManager.execute(cmd);
  },

  updateComponentProps(...args) {
    const cmd = new UpdateComponentPropsCommand(store, ...args);
    cmdManager.execute(cmd);
  },

  undo() {
    cmdManager.undo();
  },

  redo() {
    cmdManager.redo();
  },
};

const executeCommand = (cmdName, ...args) => {
  commondActions[cmdName](...args);
};

export default executeCommand;
```

```javascript
store.subscribe((state) =>
  console.log(JSON.stringify(state.componentList))
);

const comp1 = {
  id: 101,
  componentName: "Comp1",
  props: {},
  children: null,
};

const comp2 = {
  id: 102,
  componentName: "Comp2",
  props: {},
  children: null,
};

executeCommand("addComponent", comp1);
executeCommand("addComponent", comp2);
executeCommand("undo");
executeCommand("redo");

executeCommand("removeComponent", comp1);
executeCommand("undo");

executeCommand("updateComponentProps", comp1, { visible: true });
executeCommand("undo");
```

第二点，CommandManager 其实使用一个栈（Stack）加上指针也可以实现，我们参考了网上的代码（[JavaScript command pattern for undo and redo](https://developer.s24.com/blog/js-command-pattern-for-undo-and-redo.html)），优化之后代码如下：

```javascript
class CommandManager {
  _commandsList = [];
  _currentCommand = -1;

  execute(command) {
    command.execute();
    this._currentCommand++;
    this._commandsList[this._currentCommand] = command;
    if (this._commandsList[this._currentCommand + 1]) {
      this._commandsList.splice(this._currentCommand + 1);
    }
  }

  undo() {
    const command = this._commandsList[this._currentCommand];
    if (command) {
      command.undo();
      this._currentCommand--;
    }
  }

  redo() {
    const command = this._commandsList[this._currentCommand + 1];
    if (command) {
      command.execute();
      this._currentCommand++;
    }
  }
}

export default new CommandManager();
```

OK，这就是我们的第二个版本了。

参考资料：

《Head First 设计模式 - 命令模式》

[javascript - 基于Web的svg编辑器（1）——撤销重做功能 - 个人文章 - SegmentFault 思否](https://segmentfault.com/a/1190000018940715)

[JavaScript command pattern for undo and redo (s24.com)](https://developer.s24.com/blog/js-command-pattern-for-undo-and-redo.html)
