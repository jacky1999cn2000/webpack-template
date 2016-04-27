# webpack template

survivejs-kanban这本书里面讲了如何使用webpack作为building tool来进行react开发,书的第三章基本搭建了dev时需要的 *webpack.config.js* 文件,书的第八章做了进一步修改,支持了build和stats。

下面分两部分来解释 *webpack.config.js* 的演化过程

### development

这是不考虑build前的 *webpack.config.js* 文件。

注释：

1. merge:顾名思义是merge object的,是一个特意为 *webpack.config.js* 文件写的方法
2. NpmInstallPlugin: 一个探测config文件中使用到的npm包并自动安装的工具
3. TARGET: 用来捕获npm事件,比如`npm start`,`npm run build`,`npm run stats`的事件分别是start,build,stats
4. BABEL_ENV: 这是默认的babel环境变量,将此环境变量的值赋成上面的TARGET,然后在.babelrc文件里可以根据这个值来决定是否load某些presets(下面例子中只有在npm start的情况下才载入react-hmre)

*.babelrc*

```javascript
{
  "presets": [
    "es2015",
    "react",
    "survivejs-kanban"
  ],
  "env": {
    "start": {
      "presets": [
        "react-hmre"
      ]
    }
  }
}

```
5. path是node.js的标准类; `__dirname`是当前目录; PATHS变量里定义了两个文件夹入口: app & build;
6. common是基础的config object,然后根据TARGET的值(start or build),分别拓展config的定义
7. entry定义webpack开始遍历的入口,如未确定具体文件,则默认是index.js
8. resolve定义了react code里当import文件时哪些后缀可以省略
9. output定义了webpack输出的路径和文件名
10. common里面定义了两个loaders,一个用来处理.css文件,一个用来处理js(x)文件
  * test正则表达式确定哪些文件该loader处理
  * loaders定义了哪些loaders来出来这些符合条件的文件(loader处理顺序从右到左)
  * include缩小查找范围
  * 这里用babel来transform jsx文件,而babel的config在.babelrc中定义
11. 如果TARGET是start,则通过merge来增加新的config
12. devtool用来设置source-map,从而方便debug
13. devServer用了webpack自带的devServer,并提供了很多便于debug的功能
14. plugins用了两个,一个是HotModuleReplacementPlugin,另一个是之前require的那个自动安装npm包的工具
15. package.json里主要注意devDependencies即可,babel-preset-survivejs-kanban是作者自定义的一个npm包,主要包含了一些用ES6开发React的utils,详情去Google


*webpack.config.js*

```javascript
const path = require('path');
const merge = require('webpack-merge');
const webpack = require('webpack');
const NpmInstallPlugin = require('npm-install-webpack-plugin');

const TARGET = process.env.npm_lifecycle_event;
const PATHS = {
  app: path.join(__dirname, 'app'),
  build: path.join(__dirname, 'build')
};

process.env.BABEL_ENV = TARGET;

const common = {
  entry: {
    app: PATHS.app
  },
  resolve: {
    extensions: ['', '.js', '.jsx']
  },
  output: {
    path: PATHS.build,
    filename: 'bundle.js'
  },
  module: {
    loaders: [
      {
        test: /\.css$/,
        loaders: ['style', 'css'],
        include: PATHS.app
      },
      {
        test: /\.jsx?$/,
        loaders: ['babel?cacheDirectory'],
        include: PATHS.app
      }
    ]
  }
};

if(TARGET === 'start' || !TARGET) {
  module.exports = merge(common, {
    devtool: 'eval-source-map',
    devServer: {
      contentBase: PATHS.build,

      historyApiFallback: true,
      hot: true,
      inline: true,
      progress: true,

      // display only errors to reduce the amount of output
      stats: 'errors-only',

      // parse host and port from env so this is easy
      // to customize
      host: process.env.HOST,
      port: process.env.PORT
    },
    plugins: [
      new webpack.HotModuleReplacementPlugin(),
      new NpmInstallPlugin({
        save: true // --save
      })
    ]
  });
}

if(TARGET === 'build') {
  module.exports = merge(common, {});
}
```
*package.json*

```javascript
{
  "name": "kanban_app",
  "version": "0.0.0",
  "description": "Webpack demo app",
  "main": "index.js",
  "scripts": {
    "build": "webpack",
    "start": "webpack-dev-server"
  },
  "keywords": [
    "webpack",
    "demo"
  ],
  "author": "",
  "license": "MIT",
  "devDependencies": {
    "babel-core": "^6.5.2",
    "babel-loader": "^6.2.2",
    "babel-preset-es2015": "^6.5.0",
    "babel-preset-react": "^6.5.0",
    "babel-preset-react-hmre": "^1.1.0",
    "babel-preset-survivejs-kanban": "^0.3.3",
    "css-loader": "^0.23.1",
    "npm-install-webpack-plugin": "^2.0.2",
    "style-loader": "^0.13.0",
    "webpack": "^1.12.13",
    "webpack-dev-server": "^1.14.1",
    "webpack-merge": "^0.7.3"
  },
  "dependencies": {
    "alt": "^0.18.2",
    "alt-container": "^1.0.2",
    "alt-utils": "^1.0.0",
    "node-uuid": "^1.4.7",
    "react": "^0.14.7",
    "react-addons-update": "^0.14.7",
    "react-dnd": "^2.1.0",
    "react-dnd-html5-backend": "^2.1.2",
    "react-dom": "^0.14.7"
  }
}
```
