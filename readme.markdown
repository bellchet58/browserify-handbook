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

node同样包含一个搜寻路径数组的机智, 但是这种机智已经被废弃, 你应该使用 `node_modules` 除非
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

## auto-recompile

每次运行一条命令去重新打包会很慢, 以及乏味. 幸运的是有许多的工具可以解决这个问题, 其中某些支持
不同程度的热更新(live-reloading), 其他的则是传统的手动刷新机制.

这里有一堆可以使用的工具, 但是npm上有更多可用的工具. 
There are many different tools here that encompass many different tradeoffs and
development styles. It can be a little bit more work up-front to find the tools
that responate most strongly with your own personal expectations and experience,
but I think this diversity helps programmers to be more effective and provides
more room for creativity and experimentation. I think diversity in tooling and a
smaller browserify core is healthier in the medium to long term than picking a
few "winners" by including them in browserify core (which creates all kinds of
havoc in meaningful versioning and bitrot in core).

That said, here are a few modules you might want to consider for setting up a
browserify development workflow. But keep an eye out for other tools not (yet)
on this list!

### [watchify](https://npmjs.org/package/watchify)

You can use `watchify` interchangeably with `browserify` but instead of writing
to an output file once, watchify will write the bundle file and then watch all
of the files in your dependency graph for changes. When you modify a file, the
new bundle file will be written much more quickly than the first time because of
aggressive caching.

You can use `-v` to print a message every time a new bundle is written:

```
$ watchify browser.js -d -o static/bundle.js -v
610598 bytes written to static/bundle.js  0.23s
610606 bytes written to static/bundle.js  0.10s
610597 bytes written to static/bundle.js  0.14s
610606 bytes written to static/bundle.js  0.08s
610597 bytes written to static/bundle.js  0.08s
610597 bytes written to static/bundle.js  0.19s
```

Here is a handy configuration for using watchify and browserify with the
package.json "scripts" field:

``` json
{
  "build": "browserify browser.js -o static/bundle.js",
  "watch": "watchify browser.js -o static/bundle.js --debug --verbose",
}
```

To build the bundle for production do `npm run build` and to watch files for
during development do `npm run watch`.

[Learn more about `npm run`](http://substack.net/task_automation_with_npm_run).

### [beefy](https://www.npmjs.org/package/beefy)

If you would rather spin up a web server that automatically recompiles your code
when you modify it, check out [beefy](http://didact.us/beefy/).

Just give beefy an entry file:

```
beefy main.js
```

and it will set up shop on an http port.

### [wzrd](https://github.com/maxogden/wzrd)

In a similar spirit to beefy but in a more minimal form is
[wzrd](https://github.com/maxogden/wzrd).

Just `npm install -g wzrd` then you can do:

```
wzrd app.js
```

and open up http://localhost:9966 in your browser.

### browserify-middleware, enchilada

If you are using express, check out
[browserify-middleware](https://www.npmjs.org/package/browserify-middleware)
or [enchilada](https://www.npmjs.org/package/enchilada).

They both provide middleware you can drop into an express application for
serving browserify bundles.

### [livereactload](https://github.com/milankinen/livereactload)

livereactload is a tool for [react](https://github.com/facebook/react)
that automatically updates your web page state when you modify your code.

livereactload is just an ordinary browserify transform that you can load with
`-t livereactload`, but you should consult the
[project readme](https://github.com/milankinen/livereactload#livereactload)
for more information.

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

budo is a browserify development server with a stronger focus on incremental bundling and LiveReload integration (including CSS injection). 

Install it like so:

```sh
npm install budo -g
```

And run it on your entry file:

```
budo app.js
```

This starts the server at [http://localhost:9966](http://localhost:9966) with a default `index.html`, incrementally bundling your source on filesave. The requests are delayed until the bundle has finished, so you won't be served stale or empty bundles if you refresh the page mid-update.

To enable LiveReload and have the browser refresh on JS/HTML/CSS changes, you can run it like so:

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

If you use grunt, you'll probably want to use the
[grunt-browserify](https://www.npmjs.org/package/grunt-browserify) plugin.

## gulp

If you use gulp, you should use the browserify API directly.

Here is
[a guide for getting started](http://viget.com/extend/gulp-browserify-starter-faq)
with gulp and browserify.

Here is a guide on how to [make browserify builds fast with watchify using
gulp](https://github.com/gulpjs/gulp/blob/master/docs/recipes/fast-browserify-builds-with-watchify.md)
from the official gulp recipes.

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



In node all the file and network APIs deal with Buffer chunks. In browserify the
Buffer API is provided by , which
uses augmented typed arrays in a very performant way with fallbacks for old
browsers.

Here's an example of using `Buffer` to convert a base64 string to hex:

```
var buf = Buffer('YmVlcCBib29w', 'base64');
var hex = buf.toString('hex');
console.log(hex);
```

This example will print:

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

In node, `global` is the top-level scope where global variables are attached
similar to how `window` works in the browser. In browserify, `global` is just an
alias for the `window` object.

## [__filename](http://nodejs.org/docs/latest/api/all.html#all_filename)

`__filename` is the path to the current file, which is different for each file.

To prevent disclosing system path information, this path is rooted at the
`opts.basedir` that you pass to `browserify()`, which defaults to the
[current working directory](https://en.wikipedia.org/wiki/Current_working_directory).

If we have a `main.js`:

``` js
var bar = require('./foo/bar.js');

console.log('here in main.js, __filename is:', __filename);
bar();
```

and a `foo/bar.js`:

``` js
module.exports = function () {
    console.log('here in foo/bar.js, __filename is:', __filename);
};
```

then running browserify starting at `main.js` gives this output:

```
$ browserify main.js | node
here in main.js, __filename is: /main.js
here in foo/bar.js, __filename is: /foo/bar.js
```

## [__dirname](http://nodejs.org/docs/latest/api/all.html#all_dirname)

`__dirname` is the directory of the current file. Like `__filename`, `__dirname`
is rooted at the `opts.basedir`.

Here's an example of how `__dirname` works:

main.js:

``` js
require('./x/y/z/abc.js');
console.log('in main.js __dirname=' + __dirname);
```

x/y/z/abc.js:

``` js
console.log('in abc.js, __dirname=' + __dirname);
```

output:

```
$ browserify main.js | node
in abc.js, __dirname=/x/y/z
in main.js __dirname=/
```

# transforms

Instead of browserify baking in support for everything, it supports a flexible
transform system that are used to convert source files in-place.

This way you can `require()` files written in coffee script or templates and
everything will be compiled down to javascript.

To use [coffeescript](http://coffeescript.org/) for example, you can use the
[coffeeify](https://www.npmjs.org/package/coffeeify) transform.
Make sure you've installed coffeeify first with `npm install coffeeify` then do:

```
$ browserify -t coffeeify main.coffee > bundle.js
```

or with the API you can do:

```
var b = browserify('main.coffee');
b.transform('coffeeify');
```

The best part is, if you have source maps enabled with `--debug` or
`opts.debug`, the bundle.js will map exceptions back into the original coffee
script source files. This is very handy for debugging with firebug or chrome
inspector.

## writing your own

Transforms implement a simple streaming interface. Here is a transform that
replaces `$CWD` with the `process.cwd()`:

``` js
var through = require('through2');

module.exports = function (file) {
    return through(function (buf, enc, next) {
        this.push(buf.toString('utf8').replace(/\$CWD/g, process.cwd()));
        next();
    });
};
```

The transform function fires for every `file` in the current package and returns
a transform stream that performs the conversion. The stream is written to and by
browserify with the original file contents and browserify reads from the stream
to obtain the new contents.

Simply save your transform to a file or make a package and then add it with
`-t ./your_transform.js`.

For more information about how streams work, check out the
[stream handbook](https://github.com/substack/stream-handbook).

# package.json

## browser field

You can define a `"browser"` field in the package.json of any package that will
tell browserify to override lookups for the main field and for individual
modules.

If you have a module with a main entry point of `main.js` for node but have a
browser-specific entry point at `browser.js`, you can do:

``` json
{
  "name": "mypkg",
  "version": "1.2.3",
  "main": "main.js",
  "browser": "browser.js"
}
```

Now when somebody does `require('mypkg')` in node, they will get the exports
from `main.js`, but when they do `require('mypkg')` in a browser, they will get
the exports from `browser.js`.

Splitting up whether you are in the browser or not with a `"browser"` field in
this way is greatly preferrable to checking whether you are in a browser at
runtime because you may want to load different modules based on whether you are
in node or the browser. If the `require()` calls for both node and the browser
are in the same file, browserify's static analysis will include everything
whether you use those files or not.

You can do more with the "browser" field as an object instead of a string.

For example, if you only want to swap out a single file in `lib/` with a
browser-specific version, you could do:

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

or if you want to swap out a module used locally in the package, you can do:

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

You can ignore files (setting their contents to the empty object) by setting
their values in the browser field to `false`:

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

The browser field *only* applies to the current package. Any mappings you put
will not propagate down to its dependencies or up to its dependents. This
isolation is designed to protect modules from each other so that when you
require a module you won't need to worry about any system-wide effects it might
have. Likewise, you shouldn't need to wory about how your local configuration
might adversely affect modules far away deep into your dependency graph.

## browserify.transform field

You can configure transforms to be automatically applied when a module is loaded
in a package's `browserify.transform` field. For example, we can automatically
apply the [brfs](https://npmjs.org/package/brfs) transform with this
package.json:

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

Now in our `main.js` we can do:

``` js
var fs = require('fs');
var src = fs.readFileSync(__dirname + '/foo.txt', 'utf8');

module.exports = function (x) { return src.replace(x, 'zzz') };
```

and the `fs.readFileSync()` call will be inlined by brfs without consumers of
the module having to know. You can apply as many transforms as you like in the
transform array and they will be applied in order.

Like the `"browser"` field, transforms configured in package.json will only
apply to the local package for the same reasons.

### configuring transforms

Sometimes a transform takes configuration options on the command line. To apply these
from package.json you can do the following.

**on the command line**
```
browserify -t coffeeify \
           -t [ browserify-ngannotate --ext .coffee --bar ] \
           index.coffee > index.js
```

**in package.json**
``` json
"browserify": {
  "transform": [
    "coffeeify",
    ["browserify-ngannotate", {"ext": ".coffee", bar: true}]
  ]
}
```


# finding good modules

Here are [some useful heuristics](http://substack.net/finding_modules)
for finding good modules on npm that work in the browser:

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

Other metrics like number of stars on github, project activity, or a slick
landing page, are not as reliable.

## module philosophy

People used to think that exporting a bunch of handy utility-style things would
be the main way that programmers would consume code because that is the primary
way of exporting and importing code on most other platforms and indeed still
persists even on npm.

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

This section covers bundling in more detail.

Bundling is the step where starting from the entry files, all the source files
in the dependency graph are walked and packed into a single output file.

## saving bytes

One of the first things you'll want to tweak is how the files that npm installs
are placed on disk to avoid duplicates.

When you do a clean install in a directory, npm will ordinarily factor out
similar versions into the topmost directory where 2 modules share a dependency.
However, as you install more packages, new packages will not be factored out
automatically. You can however use the `npm dedupe` command to factor out
packages for an already-installed set of packages in `node_modules/`. You could
also remove `node_modules/` and install from scratch again if problems with
duplicates persist.

browserify will not include the same exact file twice, but compatible versions
may differ slightly. browserify is also not version-aware, it will include the
versions of packages exactly as they are laid out in `node_modules/` according
to the `require()` algorithm that node uses.

You can use the `browserify --list` and `browserify --deps` commands to further
inspect which files are being included to scan for duplicates.

## standalone

You can generate UMD bundles with `--standalone` that will work in node, the
browser with globals, and AMD environments.

Just add `--standalone NAME` to your bundle command:

```
$ browserify foo.js --standalone xyz > bundle.js
```

This command will export the contents of `foo.js` under the external module name
`xyz`. If a module system is detected in the host environment, it will be used.
Otherwise a window global named `xyz` will be exported.

You can use dot-syntax to specify a namespace hierarchy:

```
$ browserify foo.js --standalone foo.bar.baz > bundle.js
```

If there is already a `foo` or a `foo.bar` in the host environment in window
global mode, browserify will attach its exports onto those objects. The AMD and
`module.exports` modules will behave the same.

Note however that standalone only works with a single entry or directly-required
file.

## external bundles

## ignoring and excluding

In browserify parlance, "ignore" means: replace the definition of a module with
an empty object. "exclude" means: remove a module completely from a dependency graph.

Another way to achieve many of the same goals as ignore and exclude is the
"browser" field in package.json, which is covered elsewhere in this document.

### ignoring

Ignoring is an optimistic strategy designed to stub in an empty definition for
node-specific modules that are only used in some codepaths. For example, if a
module requires a library that only works in node but for a specific chunk of
the code:

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

browserify already "ignores" the `'fs'` module by returning an empty object, but
the `.write()` function here won't work in the browser without an extra step like
a static analysis transform or a runtime storage fs abstraction.

However, if we really want the `convert()` function but don't want to see
`mkdirp` in the final bundle, we can ignore mkdirp with `b.ignore('mkdirp')` or
`browserify --ignore mkdirp`. The code will still work in the browser if we
don't call `write()` because `require('mkdirp')` won't throw an exception, just
return an empty object.

Generally speaking it's not a good idea for modules that are primarily
algorithmic (parsers, formatters) to do IO themselves but these tricks can let
you use those modules in the browser anyway.

To ignore `foo` on the command-line do:

```
browserify --ignore foo
```

To ignore `foo` from the api with some bundle instance `b` do:

``` js
b.ignore('foo')
```

### excluding

Another related thing we might want is to completely remove a module from the
output so that `require('modulename')` will fail at runtime. This is useful if
we want to split things up into multiple bundles that will defer in a cascade to
previously-defined `require()` definitions.

For example, if we have a vendored standalone bundle for jquery that we don't want to appear in
the primary bundle:

```
$ npm install jquery
$ browserify -r jquery --standalone jquery > jquery-bundle.js
```

then we want to just `require('jquery')` in a `main.js`:

``` js
var $ = require('jquery');
$(window).click(function () { document.body.bgColor = 'red' });
```

defering to the jquery dist bundle so that we can write:

``` html
<script src="jquery-bundle.js"></script>
<script src="bundle.js"></script>
```

and not have the jquery definition show up in `bundle.js`, then while compiling
the `main.js`, you can `--exclude jquery`:

```
browserify main.js --exclude jquery > bundle.js
```

To exclude `foo` on the command-line do:

```
browserify --exclude foo
```

To exclude `foo` from the api with some bundle instance `b` do:

``` js
b.exclude('foo')
```

## browserify cdn

# shimming

Unfortunately, some packages are not written with node-style commonjs exports.
For modules that export their functionality with globals or AMD, there are
packages that can help automatically convert these troublesome packages into
something that browserify can understand.

## browserify-shim

One way to automatically convert non-commonjs packages is with
[browserify-shim](https://npmjs.org/package/browserify-shim).

[browserify-shim](https://npmjs.org/package/browserify-shim) is loaded as a
transform and also reads a `"browserify-shim"` field from `package.json`.

Suppose we need to use a troublesome third-party library we've placed in
`./vendor/foo.js` that exports its functionality as a window global called
`FOO`. We can set up our `package.json` with:

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

and now when we `require('./vendor/foo.js')`, we get the `FOO` variable that
`./vendor/foo.js` tried to put into the global scope, but that attempt was
shimmed away into an isolated context to prevent global pollution.

We could even use the [browser field](#browser-field) to make `require('foo')`
work instead of always needing to use a relative path to load `./vendor/foo.js`:

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

Now `require('foo')` will return the `FOO` export that `./vendor/foo.js` tried
to place on the global scope.

# partitioning

Most of the time, the default method of bundling where one or more entry files
map to a single bundled output file is perfectly adequate, particularly
considering that bundling minimizes latency down to a single http request to
fetch all the javascript assets.

However, sometimes this initial penalty is too high for parts of a website that
are rarely or never used by most visitors such as an admin panel.
This partitioning can be accomplished with the technique covered in the
[ignoring and excluding](#ignoring-and-excluding) section, but factoring out
shared dependencies manually can be tedious for a large and fluid dependency
graph.

Luckily, there are plugins that can automatically factor browserify output into
separate bundle payloads.

## factor-bundle

[factor-bundle](https://www.npmjs.org/package/factor-bundle) splits browserify
output into multiple bundle targets based on entry-point. For each entry-point,
an entry-specific output file is built. Files that are needed by two or more of
the entry files get factored out into a common bundle.

For example, suppose we have 2 pages: /x and /y. Each page has an entry point,
`x.js` for /x and `y.js` for /y.

We then generate page-specific bundles `bundle/x.js` and `bundle/y.js` with
`bundle/common.js` containing the dependencies shared by both `x.js` and `y.js`:

```
browserify x.js y.js -p [ factor-bundle -o bundle/x.js -o bundle/y.js ] \
  -o bundle/common.js
```

Now we can simply put 2 script tags on each page. On /x we would put:

``` html
<script src="/bundle/common.js"></script>
<script src="/bundle/x.js"></script>
```

and on page /y we would put:

``` html
<script src="/bundle/common.js"></script>
<script src="/bundle/y.js"></script>
```

You could also load the bundles asynchronously with ajax or by inserting a
script tag into the page dynamically but factor-bundle only concerns itself with
generating the bundles, not with loading them.

## partition-bundle

[partition-bundle](https://www.npmjs.org/package/partition-bundle) handles
splitting output into multiple bundles like factor-bundle, but includes a
built-in loader using a special `loadjs()` function.

partition-bundle takes a json file that maps source files to bundle files:

```
{
  "entry.js": ["./a"],
  "common.js": ["./b"],
  "common/extra.js": ["./e", "./d"]
}
```

Then partition-bundle is loaded as a plugin and the mapping file, output
directory, and destination url path (required for dynamic loading) are passed
in:

```
browserify -p [ partition-bundle --map mapping.json \
  --output output/directory --url directory ]
```

Now you can add:

``` html
<script src="entry.js"></script>
```

to your page to load the entry file. From inside the entry file, you can
dynamically load other bundles with a `loadjs()` function:

``` js
a.addEventListener('click', function() {
  loadjs(['./e', './d'], function(e, d) {
    console.log(e, d);
  });
});
```

# compiler pipeline

Since version 5, browserify exposes its compiler pipeline as a
[labeled-stream-splicer](https://www.npmjs.org/package/labeled-stream-splicer).

This means that transformations can be added or removed directly into the
internal pipeline. This pipeline provides a clean interface for advanced
customizations such as watching files or factoring bundles from multiple entry
points. 

For example, we could replace the built-in integer-based labeling mechanism with
hashed IDs by first injecting a pass-through transform after the "deps" have
been calculated to hash source files. Then we can use the hashes we captured to
create our own custom labeler, replacing the built-in "label" transform:

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

Now instead of getting integers for the IDs in the output format, we get file
hashes:

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

Note that the built-in labeler does other things like checking for the external,
excluded configurations so replacing it will be difficult if you depend on those
features. This example just serves as an example for the kinds of things you can
do by hacking into the compiler pipeline.

## build your own browserify

## labeled phases

Each phase in the browserify pipeline has a label that you can hook onto. Fetch
a label with `.get(name)` to return a
[labeled-stream-splicer](https://npmjs.org/package/labeled-stream-splicer)
handle at the appropriate label. Once you have a handle, you can `.push()`,
`.pop()`, `.shift()`, `.unshift()`, and `.splice()` your own transform streams
into the pipeline or remove existing transform streams.

### recorder

The recorder is used to capture the inputs sent to the `deps` phase so that they
can be replayed on subsequent calls to `.bundle()`. Unlike in previous releases,
v5 can generate bundle output multiple times. This is very handy for tools like
watchify that re-bundle when a file has changed.

### deps

The `deps` phase expects entry and `require()` files or objects as input and
calls [module-deps](https://npmjs.org/package/module-deps) to generate a stream
of json output for all of the files in the dependency graph.

module-deps is invoked with some customizations here such as:

* setting up the browserify transform key for package.json
* filtering out external, excluded, and ignored files
* setting the default extensions for `.js` and `.json` plus options configured
in the `opts.extensions` parameter in the browserify constructor
* configuring a global [insert-module-globals](#insert-module-globals)
transform to detect and implement `process`, `Buffer`, `global`, `__dirname`,
and `__filename`
* setting up the list of node builtins which are shimmed by browserify

### json

This transform adds `module.exports=` in front of files with a `.json`
extension.

### unbom

This transform removes byte order markers, which are sometimes used by windows
text editors to indicate the endianness of files. These markers are ignored by
node, so browserify ignores them for compatibility.

### syntax

This transform checks for syntax errors using the
[syntax-error](https://npmjs.org/package/syntax-error) package to give
informative syntax errors with line and column numbers.

### sort

This phase uses [deps-sort](https://www.npmjs.org/package/deps-sort) to sort
the rows written to it in order to make the bundles deterministic.

### dedupe

The transform at this phase uses dedupe information provided by
[deps-sort](https://www.npmjs.org/package/deps-sort) in the `sort` phase to
remove files that have duplicate contents. 

### label

This phase converts file-based IDs which might expose system path information
and inflate the bundle size into integer-based IDs.

The `label` phase will also normalize path names based on the `opts.basedir` or
`process.cwd()` to avoid exposing system path information.

### emit-deps

This phase emits a `'dep'` event for each row after the `label` phase.

### debug

If `opts.debug` was given to the `browserify()` constructor, this phase will
transform input to add `sourceRoot` and `sourceFile` properties which are used
by [browser-pack](https://npmjs.org/package/browser-pack) in the `pack` phase.

### pack

This phase converts rows with `'id'` and `'source'` parameters as input (among
others) and generates the concatenated javascript bundle as output
using [browser-pack](https://npmjs.org/package/browser-pack).

### wrap

This is an empty phase at the end where you can easily tack on custom post
transformations without interfering with existing mechanics.

## browser-unpack

[browser-unpack](https://npmjs.org/package/browser-unpack) converts a compiled
bundle file back into a format very similar to the output of
[module-deps](https://npmjs.org/package/module-deps).

This is very handy if you need to inspect or transform a bundle that has already
been compiled.

For example:

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

This decomposition is needed by tools such as
[factor-bundle](https://www.npmjs.org/package/factor-bundle)
and [bundle-collapser](https://www.npmjs.org/package/bundle-collapser).

# plugins

When loaded, plugins have access to the browserify instance itself.

## using plugins

Plugins should be used sparingly and only in cases where a transform or global
transform is not powerful enough to perform the desired functionality.

You can load a plugin with `-p` on the command-line:

```
$ browserify main.js -p foo > bundle.js
```

would load a plugin called `foo`. `foo` is resolved with `require()`, so to load
a local file as a plugin, preface the path with a `./` and to load a plugin from
`node_modules/foo`, just do `-p foo`.

You can pass options to plugins with square brackets around the entire plugin
expression, including the plugin name as the first argument:

```
$ browserify one.js two.js \
  -p [ factor-bundle -o bundle/one.js -o bundle/two.js ] \
  > common.js
```

This command-line syntax is parsed by the
[subarg](https://npmjs.org/package/subarg) package.

To see a list of browserify plugins, browse npm for packages with the keyword
"browserify-plugin": http://npmjs.org/browse/keyword/browserify-plugin

## authoring plugins

To author a plugin, write a package that exports a single function that will
receive a bundle instance and options object as arguments:

``` js
// example plugin

module.exports = function (b, opts) {
  // ...
}
```

Plugins operate on the bundle instance `b` directly by listening for events or
splicing transforms into the pipeline. Plugins should not overwrite bundle
methods unless they have a very good reason.
