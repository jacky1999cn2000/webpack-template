# webpack template

### Before

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
