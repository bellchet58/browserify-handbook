# 介绍

本文是substack/browserify-handbook 的简体中文翻译. 由magicdawn(magicdawn@qq.com)翻译.
这篇文章介绍了如何使用[browserify](http://browserify.org)来构建模块化的应用.

[![cc-by-3.0](http://i.creativecommons.org/l/by/3.0/80x15.png)](http://creativecommons.org/licenses/by/3.0/)

browserify是一个使用[node支持的CommonJS模块标准](http://nodejs.org/docs/latest/api/modules.html)
来为浏览器编译模块.

你可以使用browserify来组织你的代码以及第三方库即使你只使用[node](http://nodejs.org)的npm来安装打包代码.

browserify所使用的模块系统跟node使用的是一致的, 那些发布到[npm](https://npmjs.org)上原本只想运行在node上
的包也可以很好的运行在浏览器中.

慢慢地, 开发者们把node端以及通过browserify运行在浏览器端的代码发布到npm, npm上还有一些只运行在浏览器端的包.
[npm is for all javascript](http://maxogden.com/node-packaged-modules.html),npm是JavaScript的包管理器, 
不管是Frontend还是Backend.

# 目录

- [introduction](#introduction)
- [table of contents](#table-of-contents)
- [node packaged manuscript](#node-packaged-manuscript)
- [node packaged modules](#node-packaged-modules)
  - [require](#require)
  - [exports](#exports)
  - [bundling for the browser](#bundling-for-the-browser)
  - [how browserify works](#how-browserify-works)
  - [how node_modules works](#how-node_modules-works)
  - [why concatenate](#why-concatenate)
- [development](#development)
  - [source maps](#source-maps)
    - [exorcist](#exorcist)
  - [auto-recompile](#auto-recompile)
    - [watchify](#watchify)
    - [beefy](#beefy)
    - [wzrd](#wzrd)
    - [browserify-middleware, enchilada](#browserify-middleware-enchilada)
    - [livereactload](#livereactload)
    - [browserify-hmr](#browserify-hmr)
    - [budo](#budo)
  - [using the api directly](#using-the-api-directly)
  - [grunt](#grunt)
  - [gulp](#gulp)
- [builtins](#builtins)
  - [Buffer](#Buffer)
  - [process](#process)
  - [global](#global)
  - [__filename](#__filename)
  - [__dirname](#__dirname)
- [transforms](#transforms)
  - [writing your own](#writing-your-own)
- [package.json](#package.json)
  - [browser field](#browser-field)
  - [browserify.transform field](#browserifytransform-field)
- [finding good modules](#finding-good-modules)
  - [module philosophy](#module-philosophy)
- [organizing modules](#organizing-modules)
  - [avoiding ../../../../../../..](#avoiding-)
  - [non-javascript assets](#non-javascript-assets)
  - [reusable components](#reusable-components)
- [testing in node and the browser](#testing-in-node-and-the-browser)
  - [testing libraries](#testing-libraries)
  - [code coverage](#code-coverage)
  - [testling-ci](#testling-ci)
- [bundling](#bundling)
  - [saving bytes](#saving-bytes)
  - [standalone](#standalone)
  - [external bundles](#external-bundles)
  - [ignoring and excluding](#ignoring-and-excluding)
  - [browserify cdn](#browserify-cdn)
- [shimming](#shimming)
  - [browserify-shim](#browserify-shim)
- [partitioning](#partitioning)
  - [factor-bundle](#factor-bundle)
  - [partition-bundle](#partition-bundle)
- [compiler pipeline](#compiler-pipeline)
  - [build your own browserify](#build-your-own-browserify)
  - [labeled phases](#labeled-phases)
    - [deps](#deps)
      - [insert-module-globals](#insert-module-globals)
    - [json](#json)
    - [unbom](#unbom)
    - [syntax](#syntax)
    - [sort](#sort)
    - [dedupe](#dedupe)
    - [label](#label)
    - [emit-deps](#emit-deps)
    - [debug](#debug)
    - [pack](#pack)
    - [wrap](#wrap)
  - [browser-unpack](#browser-unpack)
- [plugins](#plugins)
  - [using plugins](#using-plugins)
  - [authoring plugins](#authoring-plugins)

# node packaged manuscript

你可以通过npm安装这本handbook, 很恰当. 只需要键入:
```
# 简体中文翻译
npm install browserify-handbook-zhcn

# english version
npm install browserify-handbook
```

现在你可以使用 `browserify-handbook` 命令使用`$PAGER` 环境变量指定的阅读器打开这份README文件.
当然你也可以继续阅读, 就像你现在这样.

# node packaged modules

在我们深入如何使用browserify以及它是如何工作的, 了解nodejs支持的commonjs 模块系统如何工作十分重要.

## require

在node中, 有一个`require()` 函数可以从其他文件中加载代码, 如果你使用npm来安装一个模块:

```
npm install uniq
```

然后有一个叫 `nums.js` 的文件, 我们可以 `require('uniq')`:

```
var uniq = require('uniq');
var nums = [ 5, 2, 1, 3, 2, 5, 4, 2, 0, 1 ];
console.log(uniq(nums));
```

使用node运行这个小程序的输出是:
```
$ node nums.js
[ 0, 1, 2, 3, 4, 5 ]
```

你可以通过给require函数传一个以 `.` 开头的string来require其他相对路径的文件.
例如, 要从 `main.js` 加载 `foo.js`, 在 `main.js` 中你可以:
``` js
var foo = require('./foo.js');
console.log(foo(4));
```
如果 `foo.js` 是在父文件夹, 你可以使用 `../foo.js` 代替:
``` js
var foo = require('../foo.js');
console.log(foo(4));
```

同样, 对于其他类型的相对路径, 相对路径总是以调用require函数的那个文件的位置来寻找.

注意 `require()` 返回了一个function, 然后我们把它赋值给了 `uniq` 变量. 我们可以另取一个名字, 也可以正常工作. 
`require()` 返回的是你指定的模块的导出值(exports).

`require()` 的工作方式不同于其他很多的模块系统, 其他很多模块系统的 imports 是类似于将名称暴露在全局空间或者
本地文件作用域的语句, 其中用到的名称不是模块使用者能够控制的. 

在node下通过 `require()` 方式导入的代码, 别人阅读你的程序的源代码的时候可以轻易的知道每一段代码或者某一个函数从
哪里来. 当应用中的模块数量多起来后这种方式可伸缩性更好. 

## exports

从一个文件导出只导出一个东西, 别的模块可以导入它, 将要导出的东西赋值给 `module.exports` :
``` js
module.exports = function (n) {
    return n * 111
};
```
现在如果某个模块 `main.js` 需要加载你的 `foo.js` , `require('./foo.js')` 的返回值会是刚导出的function:
``` js
var foo = require('./foo.js');
console.log(foo(5));
```

程序将会输出:

```
555
```

你可以通过 `module.exports` 导出任何类型的值, 不只是function类型.
例如, 下面这个例子也是极好的:
``` js
module.exports = 555
```

下面这样也是:

``` js
var numbers = [];
for (var i = 0; i < 100; i++) numbers.push(i);

module.exports = numbers;
```
还有另一种导出的方法, 通过object导出. 在这里使用 `exports` 来代替 `module.exports` :
``` js
exports.beep = function (n) { return n * 1000 }
exports.boop = 555
```

上面的程序与下面是相同的:

``` js
module.exports.beep = function (n) { return n * 1000 }
module.exports.boop = 555
```
因为 `module.exports` 与 `exports` 是相同的, 被初始化为一个空的object.

注意你不能这样做:

``` js
// this doesn't work
exports = function (n) { return n * 1000 }
```

因为导出值是附加在 `module` 对象上的, 所以为 `exports` 赋一个新值而不是 `module.exports` 会掩盖初始的引用.
如果你是要导出一个单项, 总是使用 `module.exports` :
``` js
// instead
module.exports = function (n) { return n * 1000 }
```
如果你仍然很困惑, 可以尝试理解下模块系统是如何工作的:
``` js
var module = {
  exports: {}
};

// 如果你requore一个module, 模块系统会简单的将文件包装一层function
(function(module, exports) {
  exports = function (n) { return n * 1000 };
}(module, module.exports))

console.log(module.exports); // 现在module.exports仍然还是空的object 
```
通常情况下, 你会使用 `module.exports` 来导出单个function 或者 构造器, 因为一个模块只做一件事通常来说是最好的做法.
`exports` 一开始是最主要的导出方式, `module.exports`是替补的, 但是 `module.exports` 被证明更加实用, 有用, 因为
其直接, 清晰, 防止冗余.

在早期的时候,下面这种方式更通用:

foo.js:

``` js
exports.foo = function (n) { return n * 111 }
```

main.js:

``` js
var foo = require('./foo.js');
console.log(foo.foo(5));
```

但是请注意 `foo.foo` 有点多余, 使用 `module.exports` 变得更加清晰:

foo.js:

``` js
module.exports = function (n) { return n * 111 }
```

main.js:

``` js
var foo = require('./foo.js');
console.log(foo(5));
```

## bundling for the browser

在node中运行一个模块, 你需要从某一处开始, 向 `node` 命令传递一个file参数来运行file.
```
$ node robot.js
beep boop
```

在browserify中, 做法是一样的, 只不过是生成一个拼接了所有需要的JavaScript文件的导向标准输出的流, 可以使用 `>` 运算符
写到文件中.
```
$ browserify robot.js > bundle.js
```

现在 `bundle.js` 中包含 `robot.js` 运行所必须的javascript内容. 只需将其放到某个html的单个script块中即可:
``` html
<html>
  <body>
    <script src="bundle.js"></script>
  </body>
</html>
```
小提示: 将script标签放在`</body>`之前,script 可以使用所有的dom元素,而不必等待onready事件。
bundling构建过程大有可为,请查看 bundling 部分。

## how browserify works

Browserify从指定的entry文件开始, 搜寻所有的 `require()` 调用, 通过[静态分析](http://npmjs.org/package/detective)源代码的
[AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree)树.

对于每个 `require(string module)` 调用, browserify 将被require的string转变为文件路径, 然后在这些文件中再次搜寻 `require()` 调用,
直到所有的依赖都被访问到了.

所有被访问到的文件被拼接成单个javascript文件, 带有一个最轻便的 `require()` 定义, 能够将静态分析得到的结果与拼接文件中的id相对应.

这意味着你生成的bundle文件完全是自包含的, 包含应用运行所需的所有代码, 只增加了一点微不足道的开销(指前面的require定义).

如需了解更多关于browserify的运行原理, 请查看 编译器流程线(compiler pipeline)部分.

## how node_modules works

node有一个不同于其他平台的机智的模块查找算法, node的查找机制默认是基于本地文件的, 而不是基于一个包含查找路径的数组, 例如
命令行工具中的 `$PATH` 环境变量那样.

如果你从 `/beep/boop/bar.js` 进行 `require('./foo.js')` , node将会寻找 `/beep/boop/foo.js` 文件, 
以 `./` 或者 `../` 开头的路径总是以调用 `require()` 的文件为基准开始寻找.

但是如果你require一个非相对路径的模块名例如 `require('xyz')` 从 `/beep/boop/foo.js`, node
会按照先后顺序搜索下面的路径, 从第一个匹配停止, 没匹配项则报错误.
```
/beep/boop/node_modules/xyz
/beep/node_modules/xyz
/node_modules/xyz
```

对于每一个 `xyz` 文件夹, 如果存在, node将会首先查看 `xyz/package.json` 文件的 `main` 域是否存在. 
`main` 域定义了 `require()` 文件夹时该加载哪一个文件.

例如, 如果 `/beep/node_modules/xyz` 是第一个匹配项, 而且`/beep/node_modules/xyz/package.json`
包含了:
```
{
  "name": "xyz",
  "version": "1.2.3",
  "main": "lib/abc.js"
}
```

那么 `/beep/node_modules/xyz/lib/abc.js` 文件的exports值就是 `require('xyz')` 的返回值.
如果 `package.json` 文件不存在, 或者 `main` 域不存在, 会采用默认值 `index.js` .

```
/beep/node_modules/xyz/index.js
```

如果有必要, 你可以深入到一个package内部去加载某一个文件.
例如要加载 `dat` 这个包的 `lib/clone.js` 文件, 可以:
```
var clone = require('dat/lib/clone.js')
```

递归的node_modules机制将会顺着文件夹层级向上找到第一个 `dat` 包, 然后从该位置找 `lib/clone.js` .
这种方法可以在任何可以使用 `require*('dat')` 的地方使用.

node同样包含一个搜寻路径数组的机制, 但是这种机制已经被废弃, 你应该使用 `node_modules` 除非
你有很好的理由不去使用它.

node模块查找算法以及npm安装的模块的好处在于, 你永远不会碰到版本冲突的情况, 不同于其他所有的平台, npm
会将每个模块自己的依赖安装到模块内部的你的_modules文件夹.

每一个包都有自己的本地_modules文件夹, 包含了此包的所有依赖, 而且这些依赖的依赖也有自己的node_modules文件夹,
递归向下(recursively all the way down).

这意味着你可以在一个应用中使用同一个库的不同版本, 大大降低了迭代API时需要的协调开销. 这种特性对于像npm这样没有对
包的发布, 组织进行鉴定授权的生态系统非常重要. 每个人都可以简单的按照自己喜欢发布他们的包, 而不必担心他们选择的
版本可能影响同一个应用下其他的依赖.

你可以看到 `node_modules` 是如何在你本地的应用中工作的. 请查看 `avoiding ../../../../../../..`部分了解更多.

## 为什么使用合并文件的方式

Browserify 是运行在server端的build过程。它会生产一个包含所有依赖的单文件bundle。
下面列举了module system的其他实现方式，以及他们的优缺点。

### window globals

每个js文件在window host 对象上定义一个全局变量 或者 自行组织一个namespace的形式。
这种方式因为每个文件都需要一个script 标签,不能scale up,大规模应用,而且script标签的顺序很重要.
重构或维护这种方式构成的应用很困难,但是所有的浏览器都原生支持这种方式.不需要其他的server 端红菊的参与.
每个script标签的http request导致应用很慢.

### 拼接

将所有的脚本在server端被拼接在一起, 而不是使用window全局变量, 代码仍然是顺序敏感的, 而且难以维护, 
但是加载的更快了, 因为只有一个 `<script>` 标记, 只需要一次HTTP请求.

没有source maps, 抛出的异常有偏移(?), 难以找到对应的源文件. 

### AMD

每一个文件都被包装了一层 `define()` 函数以及一个callback回调, 
[这就是AMD](http://requirejs.org/docs/whyamd.html), 而不是使用 `<script>` 标记.
``` js
define(['jquery'] , function ($) {
    return function () {};
});
```

你可以给你的模块命名, 以第一个参数传过去, 这样别的模块就可以加载它. 
There is a commonjs sugar syntax that stringifies each callback and scans it for
`require()` calls
[with a regexp](https://github.com/jrburke/requirejs/blob/master/require.js#L17).

这种方式写的代码, 比拼接模式或者使用全局变量模式在顺序敏感性方面要小, 因为加载顺序
是通过解析显示指定的依赖信息得出.

为了性能原因, 大多数情况下AMD也需要在server端打包成一个单文件, 而在开发过程中, 使用AMD的异步特性更常见.

### bundling commonjs server-side

如果你为了性能原因以及语法糖的便利要在server端有一个构建过程, 
为什么不废弃(原文 scrap)整个AMD业务, 使用commonjs打包?
在工具的帮助下,你可以解决模块的依赖顺序敏感的问题, 以及开发环境生产环境更加一致, 以及更加健壮. 
CommonJS的语法更加方便, 又有node 和 npm导致commonjs的生态系统爆发.

你可以再node & browser之间无缝的分享代码. 只需要一个build步骤以及生成source maps和自定重新rebuild的工具.

而且 , 我们可以使用node的模块查找方法来防止模块版本不一致的问题 , 我们可以在同一个app里面使用同一个lib的不同版本 , 
应用还可以工作. 为了节省空间 , 可以使用 depupe , 就是npm depupe , 详见 https://docs.npmjs.com/cli/dedupe.

# development

基于拼接模式会有一些缺点 , 但是这些问题都可以用响应的工具解决.

## source maps

Browserify 支持一个 `--debug`/`-d` 选项以及 `opts.debug` 参数来开启source maps支持.
Source maps 告诉浏览器如何转换代码的行和列以得到build之前的源代码. 

Source maps 包含所有的原始代码,所以你可以简单的把bundle 文件放在服务器上而不必确认所有的原始文件
都放在正确的相应位置.

### exorcist

将source maps 放在build之后的bundle里的缺点就是 build的文件是原来的两杯大. 本地测试是没什么问题的, 但是不适合将source maps
inline到生产环境. 然而可以使用[exorcist](https://npmjs.org/package/exorcist) 将source maps放到独立的文件中,如
`bundle.map.js`. 

``` sh
browserify main.js --debug | exorcist bundle.js.map > bundle.js
```

## 自动打包

每次运行一条命令去重新打包会很慢, 以及乏味. 幸运的是有许多的工具可以解决这个问题, 其中某些支持不同程度的热更新(live-reloading), 其他的则是传统的手动刷新机制.

这里有一堆可以使用的工具, 但是npm上有更多可用的工具. 

这里介绍了许多不同的工具, 围绕着不同的利弊权衡以及开发方式. 找到最适合的你的期望以及经验的工具还需要你点点工作, 但是我认为这种分歧是能够帮助码农变得更高效, 以及提供更多创造、实验的空间. 我认为周边工具的多样化, 以及保持browserify核心最小在中长期策略中是比从挑选出一些更优秀的winners继承进browserify核心这种方式更加健康. 上述集成的方式会在有意义的版本中创建浩劫以及对核心的破坏.

就是说, 这里有一些工具, 你可以以之建立起你的开发流程. 但是留个心眼, 还有许多其他的工具没有被列出!

### [watchify](https://npmjs.org/package/watchify)

你可以使用 `watchify` 与 `browserify` 互换, 但是watchify不是只写入到输出文件一次, watchify会写入到输出文件, 然后监控你的依赖树中的所有文件改变. 当你修改一个文件时, 新的输出文件将会很快地被重新写入, 因为使用到了聚合缓存. 

你可以使用 `-v` 参数使每次写入新文件的时候打印一条信息:

```
$ watchify browser.js -d -o static/bundle.js -v
610598 bytes written to static/bundle.js  0.23s
610606 bytes written to static/bundle.js  0.10s
610597 bytes written to static/bundle.js  0.14s
610606 bytes written to static/bundle.js  0.08s
610597 bytes written to static/bundle.js  0.08s
610597 bytes written to static/bundle.js  0.19s
```

这里有一个实用的配置代码片段, 当你使用package.json的fields字段来配置 browserify 和 watchify的使用:

``` json
{
  "build": "browserify browser.js -o static/bundle.js",
  "watch": "watchify browser.js -o static/bundle.js --debug --verbose",
}
```

使用 `npm run build` 来生成生产环境的文件, 使用 `npm run watch`来进行开发.

[了解更多 `npm run` 的使用](http://substack.net/task_automation_with_npm_run).

### [beefy](https://www.npmjs.org/package/beefy)

如果你更喜欢建立一个服务器来自动编译修改的文件, 查看 [beefy](http://didact.us/beefy/) 这个包.
只需要给beefy一个入口文件

```
beefy main.js
```

然后他会建立在一个http端口上建立输出服务器.

### [wzrd](https://github.com/maxogden/wzrd)

同beefy一样的理念, 但是形式更简洁 [wzrd](https://github.com/maxogden/wzrd)

只需 `npm install -g wzrd` 然后你可以:

```
wzrd app.js
```

然后打开 http://localhost:9966

### browserify-middleware, enchilada

若果你使用express, 请查看 [browserify-middleware](https://www.npmjs.org/package/browserify-middleware) 或者 [enchilada](https://www.npmjs.org/package/enchilada).

它们都提供了为express提供borswerify 生成服务的中间件.

### [livereactload](https://github.com/milankinen/livereactload)

livereactload 是一款为 [react](https://github.com/facebook/react) 提供当你更改代码时自动更新页面的工具. livereactload 是一个普通的transform,你可以使用 `-t livereactload` 加载它, 你应该查看该项目的说明文件 [project readme](https://github.com/milankinen/livereactload#livereactload) 了解更多.

### [browserify-hmr](https://github.com/AgentME/browserify-hmr)

browserify-hmr is a plugin for doing hot module replacement (hmr).

Files can mark themselves as accepting updates. If you modify a file that
accepts updates of itself, or if you modify a dependency of a file that accepts
updates, then the file is re-executed with the new code.

For example, if we have a file, `main.js`:

``` js
document.body.textContent = require('./msg.js')

if (module.hot) module.hot.accept()
```

and a file `msg.js`:

``` js
module.exports = 'hey'
```

We can watch `main.js` for changes and load the `browserify-hmr` plugin:

```
$ watchify main.js -p browserify-hmr -o public/bundle.js -dv
```

and serve up the static file contents in `public/` with a static file server:

```
$ ecstatic public -p 8000
```

Now if we load `http://localhost:8000`, we see the message `hey` on the page.

If we change `msg.js` to be:

``` js
module.exports = 'wow'
```

then a second later, the page updates to show `wow` all by itself.

Browserify-HMR can be used with
[react-hot-transform](https://github.com/AgentME/react-hot-transform) to
automatically allow all React components to be updated live in addition to code
using the `module.hot` API. Unlike
[livereactload](https://github.com/milankinen/livereactload), only modified
files are re-executed instead of the whole bundle on each modification.

### [budo](https://github.com/mattdesl/budo)

budo  提供一个browserify开发服务器, 主要提供增量更新以及LiveReload集成(也包括注入CSS).

使用如下命令安装
```sh
npm install budo -g
```

然后使用它运行你的entry文件:
```
budo app.js
```

这条命令在 http://localhost:9966 创建了一个服务器, 带有一个默认的 `index.html` 页面, 以及增量更新的功能. 这些请求是延迟的,直到生成文件结束, 所以你在页面更新过程中刷新会得到旧的或者空的生成文件.

使用 `--live` 来开启 LiveReload 功能:
```
budo app.js --live
```

## 直接使用browserify的API

你可以直接使用browserify的API和传统的 `http.createServer()` 的形式进行开发.
``` js
var browserify = require('browserify');
var http = require('http');

http.createServer(function (req, res) {
    if (req.url === '/bundle.js') {
        res.setHeader('content-type', 'application/javascript');
        var b = browserify(__dirname + '/main.js').bundle();
        b.on('error', console.error);
        b.pipe(res);
    }
    else res.writeHead(404, 'not found')
});
```

## grunt

[grunt-browserify](https://www.npmjs.org/package/grunt-browserify) 插件.

## gulp

如果你使用gulp构建, 你应该直接使用browserify API.

[这里](http://viget.com/extend/gulp-browserify-starter-faq)提供了一个向导介绍在gulp中使用browserify.
[这里](https://github.com/gulpjs/gulp/blob/master/docs/recipes/fast-browserify-builds-with-watchify.md)展示了如何在gulp使用watchify更快地构建browserify文件.

# 内置的包

为了使更多原本在node上使用的模块能在浏览器端工作, browserify提供了许多node核心模块的浏览器版本的实现:

* [assert](https://npmjs.org/package/assert)
* [buffer](https://npmjs.org/package/buffer)
* [console](https://npmjs.org/package/console-browserify)
* [constants](https://npmjs.org/package/constants-browserify)
* [crypto](https://npmjs.org/package/crypto-browserify)
* [domain](https://npmjs.org/package/domain-browser)
* [events](https://npmjs.org/package/events)
* [http](https://npmjs.org/package/http-browserify)
* [https](https://npmjs.org/package/https-browserify)
* [os](https://npmjs.org/package/os-browserify)
* [path](https://npmjs.org/package/path-browserify)
* [punycode](https://npmjs.org/package/punycode)
* [querystring](https://npmjs.org/package/querystring)
* [stream](https://npmjs.org/package/stream-browserify)
* [string_decoder](https://npmjs.org/package/string_decoder)
* [timers](https://npmjs.org/package/timers-browserify)
* [tty](https://npmjs.org/package/tty-browserify)
* [url](https://npmjs.org/package/url)
* [util](https://npmjs.org/package/util)
* [vm](https://npmjs.org/package/vm-browserify)
* [zlib](https://npmjs.org/package/browserify-zlib)


events, stream, url, path, 以及 querystring 在浏览器环境下特别有用.

另外, 如果浏览器侦测到了 `Buffer`, `process`, `global`,`__filename`, 或者 `__dirname`的使用, browserify会包含一个浏览器版本的实现.

这样如果一个有很多的buffer以及stream操作的模块, 在浏览器端也可以正常的工作, 只要不进行服务端的IO操作.

如果, 你之前没有使用过node, 这里有一些例子展示了每一个global变量是干啥的. 注意这些全局变量只有在你要使用他们的时候, browserify才会包含这些实现.

## [Buffer](http://nodejs.org/docs/latest/api/buffer.html)

在node端, 所有的文件和网络API都是使用Buffer块来传递数据. 在Browserify中, buffer API由[buffer](https://www.npmjs.org/package/buffer)模块提供. 它是使用Typed Array来实现, 对于旧的浏览器也有兼容处理.

下面这个例子展示了 "使用`Buffer`来转换一个base64格式string到hex格式":
```
var buf = Buffer('YmVlcCBib29w', 'base64');
var hex = buf.toString('hex');
console.log(hex);
```

输出

```
6265657020626f6f70
```

## [process](http://nodejs.org/docs/latest/api/process.html#process_process)

In node, `process` is a special object that handles information and control for
the running process such as environment, signals, and standard IO streams.

Of particular consequence is the `process.nextTick()` implementation that
interfaces with the event loop.

In browserify the process implementation is handled by the
[process module](https://www.npmjs.org/package/process) which just provides
`process.nextTick()` and little else.

Here's what `process.nextTick()` does:

```
setTimeout(function () {
    console.log('third');
}, 0);

process.nextTick(function () {
    console.log('second');
});

console.log('first');
```

This script will output:

```
first
second
third
```

`process.nextTick(fn)` is like `setTimeout(fn, 0)`, but faster because
`setTimeout` is artificially slower in javascript engines for compatibility reasons.

## [global](http://nodejs.org/docs/latest/api/all.html#all_global)

Node中, `global` 是全局head变量, 全局变量附加在head变量上, 就像浏览器中window一样. 在browserify中, `global` 是 `window` 的别名.

## [__filename](http://nodejs.org/docs/latest/api/all.html#all_filename)

`__filename` 是当前文件的路径, 每个文件各不相同. 为了防止泄露系统上的文件路径信息, 文件路径是以 传给 `browserify()` 的 `opts.basedir` 为root路径的, 默认为[当前工作路径](https://en.wikipedia.org/wiki/Current_working_directory) `process.cwd()`

如, 我们这里有一个 `main.js` 文件
``` js
var bar = require('./foo/bar.js');

console.log('here in main.js, __filename is:', __filename);

bar();
```

以及一个 `foo/bar.js` 文件如下:

``` js
module.exports = function () {
    console.log('here in foo/bar.js, __filename is:', __filename);
};
```

然后browserify以 `main.js` 起始点, 输出如下:
```
$ browserify main.js | node
here in main.js, __filename is: /main.js
here in foo/bar.js, __filename is: /foo/bar.js
```

## [__dirname](http://nodejs.org/docs/latest/api/all.html#all_dirname)

`__dirname` 代表的是当前文件所在的文件夹. 像 `__filename` 一样, `__dirname` 也是以 `opts.basedir` 为根文件夹的.

如下例子展示了 `__dirname` 是如何工作的:

main.js:

``` js
require('./x/y/z/abc.js');
console.log('in main.js __dirname=' + __dirname);
```

x/y/z/abc.js:

``` js
console.log('in abc.js, __dirname=' + __dirname);
```

输出:

```
$ browserify main.js | node
in abc.js, __dirname=/x/y/z
in main.js __dirname=/
```

# transforms

browserify支持一个灵活的转换系统, 可以转化初始的源代码, 而不是使用browserify去支持所有的东西.

在这种途径下, 你可以 `reuqire()` 使用coffee script语法编写的文件、模板文件、以及其他所有可以编译成JavaScript的文件.

要使用 [coffeescript](http://coffeescript.org) , 你可以使用 [coffeeify](https://www.npmjs.org/package/coffeeify) 转换. 确保先使用 `npm i coffeeify` 安装 `coffify` 然后使用如下命令:
```
$ browserify -t coffeeify main.coffee > bundle.js
```

或者直接使用browserify API 可以这样:
```
var b = browserify('main.coffee');
b.transform('coffeeify');
```

最妙的地方在于, 如果你使用 `--debug` 或者 `opts.debug` 开启了source maps支持, 生成的bundle.js 文件会将发生的异常错误的地方对应回原始的coffee文件. 这对于使用 firebug 或者 chrome inspector 来调试很有用.

## 编写自己的转换

转化器Transform 实现了一个简单的stram接口. 下面展示了一个简单的将 `$CWD` 转换为 `process.cwd()` 的转换器:
``` js
var through = require('through2');

module.exports = function (file) {
    return through(function (buf, enc, next) {
        this.push(buf.toString('utf8').replace(/\$CWD/g, process.cwd()));
        next();
    });
};
```


这个转换器函数在当前包的每个文件都调用一次, 而且反悔一个Transform stream去转换源代码. 这个流可以获取原始的源代码, 以及browserify从中获取转换后的内容.

只需简单的保存你的转换器至文件或者做成一个npm包, 然后使用 `-t ./your_transform.js` 使用.

了解更多关于stream的内容, 请查阅 [stream-handbook](https://github.com/substack/stream-handbook).

# package.json

## browser 字段
你可以在包中的 `package.json` 文件中定义一个 `browser` 字段, 这样可以让browserify将 `main` 字段覆盖掉, 或者覆盖其他单独的文件.

如果你有一个有着 `main.js` 入口的模块, 但是对于浏览器有特定版本的 `browser.js` ,你可以这样做:
``` json
{
  "name": "mypkg",
  "version": "1.2.3",
  "main": "main.js",
  "browser": "browser.js"
}
```
现在, 当别人在node端 `reuqire('mypkg')` 的时候, 他们会得到 `main.js` 的exports值, 当他们使用browserify的时候, 他们会得到 `browser.js` 的exports值.

将node端与browser端用这种方式去分开来, 比在代码运行时去检测是在node端还是browser端而去加载不同的模块要好. 如果node和browser的 `require()` 调用的是同一个文件, browserify的静态分析将会包含所有的被require的module, 不管是否会起到作用. 你可以使用object类型而不是字符串作为 `browser` 字段的值. 例如, 如果你想替换掉 `lib/` 文件夹中的一个文件, 替换为浏览器版本的, 你可以这样做: 

``` json
{
  "name": "mypkg",
  "version": "1.2.3",
  "main": "main.js",
  "browser": {
    "lib/foo.js": "lib/browser-foo.js"
  }
}
```

再或者, 你想替换掉你的包中require的其他包, 你可以这样做:
``` json
{
  "name": "mypkg",
  "version": "1.2.3",
  "main": "main.js",
  "browser": {
    "fs": "level-fs-browser"
  }
}
```

你可以忽略某些文件(设置他们的内容为空), 通过设置它们在browser字段中的值为 `false`:
``` json
{
  "name": "mypkg",
  "version": "1.2.3",
  "main": "main.js",
  "browser": {
    "winston": false
  }
}
```
设置的browser字段只作用域当前模块. 映射表不会向下作用于它的依赖包,或者向上作用雨依赖它的包. 这种孤立设计是为了防止模块之间的危害, 如此当你require一个模块的时候, 你不必担心它所带来的其他影响. 就像你不必担心你的本地配置会影响到在你的依赖树很深的模块.

## browserify.transform 字段

你可以在 `browserify.transform` 字段配置在模块加载的时候自动应用的转换器. 例如, 我们可以自动的应用[brfs](https://npmjs.org/package/brfs) 转化通过如下配置:
``` json
{
  "name": "mypkg",
  "version": "1.2.3",
  "main": "main.js",
  "browserify": {
    "transform": [ "brfs" ]
  }
}
```

现在, 在我们的 `main.js` 中, 我们可以:
``` js
var fs = require('fs');
var src = fs.readFileSync(__dirname + '/foo.txt', 'utf8');

module.exports = function (x) { return src.replace(x, 'zzz') };
```

然后 `fs.readFileSync()` 调用的结果就会内联在代码中, 而不需要内容消费者(指接受fs.readFileSync结果的变量)感知. 你可以在转换器的数组中应用任意多的转化器, 他们会按照先后顺序应用. 

像 `browser` 字段一样, package.json中的`transforms`只应用于当前包, 为了同样的原因.
### 配置转换器

有时候, 一个转换器可能需要配置选项. 可以使用下面的形式配置:

**命令行工具形式**
```
browserify -t coffeeify \
           -t [ browserify-ngannotate --ext .coffee --bar ] \
           index.coffee > index.js
```

**package.json 配置**
``` json
"browserify": {
  "transform": [
    "coffeeify",
    ["browserify-ngannotate", {"ext": ".coffee", bar: true}]
  ]
}
```


# finding good modules

Here are [some useful heuristics]()
for finding good modules on npm that work in the browser:
这里有[一些提示](http://substack.net/finding_modules), 对于在npm上寻找在浏览器端
工作的优秀模块:

* I can install it with npm
* code snippet on the readme using require() - from a quick glance I should see
how to integrate the library into what I'm presently working on
* has a very clear, narrow idea about scope and purpose
* knows when to delegate to other libraries - doesn't try to do too many things itself
* written or maintained by authors whose opinions about software scope,
modularity, and interfaces I generally agree with (often a faster shortcut
than reading the code/docs very closely)
* inspecting which modules depend on the library I'm evaluating - this is baked
into the package page for modules published to npm

* 可以使用npm来安装这个模块
* 项目说明(README)上的代码块在使用 `require` - 能够快速查看怎样集成到目前的工作中
* 对于项目有明确的目的, 且轻量
* 知道何时将控制权交给其他库 - 而不是尝试自己去做的更多
* 作者或维护者在软件领域、模块化、接口等方面的观点自己比较赞同(通常这比读代码/文档更快)
* 查看哪些模块依赖了正在被考察的这个包 - this is baked into the package page 
for modules published to npm(什么意思? 真心不懂)

其他标准类似github的star数量, 项目的更新活动, 或者一个一个有些的新手指南网页,
不是那么的可靠.

## module philosophy

人们常认为导出一系列方便的工具代码会是码农消耗代码的方式, 因为这是其他平台上码农
导出和导入代码的方式, 而且持续存在, 甚至在npm上也是.

However, this
[kitchen-sink mentality](https://github.com/substack/node-mkdirp/issues/17)
toward including a bunch of thematically-related but separable functionality
into a single package appears to be an artifact for the difficulty of
publishing and discovery in a pre-github, pre-npm era.

There are two other big problems with modules that try to export a bunch of
functionality all in one place under the auspices of convenience: demarcation
turf wars and finding which modules do what.

Packages that are grab-bags of features
[waste a ton of time policing boundaries](https://github.com/jashkenas/underscore/search?q=%22special-case%22&ref=cmdform&type=Issues)
about which new features belong and don't belong.
There is no clear natural boundary of the problem domain in this kind of package
about what the scope is, it's all
[somebody's smug opinion](http://david.heinemeierhansson.com/2012/rails-is-omakase.html).

Node, npm, and browserify are not that. They are avowedly ala-carte,
participatory, and would rather celebrate disagreement and the dizzying
proliferation of new ideas and approaches than try to clamp down in the name of
conformity, standards, or "best practices".

Nobody who needs to do gaussian blur ever thinks "hmm I guess I'll start checking
generic mathematics, statistics, image processing, and utility libraries to see
which one has gaussian blur in it. Was it stats2 or image-pack-utils or
maths-extra or maybe underscore has that one?"
No. None of this. Stop it. They `npm search gaussian` and they immediately see
[ndarray-gaussian-filter](https://npmjs.org/package/ndarray-gaussian-filter) and
it does exactly what they want and then they continue on with their actual
problem instead of getting lost in the weeds of somebody's neglected grand
utility fiefdom.

# organizing modules

## 避免 ../../../../../../..

Not everything in an application properly belongs on the public npm and the
overhead of setting up a private npm or git repo is still rather large in many
cases. Here are some approaches for avoiding the `../../../../../../../`
relative paths problem.

### symlink

The simplest thing you can do is to symlink your app root directory into your
node_modules/ directory.

Did you know that [symlinks work on windows
too](http://www.howtogeek.com/howto/windows-vista/using-symlinks-in-windows-vista/)?

To link a `lib/` directory in your project root into `node_modules`, do:

```
ln -s ../lib node_modules/app
```

and now from anywhere in your project you'll be able to require files in `lib/`
by doing `require('app/foo.js')` to get `lib/foo.js`.

### node_modules

People sometimes object to putting application-specific modules into
node_modules because it is not obvious how to check in your internal modules
without also checking in third-party modules from npm.

The answer is quite simple! If you have a `.gitignore` file that ignores
`node_modules`:

```
node_modules
```

You can just add an exception with `!` for each of your internal application
modules:

```
node_modules/*
!node_modules/foo
!node_modules/bar
```

Please note that you can't *unignore* a subdirectory,
if the parent is already ignored. So instead of ignoring `node_modules`,
you have to ignore every directory *inside* `node_modules` with the 
`node_modules/*` trick, and then you can add your exceptions.

Now anywhere in your application you will be able to `require('foo')` or
`require('bar')` without having a very large and fragile relative path.

If you have a lot of modules and want to keep them more separate from the
third-party modules installed by npm, you can just put them all under a
directory in `node_modules` such as `node_modules/app`:

```
node_modules/app/foo
node_modules/app/bar
```

Now you will be able to `require('app/foo')` or `require('app/bar')` from
anywhere in your application.

In your `.gitignore`, just add an exception for `node_modules/app`:

```
node_modules/*
!node_modules/app
```

If your application had transforms configured in package.json, you'll need to
create a separate package.json with its own transform field in your
`node_modules/foo` or `node_modules/app/foo` component directory because
transforms don't apply across module boundaries. This will make your modules
more robust against configuration changes in your application and it will be
easier to independently reuse the packages outside of your application.

### custom paths

You might see some places talk about using the `$NODE_PATH` environment variable
or `opts.paths` to add directories for node and browserify to look in to find
modules.

Unlike most other platforms, using a shell-style array of path directories with
`$NODE_PATH` is not as favorable in node compared to making effective use of the
`node_modules` directory.

This is because your application is more tightly coupled to a runtime
environment configuration so there are more moving parts and your application
will only work when your environment is setup correctly.

node and browserify both support but discourage the use of `$NODE_PATH`.

## non-javascript assets

There are many
[browserify transforms](https://github.com/substack/node-browserify/wiki/list-of-transforms)
you can use to do many things. Commonly, transforms are used to include
non-javascript assets into bundle files.

### brfs

One way of including any kind of asset that works in both node and the browser
is brfs.

brfs uses static analysis to compile the results of `fs.readFile()` and
`fs.readFileSync()` calls down to source contents at compile time.

For example, this `main.js`:

``` js
var fs = require('fs');
var html = fs.readFileSync(__dirname + '/robot.html', 'utf8');
console.log(html);
```

applied through brfs would become something like:

``` js
var fs = require('fs');
var html = "<b>beep boop</b>";
console.log(html);
```

when run through brfs.

This is handy because you can reuse the exact same code in node and the browser,
which makes sharing modules and testing much simpler.

`fs.readFile()` and `fs.readFileSync()` accept the same arguments as in node,
which makes including inline image assets as base64-encoded strings very easy:

``` js
var fs = require('fs');
var imdata = fs.readFileSync(__dirname + '/image.png', 'base64');
var img = document.createElement('img');
img.setAttribute('src', 'data:image/png;base64,' + imdata);
document.body.appendChild(img);
```

If you have some css you want to inline into your bundle, you can do that too
with the assistence of a module such as
[insert-css](https://npmjs.org/package/insert-css):

``` js
var fs = require('fs');
var insertStyle = require('insert-css');

var css = fs.readFileSync(__dirname + '/style.css', 'utf8');
insertStyle(css);
```

Inserting css this way works fine for small reusable modules that you distribute
with npm because they are fully-contained, but if you want a more wholistic
approach to asset management using browserify, check out 
[atomify](https://www.npmjs.org/package/atomify) and
[parcelify](https://www.npmjs.org/package/parcelify).

### hbsify

### jadeify

### reactify

## reusable components

Putting these ideas about code organization together, we can build a reusable UI
component that we can reuse across our application or in other applications.

Here is a bare-bones example of an empty widget module:

``` js
module.exports = Widget;

function Widget (opts) {
    if (!(this instanceof Widget)) return new Widget(opts);
    this.element = document.createElement('div');
}

Widget.prototype.appendTo = function (target) {
    if (typeof target === 'string') target = document.querySelector(target);
    target.appendChild(this.element);
};
```

Handy javascript constructor tip: you can include a `this instanceof Widget`
check like above to let people consume your module with `new Widget` or
`Widget()`. It's nice because it hides an implementation detail from your API
and you still get the performance benefits and indentation wins of using
prototypes.

To use this widget, just use `require()` to load the widget file, instantiate
it, and then call `.appendTo()` with a css selector string or a dom element.

Like this:

``` js
var Widget = require('./widget.js');
var w = Widget();
w.appendTo('#container');
```

and now your widget will be appended to the DOM.

Creating HTML elements procedurally is fine for very simple content but gets
very verbose and unclear for anything bigger. Luckily there are many transforms
available to ease importing HTML into your javascript modules.

Let's extend our widget example using [brfs](https://npmjs.org/package/brfs). We
can also use [domify](https://npmjs.org/package/domify) to turn the string that
`fs.readFileSync()` returns into an html dom element:

``` js
var fs = require('fs');
var domify = require('domify');

var html = fs.readFileSync(__dirname + '/widget.html', 'utf8');

module.exports = Widget;

function Widget (opts) {
    if (!(this instanceof Widget)) return new Widget(opts);
    this.element = domify(html);
}

Widget.prototype.appendTo = function (target) {
    if (typeof target === 'string') target = document.querySelector(target);
    target.appendChild(this.element);
};
```

and now our widget will load a `widget.html`, so let's make one:

``` html
<div class="widget">
  <h1 class="name"></h1>
  <div class="msg"></div>
</div>
```

It's often useful to emit events. Here's how we can emit events using the
built-in `events` module and the [inherits](https://npmjs.org/package/inherits)
module:

``` js
var fs = require('fs');
var domify = require('domify');
var inherits = require('inherits');
var EventEmitter = require('events').EventEmitter;

var html = fs.readFileSync(__dirname + '/widget.html', 'utf8');

inherits(Widget, EventEmitter);
module.exports = Widget;

function Widget (opts) {
    if (!(this instanceof Widget)) return new Widget(opts);
    this.element = domify(html);
}

Widget.prototype.appendTo = function (target) {
    if (typeof target === 'string') target = document.querySelector(target);
    target.appendChild(this.element);
    this.emit('append', target);
};
```

Now we can listen for `'append'` events on our widget instance:

``` js
var Widget = require('./widget.js');
var w = Widget();
w.on('append', function (target) {
    console.log('appended to: ' + target.outerHTML);
});
w.appendTo('#container');
```

We can add more methods to our widget to set elements on the html:

``` js
var fs = require('fs');
var domify = require('domify');
var inherits = require('inherits');
var EventEmitter = require('events').EventEmitter;

var html = fs.readFileSync(__dirname + '/widget.html', 'utf8');

inherits(Widget, EventEmitter);
module.exports = Widget;

function Widget (opts) {
    if (!(this instanceof Widget)) return new Widget(opts);
    this.element = domify(html);
}

Widget.prototype.appendTo = function (target) {
    if (typeof target === 'string') target = document.querySelector(target);
    target.appendChild(this.element);
};

Widget.prototype.setName = function (name) {
    this.element.querySelector('.name').textContent = name;
}

Widget.prototype.setMessage = function (msg) {
    this.element.querySelector('.msg').textContent = msg;
}
```

If setting element attributes and content gets too verbose, check out
[hyperglue](https://npmjs.org/package/hyperglue).

Now finally, we can toss our `widget.js` and `widget.html` into
`node_modules/app-widget`. Since our widget uses the
[brfs](https://npmjs.org/package/brfs) transform, we can create a `package.json`
with:

``` json
{
  "name": "app-widget",
  "version": "1.0.0",
  "private": true,
  "main": "widget.js",
  "browserify": {
    "transform": [ "brfs" ]
  },
  "dependencies": {
    "brfs": "^1.1.1",
    "inherits": "^2.0.1"
  }
}
```

And now whenever we `require('app-widget')` from anywhere in our application,
brfs will be applied to our `widget.js` automatically!
Our widget can even maintain its own dependencies. This way we can update
dependencies in one widgets without worrying about breaking changes cascading
over into other widgets.

Make sure to add an exclusion in your `.gitignore` for
`node_modules/app-widget`:

```
node_modules/*
!node_modules/app-widget
```

You can read more about [shared rendering in node and the
browser](http://substack.net/shared_rendering_in_node_and_the_browser) if you
want to learn about sharing rendering logic between node and the browser using
browserify and some streaming html libraries.

# testing in node and the browser

Testing modular code is very easy! One of the biggest benefits of modularity is
that your interfaces become much easier to instantiate in isolation and so it's
easy to make automated tests.

Unfortunately, few testing libraries play nicely out of the box with modules and
tend to roll their own idiosyncratic interfaces with implicit globals and obtuse
flow control that get in the way of a clean design with good separation.

People also make a huge fuss about "mocking" but it's usually not necessary if
you design your modules with testing in mind. Keeping IO separate from your
algorithms, carefully restricting the scope of your module, and accepting
callback parameters for different interfaces can all make your code much easier
to test.

For example, if you have a library that does both IO and speaks a protocol,
[consider separating the IO layer from the
protocol](https://www.youtube.com/watch?v=g5ewQEuXjsQ#t=12m30)
using an interface like [streams](https://github.com/substack/stream-handbook).

Your code will be easier to test and reusable in different contexts that you
didn't initially envision. This is a recurring theme of testing: if your code is
hard to test, it is probably not modular enough or contains the wrong balance of
abstractions. Testing should not be an afterthought, it should inform your
whole design and it will help you to write better interfaces.

## testing libraries

### [tape](https://npmjs.org/package/tape)

Tape was specifically designed from the start to work well in both node and
browserify. Suppose we have an `index.js` with an async interface:

``` js
module.exports = function (x, cb) {
    setTimeout(function () {
        cb(x * 100);
    }, 1000);
};
```

Here's how we can test this module using [tape](https://npmjs.org/package/tape). 
Let's put this file in `test/beep.js`:

``` js
var test = require('tape');
var hundreder = require('../');

test('beep', function (t) {
    t.plan(1);
    
    hundreder(5, function (n) {
        t.equal(n, 500, '5*100 === 500');
    });
});
```

Because the test file lives in `test/`, we can require the `index.js` in the
parent directory by doing `require('../')`. `index.js` is the default place that
node and browserify look for a module if there is no package.json in that
directory with a `main` field.

We can `require()` tape like any other library after it has been installed with
`npm install tape`.

The string `'beep'` is an optional name for the test.
The 3rd argument to `t.equal()` is a completely optional description.

The `t.plan(1)` says that we expect 1 assertion. If there are not enough
assertions or too many, the test will fail. An assertion is a comparison
like `t.equal()`. tape has assertion primitives for:

* t.equal(a, b) - compare a and b strictly with `===`
* t.deepEqual(a, b) - compare a and b recursively
* t.ok(x) - fail if `x` is not truthy

and more! You can always add an additional description argument.

Running our module is very simple! To run the module in node, just run
`node test/beep.js`:

```
$ node test/beep.js
TAP version 13
# beep
ok 1 5*100 === 500

1..1
# tests 1
# pass  1

# ok
```

The output is printed to stdout and the exit code is 0.

To run our code in the browser, just do:

```
$ browserify test/beep.js > bundle.js
```

then plop `bundle.js` into a `<script>` tag:

```
<script src="bundle.js"></script>
```

and load that html in a browser. The output will be in the debug console which
you can open with F12, ctrl-shift-j, or ctrl-shift-k depending on the browser.

This is a bit cumbersome to run our tests in a browser, but you can install the
`testling` command to help. First do:

```
npm install -g testling
```

And now just do `browserify test/beep.js | testling`:

```
$ browserify test/beep.js | testling

TAP version 13
# beep
ok 1 5*100 === 500

1..1
# tests 1
# pass  1

# ok
```

`testling` will launch a real browser headlessly on your system to run the tests.

Now suppose we want to add another file, `test/boop.js`:

``` js
var test = require('tape');
var hundreder = require('../');

test('fraction', function (t) {
    t.plan(1);

    hundreder(1/20, function (n) {
        t.equal(n, 5, '1/20th of 100');
    });
});

test('negative', function (t) {
    t.plan(1);

    hundreder(-3, function (n) {
        t.equal(n, -300, 'negative number');
    });
});
```

Here our test has 2 `test()` blocks. The second test block won't start to
execute until the first is completely finished, even though it is asynchronous.
You can even nest test blocks by using `t.test()`.

We can run `test/boop.js` with node directly as with `test/beep.js`, but if we
want to run both tests, there is a minimal command-runner we can use that comes
with tape. To get the `tape` command do:

```
npm install -g tape
```

and now you can run:

```
$ tape test/*.js
TAP version 13
# beep
ok 1 5*100 === 500
# fraction
ok 2 1/20th of 100
# negative
ok 3 negative number

1..3
# tests 3
# pass  3

# ok
```

and you can just pass `test/*.js` to browserify to run your tests in the
browser:

```
$ browserify test/* | testling

TAP version 13
# beep
ok 1 5*100 === 500
# fraction
ok 2 1/20th of 100
# negative
ok 3 negative number

1..3
# tests 3
# pass  3

# ok
```

Putting together all these steps, we can configure `package.json` with a test
script:

``` json
{
  "name": "hundreder",
  "version": "1.0.0",
  "main": "index.js",
  "devDependencies": {
    "tape": "^2.13.1",
    "testling": "^1.6.1"
  },
  "scripts": {
    "test": "tape test/*.js",
    "test-browser": "browserify test/*.js | testlingify"
  }
}
```

Now you can do `npm test` to run the tests in node and `npm run test-browser` to
run the tests in the browser. You don't need to worry about installing commands
with `-g` when you use `npm run`: npm automatically sets up the `$PATH` for all
packages installed locally to the project.

If you have some tests that only run in node and some tests that only run in the
browser, you could have subdirectories in `test/` such as `test/server` and
`test/browser` with the tests that run both places just in `test/`. Then you
could just add the relevant directory to the globs:

``` json
{
  "name": "hundreder",
  "version": "1.0.0",
  "main": "index.js",
  "devDependencies": {
    "tape": "^2.13.1",
    "testling": "^1.6.1"
  },
  "scripts": {
    "test": "tape test/*.js test/server/*.js",
    "test-browser": "browserify test/*.js test/browser/*.js | testling"
  }
}
```

and now server-specific and browser-specific tests will be run in addition to
the common tests.

If you want something even slicker, check out
[prova](https://www.npmjs.org/package/prova) once you have gotten the basic
concepts.

### assert

The core assert module is a fine way to write simple tests too, although it can
sometimes be tricky to ensure that the correct number of callbacks have fired.

You can solve that problem with tools like
[macgyver](https://www.npmjs.org/package/macgyver) but it is appropriately DIY.

## code coverage

### coverify

A simple way to check code coverage in browserify is to use the
[coverify](https://npmjs.org/package/coverify) transform.

```
$ browserify -t coverify test/*.js | node | coverify
```

or to run your tests in a real browser:

```
$ browserify -t coverify test/*.js | testling | coverify
```

coverify works by transforming the source of each package so that each
expression is wrapped in a `__coverageWrap()` function.

Each expression in the program gets a unique ID and the `__coverageWrap()`
function will print `COVERED $FILE $ID` the first time the expression is
executed.

Before the expressions run, coverify prints a `COVERAGE $FILE $NODES` message to
log the expression nodes across the entire file as character ranges.

Here's what the output of a full run looks like:

```
$ browserify -t coverify test/whatever.js | node
COVERAGE "/home/substack/projects/defined/test/whatever.js" [[14,28],[14,28],[0,29],[41,56],[41,56],[30,57],[95,104],[95,105],[126,146],[126,146],[115,147],[160,194],[160,194],[152,195],[200,217],[200,218],[76,220],[59,221],[59,222]]
COVERED "/home/substack/projects/defined/test/whatever.js" 2
COVERED "/home/substack/projects/defined/test/whatever.js" 1
COVERED "/home/substack/projects/defined/test/whatever.js" 0
COVERAGE "/home/substack/projects/defined/index.js" [[48,49],[55,71],[51,71],[73,76],[92,104],[92,118],[127,139],[120,140],[172,195],[172,196],[0,204],[0,205]]
COVERED "/home/substack/projects/defined/index.js" 11
COVERED "/home/substack/projects/defined/index.js" 10
COVERED "/home/substack/projects/defined/test/whatever.js" 5
COVERED "/home/substack/projects/defined/test/whatever.js" 4
COVERED "/home/substack/projects/defined/test/whatever.js" 3
COVERED "/home/substack/projects/defined/test/whatever.js" 18
COVERED "/home/substack/projects/defined/test/whatever.js" 17
COVERED "/home/substack/projects/defined/test/whatever.js" 16
TAP version 13
# whatever
COVERED "/home/substack/projects/defined/test/whatever.js" 7
COVERED "/home/substack/projects/defined/test/whatever.js" 6
COVERED "/home/substack/projects/defined/test/whatever.js" 10
COVERED "/home/substack/projects/defined/test/whatever.js" 9
COVERED "/home/substack/projects/defined/test/whatever.js" 8
COVERED "/home/substack/projects/defined/test/whatever.js" 13
COVERED "/home/substack/projects/defined/test/whatever.js" 12
COVERED "/home/substack/projects/defined/test/whatever.js" 11
COVERED "/home/substack/projects/defined/index.js" 0
COVERED "/home/substack/projects/defined/index.js" 2
COVERED "/home/substack/projects/defined/index.js" 1
COVERED "/home/substack/projects/defined/index.js" 5
COVERED "/home/substack/projects/defined/index.js" 4
COVERED "/home/substack/projects/defined/index.js" 3
COVERED "/home/substack/projects/defined/index.js" 7
COVERED "/home/substack/projects/defined/index.js" 6
COVERED "/home/substack/projects/defined/test/whatever.js" 15
COVERED "/home/substack/projects/defined/test/whatever.js" 14
ok 1 should be equal

1..1
# tests 1
# pass  1

# ok
```

These COVERED and COVERAGE statements are just printed on stdout and they can be
fed into the `coverify` command to generate prettier output:

```
$ browserify -t coverify test/whatever.js | node | coverify
TAP version 13
# whatever
ok 1 should be equal

1..1
# tests 1
# pass  1

# ok

# /home/substack/projects/defined/index.js: line 6, column 9-32

          console.log('whatever');
          ^^^^^^^^^^^^^^^^^^^^^^^^

# coverage: 30/31 (96.77 %)
```

To include code coverage into your project, you can add an entry into the
`package.json` scripts field:

``` json
{
  "scripts": {
    "test": "tape test/*.js",
    "coverage": "browserify -t coverify test/*.js | node | coverify"
  }
}
```

There is also a [covert](https://npmjs.com/package/covert) package that
simplifies the browserify and coverify setup:

``` json
{
  "scripts": {
    "test": "tape test/*.js",
    "coverage": "covert test/*.js"
  }
}
```

To install coverify or covert as a devDependency, run
`npm install -D coverify` or `npm install -D covert`.

# bundling

这部分内容包含了更详细的打包内容。

打包是一个从入口文件(entry files), 遍历整个依赖图, 将所有的源文件整合至单个输出文件的过程.

## saving bytes

你最想修改的第一件事是通过npm安装到硬盘上的文件是如何避免重复的.

当你从一个空目录运行npm install, npm通常会找出两个module所共有的依赖, 并安装到最顶级的文件夹. 
然而当你再次安装更多的包的时候, npm不会自动进行查找公共依赖操作. 你可以运行 `npm dedupe` 来
手动进行这个操作. 你也可以删掉整个 `node_modules/` 目录, 然后重新安装, 如果依赖重复的问题还存在的话. 

browserify不会多次包含同一个文件, 但是相互兼容的多个版本的清况有些不同. browserify并不能感知到版本, 它会
包含 `node_modules/` 文件夹中的包, 对应着某个版本, 根据node使用的 `require()` 算法一致. 

你可以运行 `browserify --list` 和 `browserify --deps` 命令来查看哪些文件将会被包含进来, 来检查文件重复. 

## standalone

你可以使用 `--standalone` 选项来生成UMD模块, 可以在node、浏览器全局变量, 以及AMD环境中工作.

只需要向打包命令中添加 `--standalone NAME`:
```
$ browserify foo.js --standalone xyz > bundle.js
```

这条命令会将 `foo.js` 的内容导出在一个名为 `xyz` 的外部模块中. 如果在运行环境中检
测到了一个模块加载系统, 就会使用这个模块系统. 否则将会绑定在 `window.xyz` 全局变量中.

你可以使用点号语法(dot-syntax)来指定一个命名空间:

```
$ browserify foo.js --standalone foo.bar.baz > bundle.js
```

如果在运行环境中已经存在 `foo` 或者 `foo.bar` 全局变量, browserify将会把这个模块的
导出值绑定在那些全局变量上. 

注意独立模块只在单入口或者是直接required的文件的情况下使用


## external bundles

## ignoring and excluding

对于browserify来说, 忽略(ignore) 意味着: 用一个空的对象代替那个模块定义. 排除(exclude) 
意味着将一个模块从依赖图中完全移除.

另外一种方法是使用 package.json 的 browser字段来实现ignore和exclude功能, 在本文中其他地方介绍了.

### ignoring

忽略是一种为一些只在特定代码中工作的node模块提供空的定义而设计的积极策略. 例如, 
一个模块依赖一个库, 这个库中一些代码只在node中能正常运行:

``` js
var fs = require('fs');
var path = require('path');
var mkdirp = require('mkdirp');

exports.convert = convert;
function convert (src) {
    return src.replace(/beep/g, 'boop');
}

exports.write = function (src, dst, cb) {
    fs.readFile(src, function (err, src) {
        if (err) return cb(err);
        mkdirp(path.dirname(dst), function (err) {
            if (err) return cb(err);
            var out = convert(src);
            fs.writeFile(dst, out, cb);
        });
    });
};
```

browserify 已经忽略了 `fs` 模块, 通过返回一个空的object, 但是上边代码中的 `.write()` 函数
在浏览器中如果没有进一步的操作是不能正常工作的, 进一步操作例如静态分析转换代码或者提供一个
fs模块抽象.

然而, 如果我们真的需要 `convert()` 函数, 但是不希望 `mkdirp` 这个模块出现在最后打包好的文件中,
我们可以忽略mkdirp这个模块, 通过 `b.ignore('mkdirp')` 或者 `browserify --ignore mkdirp`. 这份
代码在浏览器中依然可以正差工作, 如果我们不取调用 `write()` 函数的话. 因为 `require('mkdirp')` 不会
抛出异常, 只是返回值是一个空的object.

通常来说一些让算法模块(类似parsers, formatters)自己进行IO操作不是一个好主意. 但是
这个trick可以让你在浏览器中使用那些模块.

在命令行工具中忽略 `foo`:

```
browserify --ignore foo
```

在api中忽略 `foo`, 同时有一个bundle实例 `b`:

``` js
b.ignore('foo')
```

### excluding

另一种我们希望的相关情况是从输出中完全移除某一个模块, 然后 `require('modulename')`在
运行时会失败. 这在我们项将代码打包成多个输出, 并形成级连至前面定义的 `require()` 定义中.

例如, 如果我们有一个独立的 `jquery` 打包文件, 我们并不想它出现在主打包输出中.

```
$ npm install jquery
$ browserify -r jquery --standalone jquery > jquery-bundle.js
```

然后我们只是在 `main.js` 中 `require('jquery')`:

``` js
var $ = require('jquery');
$(window).click(function () { document.body.bgColor = 'red' });
```

延迟到前面的jquery打包, 我们可以这样写:

``` html
<script src="jquery-bundle.js"></script>
<script src="bundle.js"></script>
```

已及在 `bundle.js` 中没有jquery的定义, 然后在编译 `main.js` 时, 可以 `--exclude jquery`


```
browserify main.js --exclude jquery > bundle.js
```

通过命令行程序来排除 `foo`:

```
browserify --exclude foo
```

通过api来排除 `foo`:

``` js
b.exclude('foo')
```

## browserify cdn

# shimming

不幸的是, 某些包不是使用node-style的commonjs规范的. 对于那些使用全局变量或者AMD方式运作的
模块, 这里有些包可以自动的帮助你将那些麻烦的包转换为browserify可以理解的包.

## browserify-shim

一种可以自动转换非commonjs包的方式, 通过
[browserify-shim](https://npmjs.org/package/browserify-shim).

[browserify-shim](https://npmjs.org/package/browserify-shim) 是作为一个转换器(transform)加载的.
他可以读取 `package.json` 中的 `browserify-shim`字段.

假设我们需要转换放在 `./vendor/foo.js` 的第三方库, 这个库通过window全部变量 `FOO` 暴露它的功能. 
我们可以这样设置我们的 `package.json`: 

``` json
{
  "browserify": {
    "transform": "browserify-shim"
  },
  "browserify-shim": {
    "./vendor/foo.js": "FOO"
  }
}
```

当我们 `require('./vendor/foo.js')`, 我们可以得到 `./vendor/foo.js` 试图在全局作用域设置的 `FOO` 变量.
但是那个视图被隔离到了一个单独的上下文中, 以防止全局变量污染.

我们甚至可以使用 [browser字段](#browser-field) 来使 `require('foo')` 正常工作, 而不必
每次都是使用一个相对路径 `./vendor/foo.js` 去加载.

``` json
{
  "browser": {
    "foo": "./vendor/foo.js"
  },
  "browserify": {
    "transform": "browserify-shim"
  },
  "browserify-shim": {
    "foo": "FOO"
  }
}
```

现在 `require('foo')` 将会返回 `./vendor/foo.js` 试图设置在全局作用域的 `FOO` 变量.

# partitioning

大部分情况下, 默认的将一个或多个入口文件打包成单个输出是非常合适的, 特别是考虑到 
将等待时间减少至一个http请求就获取到了所有的javascript资源.

然而, 某些情况下这种打包方式不适合网站的某些大部分访问者很少或者从不访问的部分. 例如
管理员工作台. 这种分离策略可以使用[忽略和排除](#ignoring-and-excluding)所提到的技术完成.
但是为一个庞大且不确定的依赖图手动的分析剥离出共同的依赖是相当乏味的工作.

幸运的是, 这里有一些插件可以自动的分析browserify的输出至单独的文件中.

## factor-bundle

[factor-bundle](https://www.npmjs.org/package/factor-bundle) 根据入口文件分割browserify输出至
多个打包文件中. 对于每个入口, 一个入口相关的输出将会被创建. 被两个或多个入口所依赖的文件被
剥离至一个公有的模块中. 

例如, 假设我们有两个网页: /x 和 /y. 对于每个网页都有一个入口, /x 的入口是 `x.js`, 
/y 的入口是 `y.js`.

然后我们生成页面特定的打包文件 `bundle/x.js` 和 `bundle/y.js` 以及 `x.js` 和 `y.js`
所共有的依赖组成的 `bundle/common.js`:

```
browserify x.js y.js -p [ factor-bundle -o bundle/x.js -o bundle/y.js ] \
  -o bundle/common.js
```

现在我们可以简单的在每个页面上防止两个script标签. 在 /x 上我们可以放置:

``` html
<script src="/bundle/common.js"></script>
<script src="/bundle/x.js"></script>
```

在 /y 上可以放置:

``` html
<script src="/bundle/common.js"></script>
<script src="/bundle/y.js"></script>
```

你也可以加载通过ajax或者动态插入一个script标签来异步地加载打包好的文件. 但是 factor-nbundle
插件只关心如何生成这些打包文件, 并不是如何加载他们.

## partition-bundle

[partition-bundle](https://www.npmjs.org/package/partition-bundle) 也是分割输出至多个打包文件
的. 同 factor-bundle 一致, 但是包含了一个内置的加载器 `loadjs()` 函数.

partition-bundle 接受一个包含输入文件于打包输出之间的映射关系的json文件:

```
{
  "entry.js": ["./a"],
  "common.js": ["./b"],
  "common/extra.js": ["./e", "./d"]
}
```

然后 partition-bundle 被加载为插件, 映射文件, 输出目录以及目标url(做动态加载时必须)被
传入:

```
browserify -p [ partition-bundle --map mapping.json \
  --output output/directory --url directory ]
```

现在你可以添加:

``` html
<script src="entry.js"></script>
```

到你的页面去加载整个文件. 在入口文件内部, 你可以用 `loadjs()` 函数去动态的加载其他
打包文件:

``` js
a.addEventListener('click', function() {
  loadjs(['./e', './d'], function(e, d) {
    console.log(e, d);
  });
});
```

# compiler pipeline

从版本5以来, browserify暴露出内部的编译流程, 作为一个 [labeled-stream-splicer](https://www.npmjs.org/package/labeled-stream-splicer) 实例.

这意味着转换操作可以直接从内部流程线添加或删除. 这个流程线为高级自定义提供了一个干净
的接口, 例如监视文件更改或者根据入口分析输出分割至多个打包文件.

例如, 我们可以将内建的以整数为基准的模块命名方案替换掉, 换成Hashed IDs的命名方案.
通过注入一个pass-through, 在deps已经计算好整个源代码的哈希值之后. 然后我们可以利用计算
好的哈希值创建我们自己的命名方案, 并将内建的命名方案替换掉:

``` js
var browserify = require('browserify');
var through = require('through2');
var shasum = require('shasum');

var b = browserify('./main.js');

var hashes = {};
var hasher = through.obj(function (row, enc, next) {
    hashes[row.id] = shasum(row.source);
    this.push(row);
    next();
});
b.pipeline.get('deps').push(hasher);

var labeler = through.obj(function (row, enc, next) {
    row.id = hashes[row.id];
    
    Object.keys(row.deps).forEach(function (key) {
        row.deps[key] = hashes[row.deps[key]];
    });
    
    this.push(row);
    next();
});
b.pipeline.get('label').splice(0, 1, labeler);

b.bundle().pipe(process.stdout);
```

现在, 我们得到了基于文本内容的哈希值命名方案, 而不是整数ID:

```
$ node bundle.js
(function e(t,n,r){function s(o,u){if(!n[o]){if(!t[o]){var a=typeof require=="function"&&require;if(!u&&a)return a(o,!0);if(i)return i(o,!0);var f=new Error("Cannot find module '"+o+"'");throw f.code="MODULE_NOT_FOUND",f}var l=n[o]={exports:{}};t[o][0].call(l.exports,function(e){var n=t[o][1][e];return s(n?n:e)},l,l.exports,e,t,n,r)}return n[o].exports}var i=typeof require=="function"&&require;for(var o=0;o<r.length;o++)s(r[o]);return s})({"5f0a0e3a143f2356582f58a70f385f4bde44f04b":[function(require,module,exports){
var foo = require('./foo.js');
var bar = require('./bar.js');

console.log(foo(3) + bar(4));

},{"./bar.js":"cba5983117ae1d6699d85fc4d54eb589d758f12b","./foo.js":"736100869ec2e44f7cfcf0dc6554b055e117c53c"}],"cba5983117ae1d6699d85fc4d54eb589d758f12b":[function(require,module,exports){
module.exports = function (n) { return n * 100 };

},{}],"736100869ec2e44f7cfcf0dc6554b055e117c53c":[function(require,module,exports){
module.exports = function (n) { return n + 1 };

},{}]},{},["5f0a0e3a143f2356582f58a70f385f4bde44f04b"]);
```

注意, 内建的命名方案做了一些其他的事情, 例如检查外部依赖(external), 排除文件(exclude),
所以如果你依赖这些特性的话, 将整个命名方案替换掉会很困难. 这个例子只是简单的演示你可以
修改内部的编译器流水线.

## build your own browserify

## labeled phases

browserify处理线的每个阶段都有一个名字, 你就可以进行插入代码修改. 通过 `.get(name)`
来让 [labeled-stream-splicer](https://npmjs.org/package/labeled-stream-splicer) 
返回一个正确的label. 一旦你有了这个阶段的引用之后, 你可以 `.push()`, `.pop()`, 
`.shift()`, `.unshift()`, 以及 `.splice()` 你自己的转换流至处理器流水线, 或者移除
已有的转换流.

### recorder

记录器(recorder)是用来捕捉发送到 `deps` 阶段的输入, 以至于他们可以在随后的 `.bundle()`
调用中重放. 不同于之前的释出版本, v5可以多次生成打包文件. 这对于像 watchify 这样每次
文件更改时去重新生成打包文件, 非常方便.

### deps

依赖处理(deps) 阶段希望输入入口文件以及 `require()` 文件或者对象, 然后调用
[module-deps](https://npmjs.org/package/module-deps) 来生成一个包含依赖图中
所有文件的json的流.

module-deps 模块被调用时, 有一些自定义选项例如:

* 设置browserify为package.json的转换字段
* 过滤出外部依赖(external), 排除的文件(excluded), 以及忽略的文件(ignored)
* 设置默认的后缀名为 `.js` 和 `.json` 加上browserify构造器中的 `opt.extensions` 选项
* 确定全局变量 [insert-module-globals](#insert-module-globals), 检查以及提供 `process`, `Buffer`, `global`, `__dirname`,
以及 `__filename` 的实现.
* 设置应该由browserify提供实现的node内置模块

### json

这个转换为 `.json` 文件在最前面添加 `module.exports = `

### unbom

这个转换移除了BOM(byte order markers)头. 用于windows上某些文本编辑器检测文件编码.
这些标记会被node忽略, 所以browserify为了兼容性也忽略他们.

### syntax

这个转换使用 [syntax-error](https://npmjs.org/package/syntax-error) 包来检查语法错误.
并提供了错误行数和列数等有用信息.

### sort

这个阶段使用 [deps-sort](https://www.npmjs.org/package/deps-sort) 来排序所有的
文件(在browserify stream表现为 row), 使最后的打包具有确定的顺序.

### dedupe

这个阶段的转换使用在 `sort` 阶段的 [deps-sort](https://www.npmjs.org/package/deps-sort)
提供的重复信息来去除某些包含重复内容的文件.

### label

这个阶段将可能泄露系统路径信息的基于文件的IDs转换为基于整形数字的IDs.

命名(`label`)阶段将会使用 `opts.basedir` 或者 `process.cwd()` 将文件路径都正常化.
以防止泄露系统路径信息.

### emit-deps

这个阶段将会对 `label` 阶段后的每一行(即每一个文件)引发一个 `'dep'` 事件,

### debug

如果 `opts.debug` 被传递到了 `browserify()` 构造器, 这个阶段将会转换输入, 以添加
`sourceRoot` 和 `sourceFile` 属性, 用于 `pack` 阶段的 
[browser-pack](https://npmjs.org/package/browser-pack).

### pack

这个阶段将带有 `id` 和 `source` 的文件行作为输入, 然后使用
[browser-pack](https://npmjs.org/package/browser-pack) 来生成包含所有依赖的打包文件.

### wrap

这是结束时一个空的阶段, 你可以在这个阶段轻松修改打包信息而不用修改已有的机制.

## browser-unpack

[browser-unpack](https://npmjs.org/package/browser-unpack) 会将一个编译过的打包文件
回转成一种类似 [module-deps](https://npmjs.org/package/module-deps) 的输出的格式.

这种方式在你想查看或者转换一个已经编译打包好的文件时非常方便.

例如:

``` js
$ browserify src/main.js | browser-unpack
[
{"id":1,"source":"module.exports = function (n) { return n * 100 };","deps":{}}
,
{"id":2,"source":"module.exports = function (n) { return n + 1 };","deps":{}}
,
{"id":3,"source":"var foo = require('./foo.js');\nvar bar = require('./bar.js');\n\nconsole.log(foo(3) + bar(4));","deps":{"./bar.js":1,"./foo.js":2},"entry":true}
]
```

这种分解方式对于类似 [factor-bundle](https://www.npmjs.org/package/factor-bundle)
和 [bundle-collapser](https://www.npmjs.org/package/bundle-collapser) 等工具来说是
必要的.

# plugins

插件在加载时, 可以访问browserify实例自身.

## using plugins

插件只应该用在那些转换器或者全局转换器不够强大去完成想要的功能时使用.

你可以使用 `-p` 选项在命令行工具中去加载一个插件:

```
$ browserify main.js -p foo > bundle.js
```

会去加载一个叫 `foo` 的插件. `foo` 将会被 `require()` 函数处理, 如果要加载一个本地文件
作为插件, 加上 `./` 前缀即可. 要从 `node_modules/foo` 加载插件, 只需 `-p foo`.

你可以为插件传递参数, 使用方括号来包裹整个语句, 包括第一个参数, 即是插件名.

```
$ browserify one.js two.js \
  -p [ factor-bundle -o bundle/one.js -o bundle/two.js ] \
  > common.js
```

这中命令行语法是使用 [subarg](https://npmjs.org/package/subarg) 解析的.

查看更多的browserify插件, 使用 `browserify-plugins` 搜索npm即可: 
http://npmjs.org/browse/keyword/browserify-plugin

## authoring plugins

要创建一个插件, 需要写一个导出单个函数的包, 该函数接受一个browserify bundle实例
参数, 和一个选项(options)作为第二个参数:

``` js
// example plugin

module.exports = function (b, opts) {
  // ...
}
```

插件可通过监听各种事件或者直接在编译流水线上增删转换器(transform)来直接操作browserify
bundle 实例. 插件不应该取重写覆盖该bundle实例上的方法, 除非你有一个非常好的理由.
