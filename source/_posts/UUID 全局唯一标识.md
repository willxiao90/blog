---
title:  前端 UUID 生成 3 种方案
date: 2024-1-12 21:24:15
tags: [uuid]
---

## UUID

UUID 通用唯一识别码（Universally Unique Identifier）是用于计算机体系中以识别信息的一个128位标识符。

UUID按照标准方法生成时，在实际应用中具有唯一性，且不依赖中央机构的注册和分配。UUID重复的概率接近零，可以忽略不计。

因此，UUID 的应用非常普遍，被广泛应用于需要对数据记录、资源和实体进行唯一标识的众多应用中：数据库、资源标识符、会话和事务标识符、对象存储等。

前端可以使用 uuidjs 库实现：[https://github.com/uuidjs/uuid](https://github.com/uuidjs/uuid)

![Alt text](/blog/images/image-3.png)

## nanoid

nanoid 是 UUID 的有力竞争者，它同样可以生成唯一的标识字符串。

与 UUID 相比，它使用更大的字母表，这样一来它生成的字符串长度更短，只有21个字符。

并且它的包体积只有UUID的1/4。nanoid 大有取代 UUID 的趋势。

![Alt text](/blog/images/image-4.png)

另外，nanoid 可以自定义字母表和ID长度，这给用户提供了更多灵活性。

``` javascript
import { customAlphabet } from 'nanoid'
const nanoid = customAlphabet('1234567890abcdef', 10)
model.id = nanoid() //=> "4f90d13a42"
```

更多信息见 NPM：[https://www.npmjs.com/package/nanoid](https://www.npmjs.com/package/nanoid)

## Crypto.randomUUID

事实上，如果你的项目仅面向现代浏览器：原生 crypto.randomUUID() 是最佳选择。作为浏览器原生API，它无需引入任何库，兼容性好，且符合标准UUID格式，是零依赖方案的首选。

![Alt text](/blog/images/image-5.png)

详情见 MDN：[https://developer.mozilla.org/zh-CN/docs/Web/API/Crypto/randomUUID](https://developer.mozilla.org/zh-CN/docs/Web/API/Crypto/randomUUID)