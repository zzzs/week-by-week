# ts 编译与启动

# tag
`typescript` `tsc` `tsc-watch` `ts-node` `references` `nodemon`

# 介绍
由于 js 类型不确定的问题，目前大多都使用 ts 进行开发，常规场景的使用姿势也就随意用用，这边也随意的记录了一些使用姿势。

# 编译
平时编译 ts 文件时，如：
```
test
├── dist
│   ├── lib
│   └── src
├── lib
│   └── dd.ts
├── src
│   └── cc.ts
└── tsconfig.json
```

tsconfig.json
```json
// tsconfig.json
{
  "compilerOptions": {
    "outDir": "dist"
  },
  "include": ["src","lib"]
}
```
产物结果看着嘎嘎舒适，当你编译内容去除 `lib` 时，如：

```
test
├── dist
│   ├── cc.js
│   └── dd.js
...
└── tsconfig.json

```

tsconfig.json
```json
// tsconfig.json
{
  "compilerOptions": {
    "outDir": "dist"
  },
  "include": ["src"]
}
```

玛雅，产物咋煤油 `src` 了呢，我不理解，我觉得不应该啊，这不就告诉我 ts 编译行为不一致吗，被迫得出了这么个结论：
> 当编译目标有一个公共根目录时，编译产物是不会保留根目录的.

这简直对我的构建流程增加了考验。

直到后来的后来，发现了 `rootDir` （仅用来控制输出的目录结构）这个编译选项，看着含义感觉有戏，于是试了下，我天，这不就我想要的那嘎达吗：

tsconfig.json
```json
// tsconfig.json
{
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "../test"
  },
  "include": ["src"]
}
```

```
test
├── dist
│   └── src
...
└── tsconfig.json
```

再说了常用的选项 `paths`（模块名到基于 `baseUrl` 的路径映射的列表），可以简单理解为路径别名的意思，一般路径比较长会用到这玩意儿，如：
tsconfig.json
```json
// tsconfig.json
{
  "compilerOptions": {
    "outDir": "dist",
    "paths": {
      "@src/*": ["src/*"]
    },
  },
  "include": ["src"]
}

```

```
// dd.ts
import { cc } from "@src/cc"
export const dd = 1
console.log(cc)
```

可是呢，构建产物是这样滴：

```js
// dd.js
"use strict";
Object.defineProperty(exports, "__esModule", { value: true });
exports.dd = void 0;
var cc_1 = require("@src/cc");
exports.dd = 1;
console.log(cc_1.cc);
```
可以看到，加载的模块路径这样不行啊，于是 [ts-patch](https://github.com/nonara/ts-patch) 出来了，据说它是个 ts 补丁，能把上面那路径变为相对路径，那我必须得试一下：

- npm i ts-patch -D
- ts-patch install

完成上述后，再在 tsconfig.json 加上插件配置：
```json
// tsconfig.json
{
  "compilerOptions": {
    "paths": {
      "@src/*": ["src/*"]
    },
    "plugins": [
      { "transform": "typescript-transform-paths" }
    ],
  }
}
```

然后再进行编译，呀！还真是：
```js
// dd.js
"use strict";
Object.defineProperty(exports, "__esModule", { value: true });
exports.dd = void 0;
var cc_1 = require("./cc");
exports.dd = 1;
console.log(cc_1.cc);
```

# 启动
ts 的出现对启动方式影响很大，之前直接 `node index.js` 就启动了，如果有条件的，还会再加个 [nodemon](https://nodemon.io/) 来热编译，这样就可以开发内容实时生效。现在还得先编译，咋感觉还搞麻烦了捏，民间传言下面这个命令能如 js 般丝滑：
```shell
tsc-watch --onSuccess "node ./dist/server.js"
```
使用了一段时间后发现 [tsc-watch](https://github.com/gilamran/tsc-watch) 对 ts 是友好的，只是以前项目是 js 的话，就不能用这个了。

好了，前面扒拉了这么多，说下现在我们用的方式，[ts-node](https://github.com/TypeStrong/ts-node)，我们可以直接通过 `ts-node index.ts` 来启动。再加个 `nodemon` 也能实现实时编译的效果。

```shell
nodemon -r ts-node --watch app index.ts
```

我们知道，一般大厂的特点就是，历史包袱重，且又要统一研发工具，所以这个选择也是择中之举，但这个方式也是有缺陷的。
其中一个就是 ts-node 不支持 ts 中 `--build` 的模式，这个模式是 TypeScript3.0 引入的新模式，他们管这叫 **项目引用**，需要往你 tsconfig.json 中加个 `references` 属性，反正项目复杂起来后会用到的东西了，不过这个模式正确的使用它很重要，使用不正确还不如不使用，别到时候弄得埋拉巴汰的。顺便透露下，若结合 `rootDir`, `paths` 的编译选项也能很好应对这种复杂场景。


