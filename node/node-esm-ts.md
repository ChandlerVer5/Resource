# node-esm-ts

## 理解模块类型
首先，需要明确 CJS 和 ESM 的基本概念：
- CommonJS (CJS): 这是 Node.js 默认的模块系统，使用 require() 函数来引入模块，并通过 module.exports 导出功能。CJS 模块是同步加载的。
- ES Module (ESM): 这是 JavaScript 的新标准模块系统，使用 import 和 export 语法。ESM 模块是异步加载的，并且在编译阶段进行解析。

Node.js 从 12.x 开始引入实验性 ESM 支持（运行带有实验标志 (`--experimental-modules`) ），而从 13.2.0 开始则提供了正式的支持，不再需要实验标志。因此，如果你希望在 Node.js 中使用 ESM，建议使用至少 Node.js 13.2.0 或更高版本，最好 Node 16 以上，类似 ts 配置中指定的 `module: "Node16"`，以确保获得最佳的兼容性和功能。现在只需几个步骤即可使用 ECMAScript 模块。

如果我们想用 TypeScript 进行开发 Node.js 应用，并且希望最终使用 ES6 模块语法。那么我们就会遇到这些问题：
（推荐 ES 模块文件后缀使用 `.mjs`，commonjs 模式推荐 `.cjs`。更多：[Node File Extensions](https://nodejs.org/api/packages.html#packagejson-and-file-extensions)）
1. `package.json` 中必须要指定 `"type": "module"` 来启用 ES6 模块语法。
2. TSC 编译时并不会自动的帮你添加 `.js` 后缀，但 ESM 要求必须有扩展名。
    ```ts
    // 缺少扩展名，编译成的 ESM 产物无法使用
    import Foo from "./foo";
    // ESM不会默认引入文件夹下的 index.js
    import * from './directory'
    
    // 正确的写法
    import Foo from "./foo.js";
    import * from './directory/index.js'
    ```
4. 启用 ES6 模块语法后，就必须指定文件的扩展名，否则会报错，比如 `.mjs`。

## ESM 与 CJS

ESM 会有一些与 CJS 不同：

- 只有 ESM 才能使用 top-level await（TLA）

    TLA 属于 ES2022 的特性。可以在 [ECMAScript proposal: Top-level await](https://github.com/tc39/proposal-top-level-await) 查看关于 TLA 的历史。

- ESM 中无法使用 `__dirname/__filename` 这类 CJS 中的内置全局变量。[alternative-for-dirname-in-node-js-when-using-es6-modules](https://stackoverflow.com/questions/46745014/alternative-for-dirname-in-node-js-when-using-es6-modules)

  解决这个问题的一个简单方法是使用 shim 填充这些值，比如 [cross-dirname](https://www.npmjs.com/package/cross-dirname)，参考 [import.meta](https://nodejs.org/api/esm.html#importmetadirname)：
     ```ts
          // Node 20.11.0+, Deno 1.40.0+
        const __dirname = import.meta.dirname;
        const __filename = import.meta.filename;
        
        // Previously
        const __dirname = new URL(".", import.meta.url).pathname;
        
        import { fileURLToPath } from "node:url";
        const __filename = fileURLToPath(import.meta.url);
    ```
  
    * `import.meta.url` 是 ES2020 (ES11) 的行为，当然 ESNext 也支持，在其他要编译为 ESM 模块的地方使用时你或许需要牢记这一点。
    * 严禁使用 `new URL(import.meta.url).pathname` 这种写法，会引发不同平台表现不一致和错误，详见 [`url.fileURLToPath(url)`](http://nodejs.cn/api/url/url_fileurltopath_url.html)（很多地方都告诉你这么写其实是不对的）

   对包的寻址，CJS 中可以使用 [enhanced-resolve](https://github.com/webpack/enhanced-resolve)，ESM 模块可以使用 [import-meta-resolve](https://github.com/wooorm/import-meta-resolve)，它是 `import.meta.resolve` 的 polyfill。
   目前开发 node.js  ESM 库，所缺少的 ESM 模块工具，可以参考 [unjs/mlly](https://github.com/unjs/mlly)。

- ESM 是可以向下兼容 CJS 的。但是要 CJS 向上兼容，导入 ESM 模块，就会很麻烦，必须要使用 `dynamic import()`。

  ```ts
  // main.cjs
  import("./fileA.mjs").then((ctx) => {
    console.log(ctx.name); // "mjs"
  });
  ```

 在 NodeJS 中想要同步和动态导入 ES6 模块，可以考虑：[import-sync](https://github.com/nktnet1/import-sync)

### `.cts` 和 `.mts` 文件
随着 Node.js 12 的引入和 ES 模块支持的增加，TypeScript 引入了新的文件扩展名，更好地支持 Node.js 中的 CommonJS（CJS）和 ECMAScript Modules（ESM）：

- `.cts`（Classic TypeScript）：这是一个被视为 CommonJS 模块的 TypeScript 文件。它相当于使用 `--module commonjs` 编译选项。
- `.mts`（Modern TypeScript）：这是一个被视为 ES 模块的 TypeScript 文件。它相当于使用 `--module esnext` 编译选项。

| TypeScript source file extension | Compiled JavaScript file extension | Generated type declaration file extension |
| :------------------------------- | :--------------------------------- | :---------------------------------------- |
| `.ts`                            | `.js`                              | `.d.ts`                                   |
| `.cts`                           | `.cjs`                             | `.d.cts`                                  |
| `.mts`                           | `.mjs`                             | `.d.mts`                                  |


指定 `module: node16` 或 `module: nodenext` 的设置使 TypeScript 的模块系统更接近 Node.js 的行为，提供了更细粒度的模块类型控制，并支持混合使用 CommonJS 和 ESM。


### `require` 与 `import`

1. `require` 调用同步 CJS 模块加载器
2. `require` 在 CJS 模块中使用，在 ES 模块中可以使用 [`module.createRequire()`](https://nodejs.org/api/module.html#modulecreaterequirefilename) 构造 `require` 函数。
3. `require` 只能引用 CJS 模块，引用 ES 模块将抛出 `ERR_REQUIRE_ESM`，因为不可能从同步模块调用异步模块加载器。[node.js 22](https://nodejs.org/en/blog/announcements/v22-release-announce) adds `require()` support for synchronous ESM graphs under the flag `--experimental-require-module`，该 [pr](https://github.com/nodejs/node/pull/51977))
4. `import` 调用异步 ES 模块加载器
5. `import` 只能在 ES 模块调用
6. `import` 可以引用 ES 和 CJS 模块。

## `require('xxx').default`

有时，node 环境下 require 一个包时，会发现需要这么引入：`const axios = require('axios').default`。

其实很简单，axios 这个模块是一个 ES6 模块，是用 ESModule 的标准写的。其内部是这样导出的：`export default xxx`，相当于 `export {xxx as default}`。对外把变量命名为了 `default`，ES6 `default` 语法糖让我们在 import 时，不需要写 `default` 了。

而 require 在 commonJS 中对应 `module.exports` 的部分。当 require 去处理 ES6 的导出时，对它而言，ES6 的代码就会被转化成这样：

```js
module.exports = {
  default: xxx,
};
```

于是，在 ES6 语法糖 `default` 的影响下，require 导出的不是 `xxx` 这个对象，`xxx` 是作为 `default` 的值，被包在一个更大的对象里。

## JSON 模块

```js
import packageConfig from "./package.json" with { type: "json" };
```

必须使用 `with` 语法，会导致 node 不能识别。所以不要使用这种语法。

## Convert CommonJS to ESM

可以借助一下工具实现转化：
- [to-esm](https://www.npmjs.com/package/to-esm)
- [cjs2esm](https://www.npmjs.com/package/cjs2esm)
- [cjstoesm](https://www.npmjs.com/package/cjstoesm)
- [babel-plugin-transform-commonjs](https://github.com/tbranyen/babel-plugin-transform-commonjs)

## 库的开发

* 有依赖 commonjs 包的库，考虑输出多包；
* 同时面对多种环境，优先考虑输出多包；
* 使用 main、module、types、exports 来提供了更细粒度的入口控制（不需要 `"type": "module"`）。
  
1. 只支持 ESM，放弃对 CJS，因为 CJS 被抛弃是注定的事情
2. 使用 typescript  进行项目的开发，生成 `.d.ts` 类型定义文件

类似：
```
```

> 更多查看：轮子哥 [sindresorhus](https://github.com/sindresorhus) 的 [Pure ESM package](https://gist.github.com/sindresorhus/a39789f98801d908bbc7ff3ecc99d99c)

### 相关构建工具
- [unjs/mlly](https://github.com/unjs/mlly) - 填补在 Node 中实用 ES 模块进行开发的实用库
- [unjs/unbuild](https://github.com/unjs/unbuild) - node 库构建工具
- [isaacs/tshy](https://github.com/isaacs/tshy)
- [privatenumber/pkgroll](https://github.com/privatenumber/pkgroll)
- [huozhi/bunchee](https://github.com/huozhi/bunchee)

### 最佳实践
1. 避免在同一个项目中混用 CJS 和 ESM。建议新项目优先使用 ESM 规范，以减少转换和兼容性问题（All new JavaScript should be written in ESM for future proofing）。
2. 使用 Typescript 进行开发，或者编写良好的 JSDoc，还需要提高良好的测试（vitest、jest 工具等）。
3. 内部所有 import（包括动态，除了第三方库），都需要指定好后缀名（typescript 编译，需要通过相应工具添加后缀）。
4. 推荐使用 named export（命名导出）导出模块成员。default export（默认导出）在灵活性和可维护性、重构难度都不友好，在某些打包工具中，也容易发生意想不到的问题。
5. 通过 `sideEffects` 来声明哪些文件是具有副作用的，进行 Tree-shaking 时应该保留。参考：[深入理解sideEffects配置](https://libin1991.github.io/2019/05/01/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3sideEffects%E9%85%8D%E7%BD%AE/) [Webpack#Mark the file as side-effect-free](https://webpack.js.org/guides/tree-shaking/#mark-the-file-as-side-effect-free)
  有许多代码模式会被判定为具有 `sideEffects`，包括：
    * 顶层函数调用，如：`console.log('a')`；
    * 修改全局状态或对象，如：`document.title = 'new Title'`；
    * IIFE 函数；
    * 动态导入语句，如：`import('./mod')`；
    * 原型链污染，如：`Array.prototype.xxx = function (){xxx}`；
    * 非 JS 资源：Tree-shaking 能力仅对 ESM 代码生效，一旦引用非 JS 资源则无法树摇；
    * 等等；
6. STOP using **Barrel files**（也称为索引文件或桶文件）在TypeScript（TS）项目中经常被使用，它们是一种特殊的文件，仅用于导出其他文件的内容，而本身不包含任何业务逻辑代码。参考 [Stop using barrel files, now!](https://mp.weixin.qq.com/s/I1i-dhgFgmBsfNrpPW-akQ) [Barrel files and why you should STOP using them now](https://dev.to/tassiofront/barrel-files-and-why-you-should-stop-using-them-now-bc4) [Speeding up the JavaScript ecosystem - The barrel file debacle](https://marvinh.dev/blog/speeding-up-javascript-ecosystem-part-7/)

## 参考

- [nonara/ts-patch](https://github.com/nonara/ts-patch)

- [Modules: ECMAScript modules](https://nodejs.org/api/esm.html#modules-ecmascript-modules)

- [ts-bridge/ts-bridge](https://github.com/ts-bridge/ts-bridge)

- [GervinFung/ts-add-js-extension](https://github.com/GervinFung/ts-add-js-extension)

- [tsc-esm-fix](https://www.npmjs.com/package/tsc-esm-fix)

- [Node.js + Typescript with ESM in 2023](https://medium.com/codememo/node-js-typescript-with-esm-in-2023-6b87e6f8e737)
