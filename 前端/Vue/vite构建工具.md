### dev环境构建核心原理

利用浏览器对ES Module的原生支持，不过会做一些优化，比如Vite会有一个预编译过程，把编译之后的代码缓存到node_modules/.vite目录下，这点可以通过F12查看源代码就能看到，预编译主要做了如下事情。

* 将使用CommonJS模块的依赖转成ES Module，因为浏览器不认识CommonJs模块代码。

* 减少模块和请求数量，因为浏览器加载模块时，会递归请求所有的依赖模块文件，即便代码中没有使用到这些模块，比如lodash-es模块内部就导出了几百个子模块，每个子模块都是一个单独的文件，这会导致当代码中出现 `import { debounce } from 'lodash-es'`时浏览器发起几百次请求，尽管只使用了这个防抖函数而已。Vite在预编译时将这些有许多内部模块的依赖关系转换为单个模块，以提高后续页面加载性能。通过预构建 `lodash-es` 成为一个模块，也就只需要一个 `HTTP` 请求了！

### Tree Shaking

打包工具可以基于ES Module分析依赖关系，从而去除掉没有使用的代码，使得打包之后的文件体积变小。

```js
// utils.js
export const used = () => 'used';
export const unused = () => 'unused';

// main.js
import { used } from './utils.js';
console.log(used());
```

构建后的代码中 unused 会被移除。这在组件库，比如antd会很实用，如果只用了antd的几个组件，而不用导入整个antd库，实现按需引入的效果。

**注意: vite在dev环境中为了编译速度，不会做Tree Shaking优化。**

