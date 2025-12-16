---
title: 记一次 electron-vue 项目开发经验
date: 2020-10-29 18:14:15
tags: [electron, electron-vue]
---

最近公司让我开发一个桌面报警器，以解决浏览器页面关闭无法播放报警声音的问题。

接到这个项目，自然的选择了 [electron-vue](https://simulatedgreg.gitbooks.io/electron-vue/content/cn/) 进行开发（我们公司使用的 vue）

现在有时间了，对项目中遇到的问题进行一个总结。

## 一、项目搭建 & 打包

项目搭建比较简单，直接使用 electron-vue 的官方模板就可以生成项目，需要安装 vue-cli 命令行工具。

```
npm install -g vue-cli // 需要安装 vue-cli 脚手架
vue init simulatedgreg/electron-vue project-name // 使用 electron-vue 官方模板生成项目
npm install // 安装依赖
npm run dev // 启动项目
```

项目打包也比较简单，可能也是因为我的项目本身不复杂吧。普通打包执行 npm run build 即可，如果要打包成免安装文件，执行 npm run build:dir，非常方便！

```
npm run build // 打包成可执行文件
npm run build:dir // 打包成免安装文件
```

## 二、状态管理

因为 electron 每个网页都在自己的渲染进程（renderer process）中运行，所以如果要在多个渲染进程间共享状态，就不能直接使用 vuex 了。

[vuex-electron](https://github.com/vue-electron/vuex-electron) 这个开源库为我们提供了，在多个进程间共享状态的方案（包括主进程）。

如果需要在多个进程间共享状态，需要使用 createSharedMutations 中间件。

``` javascript
// store.js 文件
import Vue from "vue"
import Vuex from "vuex"
 
import { createPersistedState, createSharedMutations } from "vuex-electron"
 
Vue.use(Vuex)
 
export default new Vuex.Store({
  // ...
  plugins: [
    createPersistedState(),
    createSharedMutations() // 用于多个进程共享状态，包括主进程
  ],
  // ...
})
```

并在主进程中引入 store 文件。这里有点坑，最开始的时候我不知道要在 main.js 中引入 store 文件，结果状态一直无法更新，又没有任何报错，调试了一下午😓

``` javascript
// main.js 文件
import './path/to/your/store' // 需要在主进程引入 store ，否则状态无法更新
```

另外，使用 createSharedMutations 中间件，必须使用 dispatch 或 mapActions 更新状态，不能使用 commit 。

阅读 vuex-electron 的源代码，发现渲染进程对 dispatch 进行了重写，dispatch 只是通知主进程，而不实际更新 store，主进程收到 action 之后，立即更新自己的 store，主进程 store 更新成功之后，会通知所有的渲染进程，这个时候渲染进程才调用 originalCommit 更新自己的 store。

``` javascript
rendererProcessLogic() {
    // Connect renderer to main process
    this.connect()

    // Save original Vuex methods
    this.store.originalCommit = this.store.commit
    this.store.originalDispatch = this.store.dispatch

    // Don't use commit in renderer outside of actions
    this.store.commit = () => {
        throw new Error(`[Vuex Electron] Please, don't use direct commit's, use dispatch instead of this.`)
    }

    // Forward dispatch to main process
    this.store.dispatch = (type, payload) => {
        // 只是通知主进程，没有更新 store
        this.notifyMain({ type, payload })
    }

    // Subscribe on changes from main process and apply them
    this.onNotifyRenderers((event, { type, payload }) => {
        // 渲染进程真正更新自己的 store
        this.store.originalCommit(type, payload)
    })
}

// ... 省略其他代码

mainProcessLogic() {
    const connections = {}

    // Save new connection
    this.onConnect((event) => {
        const win = event.sender
        const winId = win.id

        connections[winId] = win

        // Remove connection when window is closed
        win.on("destroyed", () => {
        delete connections[winId]
        })
    })

    // Subscribe on changes from renderer processes
    this.onNotifyMain((event, { type, payload }) => {
        // 主进程更新了自己的 store
        this.store.dispatch(type, payload)
    })

    // Subscribe on changes from Vuex store
    this.store.subscribe((mutation) => {
        const { type, payload } = mutation

        // 主进程更新成功之后，通知所有渲染进程
        this.notifyRenderers(connections, { type, payload })
    })
}
```

注意，渲染进程真正更新 store 用的 originalCommit 方法，而不是 originalDispatch 方法，其实 originalDispatch 只是个代理，每一个 mutations 都需要写一个同名的 actions 方法，接收相同的参数，如下面的官方样例：

``` javascript
import Vue from "vue"
import Vuex from "vuex"

import { createPersistedState, createSharedMutations } from "vuex-electron"

Vue.use(Vuex)

export default new Vuex.Store({
  state: {
    count: 0
  },

  actions: {
    increment(store) {
      // 按照推理，这里的 commit 其实不起作用，不是必须
      // 关键是名称相同
      store.commit("increment")
    },
    decrement(store) {
      store.commit("decrement")
    }
  },

  mutations: {
    increment(state) {
      state.count++
    },
    decrement(state) {
      state.count--
    }
  },

  plugins: [createPersistedState(), createSharedMutations()],
  strict: process.env.NODE_ENV !== "production"
})
```

事实上，如果应用很简单，比如我的项目只有一个窗口，就不存在共享状态的问题，所以完全可以不用 createSharedMutations 中间件，也不用在 main.js 中引入 store 文件，store 所有用法就跟 vuex 一样了。

## 三、日志

日志我采用的是 [electron-log](https://github.com/megahertz/electron-log)，也可以用 [log4js](https://github.com/log4js-node/log4js-node)

在主进程中使用 electron-log 很简单，直接引入，调用 info 等方法即可。
electron-log 提供了 error, warn, info, verbose, debug, silly 六种级别的日志，默认都是开启。

``` javascript
import log from 'electron-log';
 
log.info('client 启动成功');
log.error('主进程出错');
```

在渲染进程使用 electron-log，可以覆盖 console.log 等方法，这样就不用到处引入 electron-log 了，需要写日志的地方直接使用 console.log 等方法即可。

``` javascript
import log from 'electron-log';
 
 // 覆盖 console 的 log、error、debug 三个方法
console.log = log.log;
Object.assign(console, {
  error: log.error,
  debug: log.debug,
});

// 之后，就可以直接使用 console 收集日志
console.error('渲染进程出错')
```

electron-log 默认会打印到 console 控制台，并写入到本地文件，本地文件路径如下：
- on Linux: ~/.config/{app name}/logs/{process type}.log
- on macOS: ~/Library/Logs/{app name}/{process type}.log
- on Windows: %USERPROFILE%\AppData\Roaming\{app name}\logs\{process type}.log

---

如果使用 log4js 的话，配置相对复杂一点，需要注意的是文件不能直接写到当前目录，而是要使用 app.getPath('logs') 获取应用程序日志文件夹路径，否则打包之后无法生成日志文件。例如：

``` javascript
import log4js from 'log4js'
 
// 注意：这里必须使用 app.getPath('logs') 获取日志文件夹路径
log4js.configure({
  appenders: { cheese: { type: 'file', filename: app.getPath('logs') + '/cheese.log' } },
  categories: { default: { appenders: ['cheese'], level: 'error' } }
})
 
const logger = log4js.getLogger('cheese')
logger.trace('Entering cheese testing')
logger.debug('Got cheese.')
logger.info('Cheese is Comté.')
logger.warn('Cheese is quite smelly.')
logger.error('Cheese is too ripe!')
logger.fatal('Cheese was breeding ground for listeria.')
```

## 四、其他问题

1.修改系统托盘图标，下面代码参考了：[https://juejin.im/post/6844903872905871373](https://juejin.im/post/6844903872905871373)

``` javascript
let tray;
function createTray() {
  const iconUrl = path.join(__static, '/app-icon.png');
  const appIcon = nativeImage.createFromPath(iconUrl);
  tray = new Tray(appIcon);
 
  const contextMenu = Menu.buildFromTemplate([
    {
      label: '显示主界面',
      click: () => {
        if (mainWindow) {
          mainWindow.show();
        }
      },
    },
    { label: '退出程序', role: 'quit' },
  ]);
 
  const appName = app.getName();
  tray.setToolTip(appName);
  tray.setContextMenu(contextMenu);
 
  let timer;
  let count = 0;
  ipcMain.on('newMessage', () => {
    // 图标闪烁
    timer = setInterval(() => {
      count += 1;
      if (count % 2 === 0) {
        tray.setImage(appIcon);
      } else {
        // 创建一个空的 nativeImage 实例
        tray.setImage(nativeImage.createEmpty());
      }
    }, 500);
      tray.setToolTip('您有一条新消息');
  });
 
  tray.on('click', () => {
    if (mainWindow) {
      mainWindow.show();
      if (timer) {
        clearInterval(timer);
        tray.setImage(appIcon);
        tray.setToolTip(appName);
        timer = undefined;
        count = 0;
      }
    }
  });
}
```

2.播放声音
``` javascript
audio = new Audio('static/alarm.wav');
audio.play(); // 开始播放
audio.pause(); // 暂停
```

3.显示通知消息

``` javascript
const notify = new Notification('标题', {
   tag: '唯一标识', // 相同 tag 只会显示一个通知
   body: '描述信息',
   icon: '图标地址',
   requireInteraction: true, // 要求用户有交互才关闭（实测无效）
   data, // 其他数据
});
 
// 通知消息被点击事件
notify.onclick = () => {
   console.log(notify.data)
};
```

4.隐藏顶部菜单栏

``` javascript
import { Menu } from 'electron'
 
// 隐藏顶部菜单
 Menu.setApplicationMenu(null);
```

## 五、参考资料

- electron 官方文档：[https://www.electronjs.org/docs](https://www.electronjs.org/docs)
- electron-vue 文档：[https://simulatedgreg.gitbooks.io/electron-vue/content/cn/](https://simulatedgreg.gitbooks.io/electron-vue/content/cn/)
- electron系统托盘及消息闪动提示：[https://juejin.im/post/6844903872905871373](https://juejin.im/post/6844903872905871373)