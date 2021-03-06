> 孤山寺北贾亭西，水面初平云脚低。几处早莺争暖树，谁家新燕啄春泥。乱花渐欲迷人眼，浅草才能没马蹄。最爱湖东行不足，绿杨阴里白沙堤。 ———— 唐.白居易《钱塘湖春行》

自从 Node8.5 以后， [Node 开始支持引入 ES 模块](https://github.com/nodejs/node/blob/master/doc/changelogs/CHANGELOG_V8.md#8.5.0)。在新开的项目中，笔者尝试使用了这种方式。由于目前 NodeJS 对于 ES 模块尚属试验性支持，因此需要在启动时候加入参数：`--experimental-modules`，完整的命令如：`node --experimental-modules index.mjs`。这样程序就愉快地运行起来了。不过当笔者尝试加入一些新的依赖之后问题出现了：

![](https://p0.ssl.qhimg.com/t01000138a5b171d59b.png)

这个问题在于，很多 NodeJS 依赖包，由于使用 CommonJS 的方式编写，对于 ES Module 的支持尚不完善。因此包在使用过程中，存在一定的兼容问题。

这篇文章我们就来谈谈，在 NodeJS 环境下，CommonJS 和 ES Module 区别和联系，以及互操作的方法。

在 NodeJS 诞生时候，ES Module 模块系统还没有诞生，因此 NodeJS 最初采用的是 CommonJS 模块系统。我们之前所使用的 NodeJS 包，大部分都是 CommonJS 模块。CommonJS 模块和 ES Module 之间有很大的区别。这也就是为什么要求使用模块的 js 使用`.mjs`各自对待的原因。 下面这张图阐述了 NodeJS 对于两种模块系统的解析流程。

![](https://p1.ssl.qhimg.com/t0194077ced433a626b.png)

那么它们在行为上有什么区别呢？

主要有两点：

一、引入模块时机的区别：CommonJS 模块是运行时加载，换句话说是在 NodeJS 脚本执行时才加载进来，而 ES Module 则是在静态分析时候就确定了引用关系。这有点像给目标模块建立了一个符号链接，或者说建立了一个指针。从这个意义上说，ES Module 的 import 的加载效率应该略高于 CommonJS。而 CommonJS 这种特性也给动态加载模块提供了可能。
二、CommonJS 模块加载实际上是把 module.exports 对象进行了一次拷贝。因此当原模块的存储在栈空间的值，也就是基本类型值在第一次加载过程中已经被缓存掉，当它发生了改变，不会影响引入这个模块的代码的值。读者可以尝试在加载后打印下`require.cache`看一看缓存的结构。下面看这个例子：

```JavaScript
// cjs.js
var inner = 3;
let innerIncrease = () => {
  inner++;
}

module.exports = {
  inner,
  innerIncrease
};
```

```JavaScript
// main.js
var mod = require('./cjs');

console.log(mod.inner);
mod.innerIncrease();
console.log(mod.inner);
```

我们如果执行 `node main.js` 会发现两次执行的结果都为 3。那是因为 inner 的值在第一次引入时候被缓存掉了。

如果上述代码使用 ES module。因为都是引用值，所以外部可以感知内部变化。

```JavaScript
// mjs.mjs
export let inner = 3;
export let innerIncrease = () => {
  inner++;
}
```

```JavaScript
// main.mjs
var mod = require('./mjs.mjs');

console.log(mod.inner);
mod.innerIncrease();
console.log(mod.inner);
```

这里因为"import"的都是引用，会内外同步，因此依次打印出 3 和 4。

如果你启用了`--experimental-modules`，则意味这几件事：

1. 对于同样的文件名，当模块没有写明扩展名时，加入`--experimental-modules`的总是先去加载`.mjs` 扩展名的同名模块，如果没有才会去加载.js。
1. 只要使用 import/export 命令的，必须使得扩展名为 .mjs。
1. .mjs 文件中不能使用 require 函数，否则会报`ReferenceError: require is not defined`错误。
1. import 命令是异步加载，意思是，后面的模块不会等待前面的加载完再去加载。如果有先后依赖关系的模块，不能在 import 中同级写，这个和 CommonJS 模块 不一样。
1. 顶层文件，`this`指向`undefined`。`arguments`、`require`、`module`、`exports`、`__filename`和`__dirname`均为`undefined`

既然差异很难弥合，那么用扩展名来区分两套体系从目前来看是很必要的。但是有时候，互相操作又是不可避免的。

那么如何互操作呢？

如果 ES Module 需要加载 CommonJS 模块，NodeJS 将把 module.exports 当作一个对象赋给 default 对象，即相当于：`export default {...module.exports 对象}`

假设我们有 cjs 模块 cjs.js，如下：

```JavaScript
// cjs.js
module.exports = {
    "Hello": "你好",
    "World": "世界"
}
```

我们可以在需要的 mjs 文件中这样使用：

```JavaScript
// main.mjs
import cjsExport from './cjs.js'
// 以下两种也可以
/***
* import {default as cjsExport} from './cjs.js'
* import * as cjsExport from './cjs.js'
***/
const {Hello, World} = cjsExport
```

但是，这种方式是不允许的：`import { someExports } from 'module'`。

那么 CommonJS 能否加载 ES Module 呢？答案是肯定的，不过稍微有点麻烦。同时，使用这个特性需要在 NodeJS 9.7.0 版本以上，同时加入`--experimental-modules`，[这里](https://github.com/nodejs/node/blob/master/doc/changelogs/CHANGELOG_V9.md#9.7.0)是当时的 changelog

```Javascript
// es.mjs
export default {
    "a" : 1,
    "b" : 2
}
```

```Javascript
// main.mjs
(async _ => {
    const es = await import('./es.mjs');
    console.log(es);
})()
```

其它几种 export 语法也可以用这种方式导入，想要知晓所有 export 语法的，可以参考高峰老师的之前在周刊发表的[这篇文章](https://mp.weixin.qq.com/s/bIU_FvesizFJ3D_6KWRPHA)。

回到我们开头提出的问题，在 mjs 模块里，可以使用上面提到的相应的加载 CommonJS 模块的写法，就可以正常运行了。

随着 ES Module 越来越完善，我们可以期待，今后 ES module 会越来越多。我们看到在现在这个多种模块机制并存的时代，很多重要的库在新版本的源码中都做了适配，基本的方案是：利用扩展名引入的优先级，提供多种扩展名的文件。如.js 适应 CommonJS，.mjs 文件适配 ES module，这种方案也使得库的使用者，比较平滑地切换各种模块系统。当然，各种预编译器也可以在不得已的时候提供必要的帮助。

![](https://p3.ssl.qhimg.com/t0176aec319809a43b5.png)

### 参考资料

1. http://es6.ruanyifeng.com/#docs/module-loader
2. https://nodejs.org/api/esm.html
3. https://www.zcfy.cc/article/es-modules-and-node-js-hard-choices-477.html
