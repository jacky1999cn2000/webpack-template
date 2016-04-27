# webpack template

survivejs-kanban这本书里面讲了如何使用webpack作为building tool来进行react开发,书的第三章基本搭建了dev时需要的 *webpack.config.js* 文件,书的第八章做了进一步修改,支持了build和stats。

下面分两部分来解释 *webpack.config.js* 的演化过程

### Before

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

### After

这是要考虑build的 *webpack.config.js* 文件

npm start: 进入开发模式
npm run build: 在build文件夹里生成各种文件,然后可以用serve工具来serve
npm run stats: 在build文件夹里生成各种文件,并生成一个stats.json文件

注释:

1. Minification
  * webpack.optimize.UglifyJsPlugin用来minify js文件
  * webpack.DefinePlugin将`process.env.NODE_ENV`设置成了`production`,这样React自己会做一些优化(比如取消property type checks等)
2. Splitting app and vendor Bundles
  * package.json文件中有dependencies和devDependencies的划分,这里将所有dependencies里的东西弄成一个vendor.js(要检查dependencies和devDependencies是否分类准确)
  * 将package.json文件load进来
  * 修改common object中的output设置,将输出文件命名为entry name
  * 在TARGET=build里增加一个名为vendor的entry
  * 在TARGET=build里使用CommonsChunkPlugin,将app.js中的相关code抽离出来放到vendor.js当中,并生成一个manifest文件(该文件告知webpack哪些module对应哪些文件)
  * 此例中alt-utils这个dependency因为自己的问题不能被放入vendor.js文件里,所以代码中做了处理
3. Adding Hashes to Filenames
  * webpack自带一些placeholder,比如[name]返回entry name,[hash]返回build hash,[chunkhash]返回chunk hash
  * 使用这些placeholder可以生成诸如`app.d587bbd6e38337f5accd.js`,`vendor.dc746a5db4ed650296e1.js`这样的文件
  * 每次build时候,如果文件内容变化,则chunkhash也会变化,这样的话其实就是变相的invalidate了浏览器的cache,浏览器就可以request新的文件
  * 在TARGET=build里面加入output参数,里面重新定义filename的格式,并定义chunkFilename
4. Generating index.html through html-webpack-plugin
  * 安装 html-webpack-template 和 html-webpack-plugin
  * 删除./build/index.html文件,因为html-webpack-plugin会帮我们生成
  * config文件里引入html-webpack-plugin
  * 在common object中加入HtmlWebpackPlugin的定义
  * 在TARGET=start里面删除devServer中这个设置:contentBase: PATHS.build
5. Cleaning the Build
  * 安装clean-webpack-plugin
  * config文件里引用clean-webpack-plugin
  * 在TARGET=build里面增加CleanPlugin(注意放在第一位,这样每次build的时候就先清理build文件夹)
6. Separating CSS
  * 目的是在build的时候把.css文件和.js文件区分开
  * 安装 extract-text-webpack-plugin
  * 因为extract-text-webpack-plugin和Hot Module Replacement (HMR)不兼容,所以我们只用在build时候,因此需要把对.css文件处理的loader从common object中移除,分别在start和build中定义
  * config文件中引用extract-text-webpack-plugin
  * 将common object中对于.css文件处理的loader删除,放在start里面
  * 在build里面设置一个新的处理.css文件的loader,并使用ExtractTextPlugin
  * 在build的plugins中添加ExtractTextPlugin用来.css文件控制输出格式
7. Analyzing Build Statistics
  * 在package.json的scripts中添加`"stats": "webpack --profile --json > stats.json"`
  * 将config文件中这一行改成`if(TARGET === 'build' || TARGET === 'stats')`
  * 运行npm run stats命令会生成一个stats.json文件,然后在 http://webpack.github.io/analyse/ 里上传这个文件, 看一些分析结果

```javascript
const path = require('path');
const merge = require('webpack-merge');
const webpack = require('webpack');
const NpmInstallPlugin = require('npm-install-webpack-plugin');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const CleanPlugin = require('clean-webpack-plugin');
const ExtractTextPlugin = require('extract-text-webpack-plugin');

const pkg = require('./package.json');

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
    filename: '[name].js'
  },
  module: {
    loaders: [
      {
        test: /\.jsx?$/,
        loaders: ['babel?cacheDirectory'],
        include: PATHS.app
      }
    ]
  },
  plugins: [
    new HtmlWebpackPlugin({
      template: 'node_modules/html-webpack-template/index.ejs',
      title: 'Kanban app',
      appMountId: 'app',
      inject: false
    })
  ]
};

if(TARGET === 'start' || !TARGET) {
  module.exports = merge(common, {
    devtool: 'eval-source-map',
    devServer: {

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
    module: {
      loaders: [
        // Define development specific CSS setup
        {
          test: /\.css$/,
          loaders: ['style', 'css'],
          include: PATHS.app
        }
      ]
    },
    plugins: [
      new webpack.HotModuleReplacementPlugin(),
      new NpmInstallPlugin({
        save: true // --save
      })
    ]
  });
}

if(TARGET === 'build' || TARGET === 'stats') {
  module.exports = merge(common, {
    entry: {
      vendor: Object.keys(pkg.dependencies).filter(function(v) {
        // Exclude alt-utils as it won't work with this setup
        // due to the way the package has been designed
        // (no package.json main).
        return v !== 'alt-utils';
      })
    },
    output: {
      path: PATHS.build,
      filename: '[name].[chunkhash].js',
      chunkFilename: '[chunkhash].js'
    },
    module: {
      loaders: [
        // Extract CSS during build
        {
          test: /\.css$/,
          loader: ExtractTextPlugin.extract('style', 'css'),
          include: PATHS.app
        }
      ]
    },
    plugins: [
      new CleanPlugin([PATHS.build]),
      // Output extracted CSS to a file
      new ExtractTextPlugin('[name].[chunkhash].css'),
      // Extract vendor and manifest files
      new webpack.optimize.CommonsChunkPlugin({
        names: ['vendor', 'manifest']
      }),
      // Setting DefinePlugin affects React library size!
      new webpack.DefinePlugin({
        'process.env.NODE_ENV': '"production"'
      }),
      new webpack.optimize.UglifyJsPlugin({
        compress: {
          warnings: false
        }
      })
    ]
  });
}
```
