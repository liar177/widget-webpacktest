# widget-webpacktest
组件低代码化下一种拼装部件的打包思路
// common
const webpack = require("webpack");
const path = require("path");
const { CleanWebpackPlugin } = require("clean-webpack-plugin");
const VueLoaderPlugin = require("vue-loader/lib/plugin");

module.exports = {
  module: {
    rules: [
      {
        test: /\.scss$/,
        use: ["style-loader", "css-loader", "sass-loader"],
      },
      {
        test: /\.less$/,
        use: ["style-loader", "css-loader", "less-loader"],
      },
      {
        test: /\.css$/,
        use: [
          "style-loader",
          "css-loader",
          {
            loader: "postcss-loader",
            options: {
              postcssOptions: {
                plugins: ["tailwindcss", "autoprefixer"],
              },
            },
          },
        ],
      },
      {
        test: /\.(png|jpg|jpeg|gif|svg)$/,
        use: [
          {
            loader: "url-loader",
            options: {
              esModule: false,
            },
          },
        ],
        exclude: path.resolve(__dirname, "../", "src/widgets"),
      },
      {
        test: /\.(png|jpg|jpeg|gif|svg)$/,
        use: [
          {
            loader: "file-loader",
            options: {
              name: "[name].[ext]",
              esModule: false,
              outputPath: (url, resourcePath, context) => {
                const pathArr = resourcePath.split(/\/|\\/);
                const widgetIndex = pathArr.findIndex((p) => p === "widgets");
                if (widgetIndex > 0) {
                  return `${pathArr[widgetIndex + 1]}/public/img/${url}`;
                }
              },
            },
          },
        ],
        include: path.resolve(__dirname, "../", "src/widgets"),
      },
      {
        test: /\.js$/,
        // exclude: /node_modules/,
        exclude: (filepath) => {
          const reg = /@hui-pro|wad-component|wad-map|gaismap|@npm-bbg|js-plugin-pro|widget-dev-utils|webVideoPlayer|vue-i18n/;
          if (reg.test(filepath)) {
            return false;
          }
          return /node_modules/.test(filepath);
        },
        use: [
          {
            loader: "babel-loader",
            options: {
              rootMode: "upward",
            },
          },
        ],
      },
      {
        test: /\.vue$/,
        use: ["vue-loader"],
      },
      {
        test: /\.(woff2?|eot|ttf|otf|webm|mp4|mov)(\?.*)?$/,
        use: [
          {
            loader: "url-loader",
            options: {
              //   limit: 0,
              name: "[name].[ext]",
            },
          },
        ],
      },
      {
        type: "javascript/auto", // https://github.com/webpack-contrib/file-loader/issues/324
        test: /locale.+\.json$/,
        include: path.resolve(__dirname, "../", "src/widgets"),
        use: [
          {
            loader: "file-loader",
            options: {
              name: "[name].[ext]",
              useRelativePath: true,
              esModule: false,
              outputPath: (url, resourcePath, context) => {
                const reg = /locale\\(.+)\\index\.json$/g;
                const lang = resourcePath.match(reg);
                const _l = lang[0].split("\\")[1];
                const pathArr = resourcePath.split(/\/|\\/);
                const widgetIndex = pathArr.findIndex((p) => p === "widgets");
                if (widgetIndex > 0) {
                  return `${
                    pathArr[widgetIndex + 1]
                  }/public/locale/${_l}/${url}`;
                }
              },
            },
          },
        ],
      },
    ],
  },
  plugins: [
    new CleanWebpackPlugin(),
    new VueLoaderPlugin(),
    new webpack.ProvidePlugin({
      $: "jquery",
      jquery: "jquery",
      jQuery: "jquery",
      "window.jQuery": "jquery",
    }),
  ],
  resolve: {
    extensions: [".js", ".jsx", ".json", ".vue"],
    alias: {
      vue$: "vue/dist/vue.esm.js",
      "@": path.resolve(__dirname, "../", "src"),
      "@@": path.resolve(__dirname, "../", "src/widget-tools"),
      webVideoPlayer: path.resolve(
        __dirname,
        "../",
        "node_modules/vue-cli-plugin-vc/js/public/videoPlayer/webVideoPlayer.js"
      ),
    },
  },
};


// prod
const webpack = require('webpack')
const path = require('path')
const common = require('./webpack.common.js')
const { merge } = require('webpack-merge')
const TerserPlugin = require('terser-webpack-plugin')
const CopyWebpackPlugin = require('copy-webpack-plugin')
const { entriesGet, copyPatternsGet } = require('./util')
const BuildAddZipPlugin = require('./webpack-custom-plugin/build-add-zip')
const { widgetPathArr } = require('./constant')

let batchArr = process.argv[4] && process.argv[4].split('--')[1].split(',')

// 增加关键字parking 用来批量打包停车场部件
if(batchArr&&batchArr.includes('parking')){
  batchArr = ['alarm-pop','alarm-number','alarm-event','cars-number','dangerous-statistics','header-statistic','parking-space','regionSwitcher','today-data']
}

module.exports = merge(common, {
  mode: 'production',
  entry: entriesGet(widgetPathArr, batchArr),
  plugins: [
    new CopyWebpackPlugin({
      patterns: copyPatternsGet(widgetPathArr, ['package.json', 'preview.png', 'HISTORY.md', 'public'], batchArr)
    }),
    new webpack.DefinePlugin({
      ISDEV: JSON.stringify(false),
      GAISCONTEXT: JSON.stringify('/gais')
    }),
    new BuildAddZipPlugin() // 打包后自动生成zip包, 可以删除, 不影响打包

  ],
  externals: {
    echarts: 'echarts',
    echarts5:'echarts5',
    hui: 'hui',
    vue: 'vue',
    huiPro: 'hui-pro',
    moment:'moment',
    lodash:'lodash'
  },
  optimization: {
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: false
        }
      })
    ]
  },
  output: {
    filename: '[name]/index.js',
    path: path.resolve(__dirname, '../', 'dist/')
  }
})

//dev
const path = require("path");
const common = require("./webpack.common.js");
const { merge } = require("webpack-merge");
const webpack = require("webpack");
const HtmlWebpackPlugin = require("html-webpack-plugin");
const { getWidgetMoudles } = require("./util");
const { widgetPathArr } = require("./constant");

module.exports = merge(common, {
  mode: "development",
  entry: {
    index: path.join(__dirname, "../src/", "main.js"),
  },
  devtool: "source-map",
  devServer: {
    contentBase: "./dist/",
    open: true,
    hot: true,
    host: "0.0.0.0",
    useLocalIp: true,
    proxy: {
      '/ptsim': { // 代理园区驾驶舱, 用于透传
        target: 'http://localhost:31650',
        changeOrigin: true
      },
      '/pemc': { // 代理园区驾驶舱, 用于透传
        target: 'http://localhost:31650',
        changeOrigin: true
      },
      '/copas': { // 代理园区驾驶舱, 用于透传
        target: 'http://localhost:31650',
        changeOrigin: true
      },
      //本地开发代理
      "/local": {
        target: "http://localhost:31650", // 设置开发环境下接口的代理地址
        // target: 'http://localhost:37799',
        //target: 'http://10.10.84.15:8088/',
        changeOrigin: true,
        pathRewrite: {
          "/local": "",
        },
      },
      '/gais': { // 代理gais的图片
        target: 'http://localhost:31650',
        changeOrigin: true
      },
      "/mock": {
        target: "http://hapi.hikvision.com.cn",
        changeOrigin: true,
      },
      "/GeoData_gais": {
        // 代理离线地图
        target: "http://localhost:31650",
        changeOrigin: true,
      },
      "/dve": {
        // 代理三维资源
        target: "http://localhost:56203",
        changeOrigin: true,
      },
      "/sops": {
        // 代理三维资源
        target: "http://localhost:31650",
        changeOrigin: true,
      },
    },
  },
  plugins: [
    new webpack.HotModuleReplacementPlugin(),
    new HtmlWebpackPlugin({
      template: "index.html",
      filename: "index.html",
    }),
    new webpack.DefinePlugin({
      ISDEV: JSON.stringify(true),
      GAISCONTEXT: JSON.stringify("https://10.19.134.167/gais"),
      __WIDGET_MODULES__: JSON.stringify(
        getWidgetMoudles(widgetPathArr).map((item) => item.widgetName)
      ),
    }),
  ],
});
