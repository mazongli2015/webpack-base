# webpack-base
webpack 的一些常用配置。

#安装命令
```command
yarn install
```

#eslint 配置
0表示关闭规则，1表示违反规则会报警告，2表示违反规则会报错
```json
{
    "extends": "eslint:recommended",  // 继承默认配置
    "parserOptions": {
        "parserOptions": { "ecmaVersion": 6 },  // es版本
        "sourceType"   : "module",  // 模块化模式，否则不能使用export和import
        "ecmaFeatures" : {
            "jsx": true  // 启用jsx语法
        }
    },
    "env": { 
        "es6"     : true,  // es6版本环境
        "browser" : true // 浏览器环境，否则直接使用document对象时，会报错
    },
    "rules": {
        "consistent-return"   : 2, //  不能直接return undefined
        "indent"              : [1, 4],  // 缩进4个空格
        "no-else-return"      : 1, // else 后面不能直接写return
        "semi"                : [1, "always"],  // 语句结束后加分号
        "space-unary-ops"     : 2, // new, delete, typeof, void, yield, -, +, --, ++, !, !!的空隙
        "no-unused-vars"      : 1, // 不能使用没有用的变量
        "includeExports"      : true, // 使用export
        "includeImports"      : true, // 使用import
        "no-duplicate-imports": 2 // 不能重复import同样的东西
    }
}
```

#babel配置
```javascript
const presets = [
    [
      "@babel/env", // babel环境，stage-0~stage2等在V7.3的版本已经被删除，统一使用@babel/env
      {
        useBuiltIns: "usage",  // polyfill的引入方式
        loose: true, // 不使用Object.definedProperties转换类的es5形式
      },
    ],
  ];
```

#webpack-dev-server配置
```javascript
const options = {
  contentBase: ['./dist'],  // 根目录
  hot: true, // 启用热替换
  host: 'localhost', // 服务器名称
};
```

#webpack配置
```javascript
const path = require('path');
const CleanWebpackPlugin = require('clean-webpack-plugin');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const webpack = require('webpack');
const presets = require('./babel.config').presets;
const OptimizeCSSAssetsPlugin = require('optimize-css-assets-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const UglifyJsPlugin = require('uglifyjs-webpack-plugin');

const mode = process.env.NODE_ENV || 'development';
const devMode = mode === 'development';

module.exports = {
  mode,
  entry: { 
    index: './src/index.js',
  },
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: '[name]-[hash:8].js' // 取生成的8位hash码，默认20位
  },
  devtool: 'inline-source-map',
  resolve: {
    alias: {
      '@src': path.resolve(__dirname, 'src')
    }
  },
  module: {
    rules: [
    /** eslint-loader 配置 **/
      {
        enforce: "pre", // 保证eslint-loader优先调用
        test: /\.jsx?$/,
        exclude: /node_modules/,
        loader: "eslint-loader"
      },
      {
        test: /\.css$/,
        use: [
            // production环境时，启用MiniCssExtractPlugin
          devMode ? 'style-loader' : MiniCssExtractPlugin.loader,  
          // { importLoaders: 1 } 表示css-loader之前有一个loader
          { loader: 'css-loader', options: { importLoaders: 1 } },
           // postcss-loader结合postcss-preset-env后，可以自动加css前缀,并且结合了polyfill
          { 
            loader: 'postcss-loader', 
            options: {
              // parser: 'sugarss',
              // exec: true,
              plugins:[require('postcss-preset-env')({  // 
                stage: 0, // 默认是2，对应babel的presets
                features: {
                  'nesting-rules': true // 启用stage配置
                }
              })]
          } }
        ]
      },
      {
        test: /\.(png|jpg|gif)$/,
        use: [
          {
            loader: 'url-loader',
            options: {
              limit: 8192 // 若超过8192字节，使用file-loader
            }
          }
        ],
      },
      {
        test: /\.(m?js|jsx)$/,
        exclude: /(node_modules|bower_components)/,
        use: {
          loader: 'babel-loader',
          options: {
            presets,
          }
        }
      }
    ]
  },
  optimization: {
    concatenateModules: !(mode === 'production'),
    minimizer: [
      new OptimizeCSSAssetsPlugin({}), // 分别将css和js打包成两类文件，若不使用，则打包成一类文件
      new UglifyJsPlugin({ // 压缩js或jsx代码
        test: /\.jsx?$/,
        cache: true,
        parallel: true,
        uglifyOptions: {
          output: {
            comments: false,
          },
          ie8: true,
        },
      }),
    ],
    splitChunks: {
      chunks: 'async',
      minSize: 51200, // 打包后，js代码文件的最小字节数
      maxSize: 1024 * 1024, // 打包后，js代码文件的最大字节数
      minChunks: 1, // 打包后至少产生一个js文件
      maxAsyncRequests: 5,
      maxInitialRequests: 3,
      automaticNameDelimiter: '~',
      name: true,
      cacheGroups: {
        vendors: {
          test: /[\\/]node_modules[\\/]/,
          priority: -10
        },
        default: {
          minChunks: 2,
          priority: -20,
          reuseExistingChunk: true
        }
      }
    }
  },
  plugins: [
    new webpack.ProgressPlugin(),
    new HtmlWebpackPlugin({template: './src/pages/index.html'}), // html模板插件，在html文件中自动引入打包后的js文件
    new MiniCssExtractPlugin({
      // Options similar to the same options in webpackOptions.output
      // both options are optional
      filename: devMode ? '[name].css' : '[name].[hash].css',
      chunkFilename: devMode ? '[id].css' : '[id].[hash].css',
    }),
    new webpack.HotModuleReplacementPlugin(), // 热替换插件
    new CleanWebpackPlugin(), // 每次重新编译，自动清除根目录的缓存文件
  ]
};
···