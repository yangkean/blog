# Webpack 3，从入门到放弃

> **Update (2017.8.27)** : 关于 `output.publicPath`、`devServer.contentBase`、`devServer.publicPath`的区别。如下：
> * **output.publicPath**: 对于这个选项，我们无需关注什么绝对相对路径，因为两种路径都可以。我们只需要知道一点：这个选项是指定 HTML 文件中资源文件 (字体、图片、JS文件等) 的`文件名`的公共 URL 部分的。在实际情况中，我们首先会通过`output.filename`或有些 loader 如`file-loader`的`name`属性设置`文件名`的原始部分，webpack 将`文件名`的原始部分和公共部分结合之后，HTML 文件就能获取到资源文件了。
> * **devServer.contentBase**: 设置静态资源的根目录，`html-webpack-plugin`生成的 html 不是静态资源。当用 html 文件里的地址无法找到静态资源文件时就会去这个目录下去找。
> * **devServer.publicPath**: 指定浏览器上访问所有 **打包(bundled)文件** (在`dist`里生成的所有文件) 的根目录，这个根目录是相对服务器地址及端口的，比`devServer.contentBase`和`output.publicPath`优先。

## 前言

> **Tips**
> 如果你用过 webpack 且一直用的是 webpack 1，请参考 [**从v1迁移到v2**](https://webpack.js.org/guides/migrating/)  (v2 和 v3 差异不大) 对版本变更的内容进行适当的了解，然后再选择性地阅读本文。

首先，这篇文章是根据当前最新的 [webpack](https://github.com/webpack/webpack) 版本 (即 v3.4.1) 撰写，较长一段时间内无需担心过时的问题。其次，这应该会是一篇极长的文章，涵盖了基本的使用方法，有更高级功能的需求可以参考官方文档继续学习。再次，即使是基本的功能，也内容繁多，我尽可能地解释通俗易懂，将我学习过程中的疑惑和坑一一解释，如有纰漏，敬请雅正。再次，为了清晰有效地讲解，我会演示从零编写 demo，只要一步步跟着做，就会清晰许多。最后，官方文档也是个坑爹货！


## Webpack，何许人也？

借用官方的说法：

> webpack is a module bundler. Its main purpose is to bundle JavaScript files for usage in a browser, yet it is also capable of transforming, bundling, or packaging just about any resource or asset.

简言之，webpack 是一个模块打包器 (*module bundler*)，能够将任何资源如 JavaScript 文件、CSS 文件、图片等打包成一个或少数文件。

## 为什么要用介个 Webpack?

首先，定义已经说明了 webpack 能将多个资源模块打包成一个或少数文件，这意味着与以往的发起多个 HTTP 请求来获得资源相比，现在只需要发起少量的 HTTP 请求。

> **Tips** 
> 想了解合并 HTTP 请求的意义，请见 [**这里**](https://www.zhihu.com/question/34401250?from=profile_question_card)。

其次，webpack 能将你的资源转换为最适合浏览器的“格式”，提升应用性能。比如只引用被应用使用的资源 (剔除未被使用的代码)，懒加载资源 (只在需要的时候才加载相应的资源)。再次，对于开发阶段，webpack 也提供了实时加载和热加载的功能，大大地节省了开发时间。除此之外，还有许多优秀之处之处值得去挖掘。不过，webpack 最核心的还是打包的功能。

## webpack，gulp/grunt，npm，它们有什么区别?

webpack 是模块打包器（*module bundler*），把所有的模块打包成一个或少量文件，使你只需加载少量文件即可运行整个应用，而无需像之前那样加载大量的图片，css文件，js文件，字体文件等等。而gulp／grunt 是自动化构建工具，或者叫任务运行器（*task runner*），是把你所有重复的手动操作让代码来做，例如压缩JS代码、CSS代码，代码检查、代码编译等等，自动化构建工具并不能把所有模块打包到一起，也不能构建不同模块之间的依赖图。两者来比较的话，gulp/grunt 无法做模块打包的事，webpack 虽然有 loader 和 plugin可以做一部分 gulp／grunt 能做的事，但是终究 webpack 的插件还是不如 gulp／grunt 的插件丰富，能做的事比较有限。于是有人两者结合着用，将 webpack 放到 gulp／grunt 中用。然而，更好的方法是用 npm scripts 取代 gulp／grunt，npm 是 node 的包管理器 (*node package manager*)，用于管理 node 的第三方软件包，npm 对于任务命令的良好支持让你最终省却了编写任务代码的必要，取而代之的，是老祖宗的几个命令行，仅靠几句命令行就足以完成你的模块打包和自动化构建的所有需求。

## 准备开始

先来看看一个 webpack 的一个完备的配置文件，是 [介样](https://webpack.js.org/configuration/#options) 的，当然啦，这里面有很多配置项是即使到这个软件被废弃你也用不上的：），所以无需担心。

## 基本配置

开始之前，请确定你已经安装了当前 [Node](https://nodejs.org/en/) 的较新版本。

然后执行以下命令以新建我们的 demo 目录：

```bash
$ mkdir webpack-demo && cd webpack-demo && npm init -y
$ npm i --save-dev webpack
$ mkdir src && cd src && touch index.js
```

我们使用工具函数库 [lodash](https://github.com/lodash/lodash) 来演示我们的 demo。先安装之：

```bash
$ npm i --save lodash
```

**src/index.js**

```js
import _ from 'lodash';

function component() {
  const element = document.createElement('div');
    
  element.innerHTML = _.join(['Hello', 'webpack'], ' ');
    
  return element;
}

document.body.appendChild(component());
```

> **Tips**
> `import` 和 `export` 已经是 ES6 的标准，但是仍未得到大多数浏览器的支持 (可喜的是， Chrome 61 已经开始默认支持了，见 [**ES6 modules**](http://caniuse.com/#feat=es6-module))，不过 webpack 提供了对这个特性的支持，但是除了这个特性，其他的 ES6 特性并不会得到 webpack 的特别支持，如有需要，须借助 [**Babel**](https://babeljs.io/) 进行转译 (*transpile*)。

然后新建发布版本目录：

```bash
$ cd .. && mkdir dist && cd dist && touch index.html 
```

**dist/index.html**

```html
<!DOCTYPE html>
<html>
<head>
    <title>webpack demo</title>
</head>
<body>
    <script src="bundle.js"></script>
</body>
</html>
```

现在，我们运行 webpack 来打包 `index.js` 为 `bundle.js`，本地安装了 webpack 后可以通过 `node_modules/.bin/webpack` 来访问 webpack 的二进制版本。

```bash
$ cd ..
$ ./node_modules/.bin/webpack src/index.js dist/bundle.js # 第一个参数是打包的入口文件，第二个参数是打包的出口文件
```

咻咻咻，大致如下输出一波：

```bash
Hash: de8ed072e2c7b3892179
Version: webpack 3.4.1
Time: 390ms
    Asset    Size  Chunks                    Chunk Names
bundle.js  544 kB       0  [emitted]  [big]  main
   [0] ./src/index.js 225 bytes {0} [built]
   [2] (webpack)/buildin/global.js 509 bytes {0} [built]
   [3] (webpack)/buildin/module.js 517 bytes {0} [built]
    + 1 hidden module
```

现在，你已经得到了你的第一个打包文件 (bundle.js) 了。

## 使用配置文件

像上面这样使用 webpack 应该是最挫的姿势了，所以我们要使用 webpack 的配置文件来提高我们的姿势水平。

```bash
$ touch webpack.config.js
```

**webpack.config.js**

```js
const path = require('path');

module.exports = {
  entry: './src/index.js', // 入口起点，可以指定多个入口起点
  output: { // 输出，只可指定一个输出配置
    filename: 'bundle.js', // 输出文件名
    path: path.resolve(__dirname, 'dist') // 输出文件所在的目录
  }
};
```

执行：

```bash
$ ./node_modules/.bin/webpack --config webpack.config.js # `--config` 制定 webpack 的配置文件，默认是 `webpack.config.js`
```

所以这里可以省却 `--config webpack.config.js`。但是每次都要写 `./node_modules/.bin/webpack` 实在让人不爽，所以我们要动用 **NPM Scripts**。

**package.json**

```json
{
  ...
  "scripts": {
    "build": "webpack"
  },
  ...
}
```

> **Tips**
> 在 `npm scripts` 中我们可以通过包名直接引用本地安装的 npm 包的二进制版本，而无需编写包的整个路径。

执行：

```bash
$ npm run build
```

一波输出后便得到了打包文件。

> **Tips**
> `bulid` 并不是 `npm scripts` 的内置属性，需要使用 `npm run` 来执行脚本，详情见 [**npm run**](https://docs.npmjs.com/cli/run-script)。

## 打包其他类型的文件

因为其他文件和 JS 文件类型不同，要把他们加载到 JS 文件中就需要经过加载器 (*loader*) 的处理。

### 加载 CSS

我们需要安装两个 loader 来处理 CSS 文件：

```bash
$ npm i --save-dev style-loader css-loader
```

[style-loader](https://github.com/webpack-contrib/style-loader) 通过插入 \<style\> 标签将 CSS 加入到 DOM 中，[css-loader](https://github.com/webpack-contrib/css-loader) 会像解释 import/require() 一样解释 @import 和 url()。

```js
const path = require('path');

module.exports = {
  entry: './src/index.js',
  output: {
    filename: 'bundle.js', 
    path: path.resolve(__dirname, 'dist')
  },
  module: { // 如何处理项目中不同类型的模块
    rules: [ // 用于规定在不同模块被创建时如何处理模块的规则数组
      {
        test: /\.css$/, // 匹配特定文件的正则表达式或正则表达式数组
        use: [ // 应用于模块的 loader 使用列表
          'style-loader',
          'css-loader'
        ]
      }
    ]
  }
};
```

我们来创建一个 CSS 文件：

```bash
$ cd src && touch style.css
```

**src/style.css**

```css
.hello {
  color: red;
}
```

**src/index.js**

```js
import _ from 'lodash';
import './style.css'; // 通过`import`引入 CSS 文件

function component() {
  const element = document.createElement('div');
    
  element.innerHTML = _.join(['Hello', 'webpack'], ' ');
  element.classList.add('hello'); // 在相应元素上添加类名
    
  return element;
}

document.body.appendChild(component());
```

执行`npm run build`，然后打开`index.html`，就可以看到红色的字体了。CSS 文件此时已经被打包到 bundle.js 中。再打开浏览器控制台，就可以看到 webpack 做了些什么。

### 加载图片

```bash
$ npm install --save-dev file-loader
```

[file-loader](https://github.com/webpack-contrib/file-loader) 指示 webpack 以文件格式发出所需对象并返回文件的公共URL，可用于任何文件的加载。

**webpack.config.js**

```js
const path = require('path');

module.exports = {
  entry: './src/index.js',
  output: {
    filename: 'bundle.js', 
    path: path.resolve(__dirname, 'dist')
  },
  module: {
    rules: [
      {
        test: /\.css$/,
        use: [
          'style-loader',
          'css-loader'
        ]
      },
      { // 增加加载图片的规则
        test: /\.(png|svg|jpg|gif)$/,
        use: [
          'file-loader'
        ]
      }
    ]
  }
};
```

我们在当前项目的目录中如下增加图片：

```
  webpack-demo
  |- package.json
  |- webpack.config.js
  |- /dist
    |- bundle.js
    |- index.html
  |- /src
+   |- icon.jpg
    |- style.css
    |- index.js
  |- /node_modules
```

**src/index.js**

```js
import _ from 'lodash';
import './style.css';
import Icon from './icon.jpg'; // Icon 是图片的 URL

function component() {
  const element = document.createElement('div');
    
  element.innerHTML = _.join(['Hello', 'webpack'], ' ');
  element.classList.add('hello');
  
  const myIcon = new Image();
  myIcon.src = Icon;

  element.appendChild(myIcon);
  
  return element;
}

document.body.appendChild(component());
```

**src/style.css**

```css
.hello {
  color: red;
  background: url(./icon.jpg);
}
```

再`npm run build`之。现在你可以看到单独的图片和以图片为基础的背景图了。

### 加载字体

加载字体用的也是 file-loader。

**webpack.config.js**

```js
const path = require('path');

module.exports = {
  entry: './src/index.js',
  output: {
    filename: 'bundle.js', 
    path: path.resolve(__dirname, 'dist')
  },
  module: {
    rules: [
      {
        test: /\.css$/,
        use: [
          'style-loader',
          'css-loader'
        ]
      },
      {
        test: /\.(png|svg|jpg|gif)$/,
        use: [
          'file-loader'
        ]
      },
      { // 增加加载字体的规则
        test: /\.(woff|woff2|eot|ttf|otf)$/,
        use: [
          'file-loader'
        ]
      }
    ]
  }
};
```

在当前项目的目录中如下增加字体：

```
  webpack-demo
  |- package.json
  |- webpack.config.js
  |- /dist
    |- bundle.js
    |- index.html
  |- /src
+   |- my-font.ttf
    |- icon.jpg
    |- style.css
    |- index.js
  |- /node_modules
```

**src/style.css**

```css
@font-face {
  font-family: MyFont;
  src: url(./my-font.ttf);
}

.hello {
  color: red;
  background: url(./icon.jpg);
  font-family: MyFont;
}
```

运行打包命令之后便可以看到打包好的文件和发生改变的页面。

### 加载 JSON 文件

因为 webpack 对 JSON 文件的支持是内置的，所以可以直接添加。

**src/data.json**

```json
{
  "name": "webpack-demo",
  "version": "1.0.0",
  "author": "Sam Yang"
}
```

**src/index.js**

```js
import _ from 'lodash';
import './style.css';
import Icon from './icon.jpg';
import Data from './data.json'; // Data 变量包含可直接使用的 JSON 解析得到的对象

function component() {
  const element = document.createElement('div');
    
  element.innerHTML = _.join(['Hello', 'webpack'], ' ');
  element.classList.add('hello');

  const myIcon = new Image();
  myIcon.src = Icon;

  element.appendChild(myIcon);

  console.log(Data);
    
  return element;
}

document.body.appendChild(component());
```

关于其他文件的加载，可以寻求相应的 loader。

## 输出管理

前面我们只有一个输入文件，但现实是我们往往有不止一个输入文件，这时我们就需要输入多个入口文件并管理输出文件。我们在 src 目录下增加一个 print.js 文件。

**src/print.js**

```js
export default function printMe() {
  console.log('I get called from print.js!');
}
```

**src/index.js**

```js
import _ from 'lodash';
import printMe from './print.js';
// import './style.css';
// import Icon from './icon.jpg';
// import Data from './data.json';

function component() {
  const element = document.createElement('div');
  const btn = document.createElement('button');
    
  element.innerHTML = _.join(['Hello', 'webpack'], ' ');
  // element.classList.add('hello');

  // const myIcon = new Image();
  // myIcon.src = Icon;

  // element.appendChild(myIcon);

  // console.log(Data);

  btn.innerHTML = 'Click me and check the console!';
  btn.onclick = printMe;

  element.appendChild(btn);
    
  return element;
}

document.body.appendChild(component());
```

**dist/index.html**

```html
<!DOCTYPE html>
<html>
<head>
    <title>webpack demo</title>
    <script src="./print.bundle.js"></script>
</head>
<body>
    <!-- <script src="bundle.js"></script> -->
    <script src="./app.bundle.js"></script>
</body>
</html>
```

**webpack.config.js**

```js
const path = require('path');

module.exports = {
  // entry: './src/index.js',
  entry: {
    app: './src/index.js',
    print: './src/print.js'
  },
  output: {
    // filename: 'bundle.js',
    filename: '[name].bundle.js', // 根据入口起点名动态生成 bundle 名，可以使用像 "js/[name]/bundle.js" 这样的文件夹结构
    path: path.resolve(__dirname, 'dist')
  },
  // ...
};
```

> **Tips**
> `filename: '[name].bundle.js'`中的`[name]`会替换为对应的入口起点名，其他可用的替换请参见 [**output.filename**](https://webpack.js.org/configuration/output/#output-filename)。

现在可以打包文件了。但是如果我们修改了入口文件名或增加了入口文件，`index.html`是不会自动引用新文件的，而手动修改实在太挫。是时候使用插件 (*plugin*) 来完成这一任务了。我们使用 [HtmlWebpackPlugin](https://github.com/jantimon/html-webpack-plugin) 自动生成 html 文件。

> **loader 和 plugin，有什么区别？**
> loader (加载器)，重在“加载”二字，是用于预处理文件的，只用于在加载不同类型的文件时对不同类型的文件做相应的处理。而 plugin (插件)，顾名思义，是用来增加 webpack 的功能的，作用于整个 webpack 的构建过程。在 webpack 这个大公司中，loader 是保安大叔，负责对进入公司的不同人员的处理，而 plugin 则是公司里不同职位的职员，负责公司里的各种不同业务，每增加一种新型的业务需求，我们就需要增加一种 plugin。

安装插件：

```bash
$ npm i --save-dev html-webpack-plugin
```

**webpack.config.js**

```js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');

module.exports = {
  // entry: './src/index.js',
  entry: {
    app: './src/index.js',
    print: './src/print.js'
  },
  output: {
    // filename: 'bundle.js',
    filename: '[name].bundle.js',
    path: path.resolve(__dirname, 'dist')
  },
  plugins: [ // 插件属性，是插件的实例数组
    new HtmlWebpackPlugin({
      title: 'webpack demo',  // 生成 HTML 文档的标题
      filename: 'index.html' // 写入 HTML 文件的文件名，默认 `index.html`
    })
  ],
  // ...
};
```

你可以先把 dist 文件夹的`index.html`文件删除，然后执行打包命令。咻咻咻，我们看到 dist 目录下已经自动生成了一个`index.html`文件，但即使不删除原先的`index.html`，该插件默认生成的`index.html`也会替换原本的`index.html`。

此刻，当你细细观察 dist 目录时，虽然现在生成了新的打包文件，但原本的打包文件`bundle.js`及其他不用的文件仍然存在在 dist 目录中，所以在每次构建前我们需要晴空 dist 目录，我们使用 [CleanWebpackPlugin](https://github.com/johnagan/clean-webpack-plugin) 插件。

```bash
$ npm i clean-webpack-plugin --save-dev
```

**webpack.config.js**

```js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const CleanWebpackPlugin = require('clean-webpack-plugin');

module.exports = {
  // entry: './src/index.js',
  entry: {
    app: './src/index.js',
    print: './src/print.js'
  },
  output: {
    // filename: 'bundle.js',
    filename: '[name].bundle.js',
    path: path.resolve(__dirname, 'dist')
  },
  plugins: [
    new HtmlWebpackPlugin({
      title: 'webpack demo',
      filename: 'index.html'
    }),
    new CleanWebpackPlugin(['dist']) // 第一个参数是要清理的目录的字符串数组
  ],
  // ...
};
```

打包之，现在，dist 中只存在打包生成的文件。

## 开发环境

webpack 提供了很多便于开发时使用的功能，来一一看看吧。

### 使用代码映射 (*source map*)

当你的代码被打包后，如果打包后的代码发生了错误，你很难追踪到错误发生的原始位置，这个时候，我们就需要代码映射 (*source map*) 这种工具，它能将编译后的代码映射回原始的源码，你的错误是起源于打包前的`b.js`的某个位置，代码映射就能告诉你错误是那个模块的那个位置。webpack 默认提供了 10 种风格的代码映射，使用它们会明显影响到构建 (*build*) 和重构建 (*rebuild*，每次修改后需要重新构建) 的速度，十种风格的差异可以参看 [devtool](https://webpack.js.org/configuration/devtool/#devtool)。关于如何选择映射风格可以参看 [Webpack devtool source map](http://cheng.logdown.com/posts/2016/03/25/679045)。这里，我们为了准确显示错误位置，选择速度较慢的`inline-source-map`。

**webpack.config.js**

```js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const CleanWebpackPlugin = require('clean-webpack-plugin');

module.exports = {
  devtool: 'inline-source-map', // 控制是否生成以及如何生成 source map
  // entry: './src/index.js',
  entry: {
    app: './src/index.js',
    print: './src/print.js'
  },
  // ...
};
```

现在来手动制造一些错误：

**src/print.js**

```js
  export default function printMe() {
-   console.log('I get called from print.js!');
+   cosnole.log('I get called from print.js!');
  }
```

打包之后打开`index.html`再点击按钮，你就会看到控制台显示如下报错：

```
 Uncaught ReferenceError: cosnole is not defined
    at HTMLButtonElement.printMe (print.js:2)
```

现在，我们很清楚哪里发生了错误，然后轻松地改正之。

### 使用 webpack-dev-server

你一定有这样的体验，开发时每次修改代码保存后都需要重新手动构建代码并手动刷新浏览器以观察修改效果，这是很麻烦的，所以，我们要实时加载代码。可喜的是，webpack 提供了对实时加载代码的支持。我们需要安装 [webpack-dev-server](https://github.com/webpack/webpack-dev-server) 以获得支持。

```bash
$ npm i --save-dev webpack-dev-server
```

**webpack.config.js**

```js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const CleanWebpackPlugin = require('clean-webpack-plugin');

module.exports = {
  devtool: 'inline-source-map',
  devServer: { // 检测代码变化并自动重新编译并自动刷新浏览器
    contentBase: path.resolve(__dirname, 'dist') // 设置静态资源的根目录
  },
  // entry: './src/index.js',
  entry: {
    app: './src/index.js',
    print: './src/print.js'
  },
  // ...
};
```

**package.json**

```json
{
  ...
  "scripts": {
    "build": "webpack",
    "start": "webpack-dev-server --open"
  },
  ...
}
```

> **Tips**
> 使用 webpack-dev-server 时，webpack 并没有将所有生成的文件写入磁盘，而是放在内存中，提供更快的内存内访问，便于实时更新。

现在，可以直接运行`npm start` (`start`是 npm scripts 的内置属性，可直接运行)，然后浏览器自动加载应用的页面，默认在`localhost:8080`显示。

## 模块热替换 (*HMR, Hot Module Replacement*)

webpack 提供了对模块热替换 (或者叫热加载) 的支持。这一特性能够让应用运行的时候替换、增加或删除模块，而无需进行完全的重载。想进一步地了解其工作机理，可以参见 [Hot Module Replacement](https://webpack.js.org/concepts/hot-module-replacement/)，但这并不是必需的，你可以选择跳过机理部分继续往下阅读。

> **Tips**
> 模块热替换（*HMR*）只更新发生变更（替换、添加、删除）的模块，而无需重新加载整个页面（实时加载，*LiveReload*），这样可以显著加快开发速度，一旦打开了 webpack-dev-server 的 hot 模式，在试图重新加载整个页面之前，热模式会尝试使用 HMR 来更新。

**webpack.config.js**

```js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const CleanWebpackPlugin = require('clean-webpack-plugin');
const webpack = require('webpack'); // 引入 webpack 便于调用其内置插件

module.exports = {
  devtool: 'inline-source-map',
  devServer: {
    contentBase: path.resolve(__dirname, 'dist'),
    hot: true, // 告诉 dev-server 我们在用 HMR
    hotOnly: true // 指定如果热加载失败了禁止刷新页面 (这是 webpack 的默认行为)，这样便于我们知道失败是因为何种错误
  },
  // entry: './src/index.js',
  entry: {
    app: './src/index.js',
    // print: './src/print.js'
  },
  // ...
  plugins: [
    new HtmlWebpackPlugin({
      title: 'webpack demo',
      filename: 'index.html'
    }),
    new CleanWebpackPlugin(['dist']),
    new webpack.HotModuleReplacementPlugin(), // 启用 HMR
    new webpack.NamedModulesPlugin() // 打印日志信息时 webpack 默认使用模块的数字 ID 指代模块，不便于 debug，这个插件可以将其替换为模块的真实路径
  ],
  // ...
};
```

> **Tips**
> webpack-dev-server 会为每个入口文件创建一个客户端脚本，这个脚本会监控该入口文件的依赖模块的更新，如果该入口文件编写了 HMR 处理函数，它就能接收依赖模块的更新，反之，更新会向上冒泡，直到客户端脚本仍没有处理函数的话，webpack-dev-server 会重新加载整个页面。如果入口文件本身发生了更新，因为向上会冒泡到客户端脚本，并且不存在 HMR 处理函数，所以会导致页面重载。

我们已经开启了 HMR 的功能，HMR 的接口已经暴露在`module.hot`属性之下，我们只需要调用 [HMR API](https://webpack.js.org/api/hot-module-replacement/) 即可实现热加载。当“被加载模块”发生改变时，依赖该模块的模块便能检测到改变并接收改变之后的模块。

**src/index.js**

```js
import _ from 'lodash';
import printMe from './print.js';
// import './style.css';
// import Icon from './icon.jpg';
// import Data from './data.json';

function component() {
  const element = document.createElement('div');
  const btn = document.createElement('button');
    
  element.innerHTML = _.join(['Hello', 'webpack'], ' ');
  // element.classList.add('hello');

  // const myIcon = new Image();
  // myIcon.src = Icon;

  // element.appendChild(myIcon);

  // console.log(Data);

  btn.innerHTML = 'Click me and check the console!';
  btn.onclick = printMe;

  element.appendChild(btn);
    
  return element;
}

document.body.appendChild(component());

if(module.hot) { // 习惯上我们会检查是否可以访问 `module.hot` 属性
  module.hot.accept('./print.js', function() { // 接受给定依赖模块的更新，并触发一个回调函数来对这些更新做出响应
    console.log('Accepting the updated printMe module!');
    printMe();
  });
}
```

`npm start`之。为了演示效果，我们做如下修改：

**src/print.js**

```js
  export default function printMe() {
-   console.log('I get called from print.js!');
+   console.log('Updating print.js...');
  }
```

我们会看到控制台打印出的信息中含有以下几行：

```bash
index.js:33 Accepting the updated printMe module!
print.js:2 Updating print.js...
log.js:23 [HMR] Updated modules:
log.js:23 [HMR]  - ./src/print.js
log.js:23 [HMR] App is up to date.
```

> **Tips**
> webpack-dev-server 在 [**inline mode**](https://webpack.js.org/configuration/dev-server/#devserver-inline) (此为默认模式) 时，会为每个入口起点 (*entry*) 创建一个客户端脚本，所以你会在上面的输出中看到有些信息重复输出两次。

但是当你点击页面的按钮时，你会发现控制台输出的是旧的`printMe`函数输出的信息，因为`onclick`事件绑定的仍是原始的`printMe`函数。我们需要在`module.hot.accept`里更新绑定。

**src/index.js**

```js
import _ from 'lodash';
import printMe from './print.js';
// import './style.css';
// import Icon from './icon.jpg';
// import Data from './data.json';

// ...

// document.body.appendChild(component());
var element = component();
document.body.appendChild(element);

if(module.hot) {
  module.hot.accept('./print.js', function() {
    console.log('Accepting the updated printMe module!');
    // printMe();
    
    document.body.removeChild(element);
    element = component();
    document.body.appendChild(element);
  });
}
```

> **Tips**
> [**uglifyjs-webpack-plugin**](https://github.com/webpack-contrib/uglifyjs-webpack-plugin) 升级到 v0.4.6 时无法正确压缩 ES6 的代码，所以上面有些代码采用 ES5 以暂时方便后面的压缩，详见 [#49](https://github.com/webpack-contrib/uglifyjs-webpack-plugin/issues/49)。

模块热替换也可以用于样式的修改，效果跟控制台修改一样一样的。

**src/index.js**

```js
import _ from 'lodash';
import printMe from './print.js';
import './style.css';
// import Icon from './icon.jpg';
// import Data from './data.json';

// ...
```

`npm start`之，做如下修改：

```css
/* ... */

body {
  background-color: yellow;
}
```

可以发现在不重载页面的前提下我们对样式的修改进行了热加载，棒！

## 生产环境

### 自动方式

我们只需要运行`webpack -p` (相当于 `webpack --optimize-minimize --define process.env.NODE_ENV="'production'"`)这个命令，便可以自动构建生产版本的应用，这个命令会完成以下步骤：

* 使用 `UglifyJsPlugin` (*webpack.optimize.UglifyJsPlugin*) 压缩 JS 文件 (此插件和  [uglifyjs-webpack-plugin](https://github.com/webpack-contrib/uglifyjs-webpack-plugin) 相同)
* 运行 `LoaderOptionsPlugin` 插件，这个插件是用来迁移的，见 [document](https://webpack.js.org/plugins/loader-options-plugin/)
* 设置 NodeJS 的环境变量，触发某些 package 包以不同方式编译

值得一提的是，`webpack -p`设置的`process.env.NODE_ENV`环境变量，是用于编译后的代码的，只有在打包后的代码中，这一环境变量才是有效的。如果在 webpack 配置文件中引用此环境变量，得到的是 undefined，可以参见 [#2537](https://github.com/webpack/webpack/issues/2537)。但是，有时我们确实需要在 webpack 配置文件中使用 `process.env.NODE_ENV`，怎么办呢？一个方法是运行`NODE_ENV='production' webpack -p`命令，不过这个命令在Windows中是会出问题的。为了解决兼容问题，我们采用 [cross-env](https://github.com/kentcdodds/cross-env) 解决跨平台的问题。

```bash
$ npm i --save-dev cross-env
```

**package.json**

```json
{
  ...
  "scripts": {
    "build": "cross-env NODE_ENV=production webpack -p",
    "start": "webpack-dev-server --open"
  },
  ...
}
```

现在可以在配置文件中使用`process.env.NODE_ENV`了。

**webpack.config.js**

```js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const CleanWebpackPlugin = require('clean-webpack-plugin');
const webpack = require('webpack');

module.exports = {
  // ...
  output: {
    // filename: 'bundle.js',
    // filename: '[name].bundle.js',
    filename: process.env.NODE_ENV === 'production' ? '[name].[chunkhash].js' : '[name].bundle.js', // 在配置文件中使用`process.env.NODE_ENV`
    path: path.resolve(__dirname, 'dist')
  },
  plugins: [
    new HtmlWebpackPlugin({
      title: 'webpack demo',
      filename: 'index.html'
    }),
    new CleanWebpackPlugin(['dist']),
    // new webpack.HotModuleReplacementPlugin(), // 关闭 HMR 功能
    new webpack.NamedModulesPlugin()
  ],
  // ...
};
```

> **Tips**
> [chunkhash]不能和 HMR 一起使用，换句话说，不应该在开发环境中使用 [chunkhash] (或者 [hash])，这会导致许多问题。详情见 [#2393](https://github.com/webpack/webpack/issues/2393) 和 [#377](https://github.com/webpack/webpack-dev-server/issues/377)。

build 之，我们得到了生产版本的压缩好的打包文件。

### 多配置文件配置

有时我们会需要为不同的环境配置不同的配置文件，可以选择 [简易方法](https://webpack.js.org/guides/production/#simple-approach)，这里我们采用较为先进的方法。先准备一个基本的配置文件，包含了所有环境都包含的配置，然后用 [webpack-merge](https://github.com/survivejs/webpack-merge) 将它和特定环境的配置文件合并并导出，这样就减少了基本配置的重复。

```bash
$ npm i --save-dev webpack-merge
```

**webpack.common.js**

```js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const CleanWebpackPlugin = require('clean-webpack-plugin');

module.exports = {
  entry: {
    app: './src/index.js',
    print: './src/print.js'
  },
  output: {
    path: path.resolve(__dirname, 'dist')
  },
  plugins: [
    new HtmlWebpackPlugin({
      title: 'webpack demo',
      filename: 'index.html'
    }),
    new CleanWebpackPlugin(['dist'])
  ],
  module: {
    rules: [
      {
        test: /\.css$/,
        use: [
          'style-loader',
          'css-loader'
        ]
      },
      {
        test: /\.(png|svg|jpg|gif)$/,
        use: [
          'file-loader'
        ]
      },
      {
        test: /\.(woff|woff2|eot|ttf|otf)$/,
        use: [
          'file-loader'
        ]
      }
    ]
  }
};
```

**webpack.dev.js**

```js
const path = require('path');
const webpack = require('webpack');
const Merge = require('webpack-merge');
const CommonConfig = require('./webpack.common.js');

module.exports = Merge(CommonConfig, {
  devtool: 'cheap-module-eval-source-map',
  devServer: {
    contentBase: path.resolve(__dirname, 'dist'),
    hot: true,
    hotOnly: true
  },
  output: {
    filename: '[name].bundle.js'
  },
  plugins: [
    new webpack.DefinePlugin({
      'process.env.NODE_ENV': JSON.stringify('development') // 在编译的代码里设置了`process.env.NODE_ENV`变量
    }),
    new webpack.HotModuleReplacementPlugin(),
    new webpack.NamedModulesPlugin()
  ]
});
```

**webpack.prod.js**

```js
const path = require('path');
const webpack = require('webpack');
const Merge = require('webpack-merge');
const CommonConfig = require('./webpack.common.js');

module.exports = Merge(CommonConfig, {
  devtool: 'cheap-module-source-map',
  output: {
    filename: '[name].[chunkhash].js'
  },
  plugins: [
    new webpack.DefinePlugin({
      'process.env.NODE_ENV': JSON.stringify('production')
    }),
    new webpack.optimize.UglifyJsPlugin()
  ]
});
```

**package.json**

```json
{
  ...
  "scripts": {
    "build": "cross-env NODE_ENV=production webpack -p",
    "start": "webpack-dev-server --open",
    "build:dev": "webpack-dev-server --open --config webpack.dev.js",
    "build:prod": "webpack --progress --config webpack.prod.js"
  },
  ...
}
```

现在只需执行`npm run build:dev`或`npm run build:prod`便可以得到开发版或者生产版了！

> **Tips**
> webpack 命令行选项见 [Command Line Interface](https://webpack.js.org/api/cli/)。

## 代码分离

### 入口分离

我们先创建一个新文件：

```bash
$ cd src && touch another.js
```

**src/another.js**

```js
import _ from 'lodash';

console.log(_.join(['Another', 'module', 'loaded!'], ' '));
```

**webpack.config.js**

```js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const CleanWebpackPlugin = require('clean-webpack-plugin');
const webpack = require('webpack');

module.exports = {
  // ...
  entry: {
    app: './src/index.js',
    // print: './src/print.js'
    another: './src/another.js'
  },
  // ...
};
```

`cd .. && npm run build`之，我们发现用入口分离的代码得到了两个大文件，这是因为两个入口文件都引入了`lodash`，这很大程度上造成了冗余，在同一个页面中我们只需要引入一个`lodash`就可以了。

### 抽取相同部分

我们使用 [CommonsChunkPlugin](https://webpack.js.org/plugins/commons-chunk-plugin/) 插件来将相同的部分提取出来放到一个单独的模块中。

**webpack.config.js**

```js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const CleanWebpackPlugin = require('clean-webpack-plugin');
const webpack = require('webpack');

module.exports = {
  // devtool: 'inline-source-map',
  // ...
  output: {
    // filename: 'bundle.js',
    filename: '[name].bundle.js',
    // filename: process.env.NODE_ENV === 'production' ? '[name].[chunkhash].js' : '[name].bundle.js',
    path: path.resolve(__dirname, 'dist')
  },
  plugins: [
    new HtmlWebpackPlugin({
      title: 'webpack demo',
      filename: 'index.html'
    }),
    new CleanWebpackPlugin(['dist']),
    new webpack.optimize.CommonsChunkPlugin({
      name: 'common' // 抽取出的模块的模块名
    }),
    // new webpack.HotModuleReplacementPlugin(),
    // new webpack.NamedModulesPlugin()
  ],
  // ...
};
```

build 之，可以看到结果中包含以下部分：

```bash
    app.bundle.js    6.14 kB       0  [emitted]  app
another.bundle.js  185 bytes       1  [emitted]  another
 common.bundle.js    73.2 kB       2  [emitted]  common
       index.html  314 bytes          [emitted]
```

我们把`lodash`分离出来了。

### 动态引入

我们还可以选择以动态引入的方式来实现代码分离，借助 [*import()*](https://webpack.js.org/api/module-methods/#import-) 实现之。

**webpack.config.js**

```js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const CleanWebpackPlugin = require('clean-webpack-plugin');
// const webpack = require('webpack');

module.exports = {
  // ...
  entry: {
    app: './src/index.js',
    // print: './src/print.js'
    // another: './src/another.js'
  },
  output: {
    // filename: 'bundle.js',
    filename: '[name].bundle.js',
    chunkFilename: '[name].bundle.js', // 指定非入口块文件输出的名字
    // filename: process.env.NODE_ENV === 'production' ? '[name].[chunkhash].js' : '[name].bundle.js',
    path: path.resolve(__dirname, 'dist')
  },
  plugins: [
    new HtmlWebpackPlugin({
      title: 'webpack demo',
      filename: 'index.html'
    }),
    new CleanWebpackPlugin(['dist'])
    // new webpack.optimize.CommonsChunkPlugin({
    //   name: 'common'
    // }),
    // new webpack.HotModuleReplacementPlugin(),
    // new webpack.NamedModulesPlugin()
  ],
  // ...
};
```

**src/index.js**

```js
// import _ from 'lodash';
import printMe from './print.js';
// import './style.css';
// import Icon from './icon.jpg';
// import Data from './data.json';

function component() {
  // 此函数原来的内容全部注释掉...

  return import(/* webpackChunkName: "lodash" */ 'lodash').then(function(_) {
    const element = document.createElement('div');
    const btn = document.createElement('button');

    element.innerHTML = _.join(['Hello', 'webpack'], ' ');

    btn.innerHTML = 'Click me and check the console!';
    btn.onclick = printMe;

    element.appendChild(btn);

    return element;
  }).catch(function(error) {
    console.log('An error occurred while loading the component')
  });
}

// document.body.appendChild(component());
// var element = component();
// document.body.appendChild(element);

// 原本热加载的部分全部注释掉...

component().then(function(component) {
   document.body.appendChild(component);
 });
```

> **Tips**
> 注意上面中的`/* webpackChunkName: "lodash" */`这段注释，它并不是可有可无的，它能帮助我们结合`output.chunkFilename`把分离出的模块最终命名为`lodash.bundle.js`而非`[id].bundle.js`。

现在 build 之看看吧。

## 懒加载 (*lazy loading*)

既然有了`import()`，我们可以选择在需要的时候才加载相应的模块，减少了应用初始化时加载大量暂不需要的模块的压力，这能让我们的应用更高效地运行。

**src/print.js**

```js
console.log('The print.js module has loaded! See the network tab in dev tools...');

export default function printMe() {
  // console.log('Updating print.js...');
  console.log('Button Clicked: Here\'s "some text"!');
}
``` 

**src/index.js**

```js
import _ from 'lodash';
// 其他引入注释...

function component() {
  const element = document.createElement('div');
  const btn = document.createElement('button');
    
  element.innerHTML = _.join(['Hello', 'webpack'], ' ');
  // element.classList.add('hello');

  // const myIcon = new Image();
  // myIcon.src = Icon;

  // element.appendChild(myIcon);

  // console.log(Data);

  btn.innerHTML = 'Click me and check the console!';
  // btn.onclick = printMe;

  element.appendChild(btn);

  btn.onclick = function() {
    import(/* webpackChunkName: "print" */ './print')
    .then(function(module) {
      const printMe = module.default; // 引入模块的默认函数

      printMe();
    });
  };
    
  return element;

  // 原本的动态引入注释...
}

document.body.appendChild(component());
// var element = component();
// document.body.appendChild(element);

// 热加载部分注释

// component().then(function(component) {
//    document.body.appendChild(component);
//  });
```

构建之，控制台此时并无输出，点击按钮，会看到控制台如下输出：

```bash
print.bundle.js:1 The print.js module has loaded! See the network tab in dev tools...
print.bundle.js:1 Button Clicked: Here's "some text"!
```

说明 print 模块只在我们点击时才引入了，すっげえ！

## 缓存 (*caching*)

浏览器在初次加载网站时，会下载很多文件，为了较少下载大量资源的压力，浏览器会对资源进行缓存 (*caching*)，这样浏览器便可以更迅速地加载网站，但是我们需要在文件内容发生改变时更新文件。

我们可以在输出文件名上下手脚：

**webpack.config.js**

```js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const CleanWebpackPlugin = require('clean-webpack-plugin');
// const webpack = require('webpack');

module.exports = {
  // ...
  output: {
    // filename: 'bundle.js',
    filename: '[name].[chunkhash].js',
    // chunkFilename: '[name].bundle.js',
    // filename: process.env.NODE_ENV === 'production' ? '[name].[chunkhash].js' : '[name].bundle.js',
    path: path.resolve(__dirname, 'dist')
  },
  // ...
};
``` 

> **Tips**
> [chunkhash] 是内容相关的，只要内容发生了改变，构建后文件名的 hash 就会发生改变。

还有一个要点是提取出第三方库放到单独模块中，因为它们是不太可能频繁发生改变的，所以无需多次加载这些模块，提取的方法用 [CommonsChunkPlugin](https://webpack.js.org/plugins/commons-chunk-plugin/) 插件，这个插件[上文](#抽取相同部分)中提到过，指定入口文件名时它会提取改入口文件为单个文件，不指定则会提取 webpack 的运行时代码。

**webpack.config.js**

```js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const CleanWebpackPlugin = require('clean-webpack-plugin');
const webpack = require('webpack');

module.exports = {
  // ...
  entry: {
    app: './src/index.js',
    vendor: [ // 第三方库可以统一放在这个入口一起合并
      'lodash'
    ]
    // print: './src/print.js'
    // another: './src/another.js'
  },
  output: {
    // filename: 'bundle.js',
    filename: '[name].[chunkhash].js',
    chunkFilename: '[name].bundle.js',
    // filename: process.env.NODE_ENV === 'production' ? '[name].[chunkhash].js' : '[name].bundle.js',
    path: path.resolve(__dirname, 'dist')
  },
  plugins: [
    new HtmlWebpackPlugin({
      title: 'webpack demo',
      filename: 'index.html'
    }),
    new CleanWebpackPlugin(['dist']),
    new webpack.optimize.CommonsChunkPlugin({
      name: 'vendor' // 将 vendor 入口处的代码放入 vendor 模块
    }),
    new webpack.optimize.CommonsChunkPlugin({
      name: 'runtime' // 将 webpack 自身的运行时代码放在 runtime 模块
    })
    // new webpack.HotModuleReplacementPlugin(),
    // new webpack.NamedModulesPlugin()
  ],
  // ...
};
``` 

> **Tips**
> 包含 vendor 的 CommonsChunkPlugin 实例必须在包含 runtime 的之前，否则会报错。

**src/index.js**

```js
// import _ from 'lodash';
// ...

// ...
```

如果我们在 src 下新建一个文件`h.js`，再在`index.js`中引入它，保存，构建之，我们发现有些没改变的模块的 hash 也发生了改变，这是因为加入`h.js`后它们的`module.id`变了，但这明显是不合理的。在开发环境，我们可以用 NamedModulesPlugin 将 id 换成具体路径名。而在生产环境，我们可以使用 [HashedModuleIdsPlugin](https://webpack.js.org/plugins/hashed-module-ids-plugin/)。

**webpack.config.js**

```js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const CleanWebpackPlugin = require('clean-webpack-plugin');
const webpack = require('webpack');

module.exports = {
  // ...
  plugins: [
    new HtmlWebpackPlugin({
      title: 'webpack demo',
      filename: 'index.html'
    }),
    new webpack.HashedModuleIdsPlugin(), // 替换掉原来的`module.id`
    new CleanWebpackPlugin(['dist']),
    new webpack.optimize.CommonsChunkPlugin({
      name: 'vendor'
    }),
    new webpack.optimize.CommonsChunkPlugin({
      name: 'runtime'
    })
    // new webpack.HotModuleReplacementPlugin(),
    // new webpack.NamedModulesPlugin()
  ],
  // ...
};
``` 

再来执行刚才那波操作，就会发现无关修改的模块 hash 未变了。

## *Shimming*

> **Tips**
> 你可以将 shim 简单理解为是用于兼容 API 的小型库。

使用 jQuery 时我们习惯性地使用`$`或`jQuery`变量，每次都使用`const $ = require(“jquery”)`引入的话太麻烦，如果能直接把这两个变量设置为全局变量岂不美滋滋？这样就可以在每个模块中直接使用这两个变量了。为了兼容这一做法，我们使用 [ProvidePlugin](https://webpack.js.org/plugins/provide-plugin/) 插件为我们完成这一任务。

```bash
$ npm i --save jquery
```

**webpack.config.js**

```js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const CleanWebpackPlugin = require('clean-webpack-plugin');
const webpack = require('webpack');

module.exports = {
  // ...
  plugins: [
    new HtmlWebpackPlugin({
      title: 'webpack demo',
      filename: 'index.html'
    }),
    new webpack.ProvidePlugin({ // 设置全局变量
      $: 'jquery',
      jQuery: 'jquery'
    }),
    new webpack.HashedModuleIdsPlugin(),
    new CleanWebpackPlugin(['dist']),
    new webpack.optimize.CommonsChunkPlugin({
      name: 'vendor'
    }),
    new webpack.optimize.CommonsChunkPlugin({
      name: 'runtime'
    })
    // new webpack.HotModuleReplacementPlugin(),
    // new webpack.NamedModulesPlugin()
  ],
  // ...
};
``` 

**src/print.js**

```js
console.log('The print.js module has loaded! See the network tab in dev tools...');
console.log($('title').text()); // 使用 jQuery

export default function printMe() {
  // console.log('Updating print.js...');
  console.log('Button Clicked: Here\'s "some text"!');
}
```

build，点击页面按钮，成功了。

另外，如果你需要在某些模块加载时设置该模块的全局变量，请看 [这里](https://webpack.js.org/guides/shimming/#imports-loader)。

## 结尾的一点废话

终于写完了 ：），也感谢你能耐心看到这里。webpack 这个工具的配置还是有些麻烦的。但是呢，某人说这个东东前期会花比较多时间，后期会大大提高你的效率。所以呢，还是拿下这个东东吧。有其他需求的话可以继续看[官方的文档](https://webpack.js.org/configuration/)。遇到困难可以找：

* [Stack Overflow](https://stackoverflow.com/)
* [Google](https://www.google.com)
* [Gitter](https://gitter.im/webpack/webpack)
* [Webpack Issues](https://github.com/webpack/webpack/issues)

我写好的 demo 文件放在了[这里](https://github.com/yangkean/webpack-demo)。

## Reference

* [入门 Webpack，看这篇就够了 - v1](https://segmentfault.com/a/1190000006178770)
* [Webpack Guides](https://webpack.js.org/guides/)
* [Webpack: When To Use And Why](http://blog.andrewray.me/webpack-when-to-use-and-why/)
* stackoverflow / github issues
