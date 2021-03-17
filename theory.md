## 过程
现将骨架屏代码放到<div id="app"></div>里，然后等到页面的其他资源来了之后，把里面的内容替换了而已

在骨架屏展示期间，会在<div id="app"></div>标签里有类似如下代码：

```js
// 其实就是根据路由匹配对应的骨架屏组件
var pathname = window.location.pathname;
var hash = window.location.hash;
var skeletons = [{
    id: 'skeleton',
    el: document.querySelector('#skeleton')
},{
    id: 'skeletonOne',
    el: document.querySelector('#skeletonOne')
}];
var isMatched = function(pathReg, mode) {
    if (mode === 'hash') {
        return pathReg.test(hash.replace('#', ''));
    }
    else if (mode === 'history') {
        return pathReg.test(pathname);
    }
    return false;
};
var showSkeleton = function(skeletonId) {
    for (var i = 0; i < skeletons.length; i++) {
        var skeleton = skeletons[i];
        if (skeletonId === skeleton.id) {
            skeleton.el.style = 'display:block;';
        }
        else {
            skeleton.el.style = 'display:none;';
        }
    }
};

if (isMatched(/^\/hotFoodList(?:\/)?$/i, 'hash')) {
    showSkeleton('skeleton');
}

else if (isMatched(/^\/reportMsg(?:\/)?$/i, 'hash')) {
    showSkeleton('skeletonOne');
}
```

1. 首先定义骨架屏的vue文件(骨架屏组件其实可以使用svg，图片等，但也可以手写页面)，以及入口文件，入口文件类似其他页面的main.js，因为webapck入口解析就是js
2. 在webpack的配置文件中引入vue-skeleton-webpack-plugin插件，并注册
3. 注册时传入关于骨架屏组件的webapck配置
4. VueSkeletonWebpackPlugin首先会将配置与默认配置合并
5. VueSkeletonWebpackPlugin会执行apply方法，
6. apply方法会执行generateSkeletonForEntries方法
7. generateSkeletonForEntries方法根据骨架屏文件入口地址列表，根据入库数量生成多个骨架屏的配置，然后分别调用('vue-server-renderer').createBundleRenderer方法，从webpack输出的boundle文件中解析出骨架屏源代码（这里都是新建childCompiler执行的）
8. 拿到所有入口的源代码后，在webpack的html-webpack-plugin-before-html-processing插件回调中，将骨架屏的源代码插入到项目主入口的源码中

在显示骨架屏阶段，在app.vue里会有类似如下脚本:

```js
var pathname = window.location.pathname;
var hash = window.location.hash;
var skeletons = [{
    id: 'skeleton',
    el: document.querySelector('#skeleton')
}];
var isMatched = function(pathReg, mode) {
    if (mode === 'hash') {
        return pathReg.test(hash.replace('#', ''));
    }
    else if (mode === 'history') {
        return pathReg.test(pathname);
    }
    return false;
};
var showSkeleton = function(skeletonId) {
    for (var i = 0; i < skeletons.length; i++) {
        var skeleton = skeletons[i];
        if (skeletonId === skeleton.id) {
            skeleton.el.style = 'display:block;';
        }
        else {
            skeleton.el.style = 'display:none;';
        }
    }
};

if (isMatched(/^\/hotFoodList(?:\/)?$/i, 'hash')) {
    showSkeleton('skeleton');
}
```


```js
// 步骤一：webpack的主入口
'use strict'
const utils = require('./utils')
const webpack = require('webpack')
const config = require('../config')
const merge = require('webpack-merge')
const path = require('path')
const baseWebpackConfig = require('./webpack.base.conf')
const CopyWebpackPlugin = require('copy-webpack-plugin')
const HtmlWebpackPlugin = require('html-webpack-plugin')
const FriendlyErrorsPlugin = require('friendly-errors-webpack-plugin')
const ExtractTextPlugin = require('extract-text-webpack-plugin')
const SkeletonWebpackPlugin = require('vue-skeleton-webpack-plugin')
const portfinder = require('portfinder')

const HOST = process.env.HOST
const PORT = process.env.PORT && Number(process.env.PORT)

const devWebpackConfig = merge(baseWebpackConfig, {
  module: {
    rules: utils.styleLoaders({ sourceMap: config.dev.cssSourceMap, usePostCSS: true })
  },
  // cheap-module-eval-source-map is faster for development
  devtool: config.dev.devtool,

  // these devServer options should be customized in /config/index.js
  devServer: {
    clientLogLevel: 'warning',
    historyApiFallback: {
      rewrites: [
        { from: /.*/, to: path.posix.join(config.dev.assetsPublicPath, 'index.html') },
      ],
    },
    hot: true,
    contentBase: false, // since we use CopyWebpackPlugin.
    compress: true,
    host: HOST || config.dev.host,
    port: PORT || config.dev.port,
    open: config.dev.autoOpenBrowser,
    overlay: config.dev.errorOverlay
      ? { warnings: false, errors: true }
      : false,
    publicPath: config.dev.assetsPublicPath,
    proxy: config.dev.proxyTable,
    quiet: true, // necessary for FriendlyErrorsPlugin
    watchOptions: {
      poll: config.dev.poll,
    }
  },
  plugins: [
    new webpack.DefinePlugin({
      'process.env': require('../config/dev.env')
    }),
    new webpack.HotModuleReplacementPlugin(),
    new webpack.NamedModulesPlugin(), // HMR shows correct file names in console on update.
    new webpack.NoEmitOnErrorsPlugin(),
    // https://github.com/ampedandwired/html-webpack-plugin
    new HtmlWebpackPlugin({
      filename: 'index.html',
      template: 'index.html',
      inject: true
    }),
    // copy custom static assets
    new CopyWebpackPlugin([
      {
        from: path.resolve(__dirname, '../static'),
        to: config.dev.assetsSubDirectory,
        ignore: ['.*']
      }
    ]),
    new ExtractTextPlugin({
      filename: utils.assetsPath('css/[name].[contenthash].css'),
      // Setting the following option to `false` will not extract CSS from codesplit chunks.
      // Their CSS will instead be inserted dynamically with style-loader when the codesplit chunk has been loaded by webpack.
      // It's currently set to `true` because we are seeing that sourcemaps are included in the codesplit bundle as well when it's `false`,
      // increasing file size: https://github.com/vuejs-templates/webpack/issues/1110
      allChunks: true,
    }),
    // inject skeleton content(DOM & CSS) into HTML
    new SkeletonWebpackPlugin({
      webpackConfig: require('./webpack.skeleton.conf'),
      quiet: true
    }),
  ]
})

module.exports = new Promise((resolve, reject) => {
  portfinder.basePort = process.env.PORT || config.dev.port
  portfinder.getPort((err, port) => {
    if (err) {
      reject(err)
    } else {
      // publish the new Port, necessary for e2e tests
      process.env.PORT = port
      // add port to devServer config
      devWebpackConfig.devServer.port = port

      // Add FriendlyErrorsPlugin
      devWebpackConfig.plugins.push(new FriendlyErrorsPlugin({
        compilationSuccessInfo: {
          messages: [`Your application is running here: http://${devWebpackConfig.devServer.host}:${port}`],
        },
        onErrors: config.dev.notifyOnErrors
        ? utils.createNotifierCallback()
        : undefined
      }))

      resolve(devWebpackConfig)
    }
  })
})


// 步骤二：骨架屏的配置文件
'use strict';

const path = require('path')
const merge = require('webpack-merge')
const baseWebpackConfig = require('./webpack.base.conf')
const nodeExternals = require('webpack-node-externals')

function resolve(dir) {
  return path.join(__dirname, dir)
}

module.exports = merge(baseWebpackConfig, {
  target: 'node',
  devtool: false,
  entry: {
    // 路径是绝对路径
    app: resolve('../src/entry-skeleton.js')
  },
  // entry: resolve('../src/entry-skeleton.js'), // 这种格式不行
  output: Object.assign({}, baseWebpackConfig.output, {
    libraryTarget: 'commonjs2'
  }),
  externals: nodeExternals({
    whitelist: /\.css$/
  }),
  plugins: []
})

// 步骤三：SkeletonWebpackPlugin插件的主入口
/**
 * @file generate skeleton
 * @author panyuqi <panyuqi@baidu.com>
 */

/* eslint-disable no-console, fecs-no-require */

var ssr = require('./ssr');
var ref = require('./util');
var insertAt = ref.insertAt;
var isObject = ref.isObject;
var isFunction = ref.isFunction;
var generateRouterScript = ref.generateRouterScript;


// webpackConfig 必填，渲染 skeleton 的 webpack 配置对象
// insertAfter 选填，渲染 DOM 结果插入位置，默认值为字符串 '<div id="app">'
// 也可以传入 Function，方法签名为 insertAfter(entryKey: string): string，返回值为挂载点字符串
// quiet 选填，在服务端渲染时是否需要输出信息到控制台
// router 选填 SPA 下配置各个路由路径对应的 Skeleton
//  mode 选填 路由模式，两个有效值 history|hash
//  routes 选填 路由数组，其中每个路由对象包含两个属性：
//      path 路由路径 string|RegExp
//      skeletonId Skeleton DOM 的 id string
// minimize 选填 SPA 下是否需要压缩注入 HTML 的 JS 代码
var DEFAULT_PLUGIN_OPTIONS = {
    webpackConfig: {},
    insertAfter: '<div id="app">',
    quiet: false
};

var DEFAULT_ENTRY_NAME = 'main';

// 这个名字怎么和安装组件的名字联系起来？
var PLUGIN_NAME = 'VueSkeletonWebpackPlugin';

var SkeletonPlugin = function SkeletonPlugin(options) {
    if ( options === void 0 ) options = {};

    this.options = Object.assign({}, DEFAULT_PLUGIN_OPTIONS, options);
    // console.log('options', this.options);
    // 这里在其他配置的基础上增加了skeleton的一些配置而已，但切记是后者覆盖前者，所以入口文件也变为了skeleton.js了
    // entry: {
    //     app: '/Users/dujichong/workDir/source-code/skeleton-demo/src/entry-skeleton.js'
    // },
};

SkeletonPlugin.prototype.apply = function apply (compiler) {
    // console.log('1 apply compiler.options', compiler.options)
    // compiler.options是整个编译的入口

    var this$1 = this;

    var skeletons;
    // compatible with webpack 4.x
    if (compiler.hooks) {
        compiler.hooks.make.tapAsync(PLUGIN_NAME, function (compilation, cb) {
            if (!compilation.hooks.htmlWebpackPluginBeforeHtmlProcessing) {
                console.error('VueSkeletonWebpackPlugin must be placed after HtmlWebpackPlugin in `plugins`.');
                return;
            }

            this$1.generateSkeletonForEntries(this$1.extractEntries(compiler.options.entry), compiler, compilation)
                .then(function (skeletonResults) {
                    skeletons = skeletonResults.reduce(function (cur, prev) { return Object.assign(prev, cur); }, {});
                    console.log('skeletons reduce', skeletons);
                    cb();
                })
                .catch(function (e) { return console.log(e); });

            compilation.hooks.htmlWebpackPluginBeforeHtmlProcessing.tapAsync(PLUGIN_NAME, function (htmlPluginData, callback) {
                this$1.injectToHtml(htmlPluginData, skeletons);
                callback(null, htmlPluginData);
            });
        });
    }
    else {
        compiler.plugin('make', function (compilation, cb) {
            // 根据入口文件产生骨架屏
            this$1.generateSkeletonForEntries(this$1.extractEntries(compiler.options.entry), compiler, compilation)
                .then(function (skeletonResults) {
                    skeletons = skeletonResults.reduce(function (cur, prev) { return Object.assign(prev, cur); }, {});
                    cb();
                })
                .catch(function (e) { return console.log(e); });
                
            // 上面拿到骨架屏的资源，然后这里结合主项目的htmlPluginData
            compilation.plugin('html-webpack-plugin-before-html-processing', function (htmlPluginData, callback) {
                console.log('htmlPluginData', htmlPluginData,  skeletons)
                // console.log('skeletons htmlPluginData', skeletons, htmlPluginData);
                // 骨架屏的资源, skeletons
                // {
                //     app: {
                //         html: 'xxx',
                //         css: 'xxx',
                //         script: 'xx',
                //     }
                // }

                // 此时的htmlPluginData是一个对象
                // {
                //     html: 'xxx',
                //     assets: {},
                //     plugins: 
                // }
                this$1.injectToHtml(htmlPluginData, skeletons);
                callback(null, htmlPluginData);
            });
        });
    }
};

/**
 * format entries for all skeletons from options
 * 函数的目的就是拿到骨架屏的入口地址
 * @param {Object} parentEntry entry in webpack.config
 * @return {Object} entries entries for all skeletons
 */
SkeletonPlugin.prototype.extractEntries = function extractEntries (parentEntry) {
    // console.log('compiler.options.entry parentEntry', parentEntry)
    // 其实在给webpackConfig.entry传一个字符串，到这里也会变成一个对象，只是对象被一个个字符分开了
    var entry = Object.assign({}, this.options.webpackConfig.entry);
    // entry是骨架屏的入口， parentEntry是项目的入口
    // console.log('entry', parentEntry, entry)
    // {
    //     app: [
    //       '/Users/dujichong/workDir/source-code/skeleton-demo/node_modules/webpack-dev-server/client/index.js?http://localhost:8080',
    //       'webpack/hot/dev-server',
    //       './src/main.js'
    //     ]
    // } 
    // {
    // app: '/Users/dujichong/workDir/source-code/skeleton-demo/src/entry-skeleton.js'
    // }

    var skeletonEntries;

    if (isObject(entry)) {
        skeletonEntries = entry;
    }
    else {
        var entryName = DEFAULT_ENTRY_NAME;

        if (isObject(parentEntry)) {
            entryName = Object.keys(parentEntry)[0];
        }
        skeletonEntries = {};
        skeletonEntries[entryName] = entry;
    }
   
    return skeletonEntries;
};

/**
 * find skeleton for current html-plugin in all skeletons
 * 
 * @param {Object} htmlPluginData data for html-plugin
 * @param {Object} skeletons skeletons
 * @param {Object} target skeleton
 */
SkeletonPlugin.prototype.findSkeleton = function findSkeleton (htmlPluginData, skeletons) {
    // console.log('findSkeleton', htmlPluginData);
    if ( skeletons === void 0 ) skeletons = {};

    var usedChunks = Object.keys(htmlPluginData.assets.chunks);
    var entryKey;

    // find current processing entry
    if (Array.isArray(usedChunks)) {
        entryKey = Object.keys(skeletons).find(function (v) { return usedChunks.indexOf(v) > -1; });
    }
    else {
        entryKey = DEFAULT_ENTRY_NAME;
    }

    return {
        name: entryKey,
        skeleton: skeletons[entryKey]
    };
};

/**
 * inject HTML, CSS and JS
 * 
 * @param {Object} htmlPluginData data for html-plugin
 * @param {Object} skeletons skeletons
 */
SkeletonPlugin.prototype.injectToHtml = function injectToHtml (htmlPluginData, skeletons) {
    console.log('injectToHtml');
        if ( skeletons === void 0 ) skeletons = {};

    var ref = this.options;
        var insertAfter = ref.insertAfter;

    var ref$1 = this.findSkeleton(htmlPluginData, skeletons);
        var name = ref$1.name;
        var skeleton = ref$1.skeleton;
    if (!skeleton) {
        console.log('Empty entry for skeleton, please check your webpack.config.');
        return;
    }
    var html = skeleton.html; if ( html === void 0 ) html = '';
        var css = skeleton.css; if ( css === void 0 ) css = '';
        var script = skeleton.script; if ( script === void 0 ) script = '';
        
    // insert inlined styles into html
    var headTagEndPos = htmlPluginData.html.lastIndexOf('</head>');
    htmlPluginData.html = insertAt(htmlPluginData.html, ("<style>" + css + "</style>"), headTagEndPos);

    // replace mounted point with ssr result in html
    if (isFunction(insertAfter)) {
        insertAfter = insertAfter(name);
    }

    // html是字符串，insertAfter是一个插入的标识
    // console.log('insertAfter', insertAfter);
    // insertAfter <div id="app">
    // console.log('htmlPluginData.html', htmlPluginData.html);
    var appPos = htmlPluginData.html.lastIndexOf(insertAfter) + insertAfter.length;
    htmlPluginData.html = insertAt(htmlPluginData.html, html + script, appPos);
    // 将骨架屏的代码插入到页面中了
    // console.log('htmlPluginData.html', htmlPluginData.html);
};

/**
 * generate skeletons for all entries
 * 
 * @param {Object} entries entries for all skeletons
 * @param {Object} compiler compiler
 * @param {Object} compilation compilation
 * @return {Promise} promise
 */
SkeletonPlugin.prototype.generateSkeletonForEntries = function generateSkeletonForEntries (entries, compiler, compilation) {
    console.log('generateSkeletonForEntries');
    var this$1 = this;

    var ref = this.options;
        var router = ref.router;
        var minimize = ref.minimize;
        var quiet = ref.quiet;

    // console.log('ref', ref);
    // {
    //     webpackConfig: {
    //         context: '/Users/dujichong/workDir/source-code/skeleton-demo',
    //         entry: {
    //         app: '/Users/dujichong/workDir/source-code/skeleton-demo/src/entry-skeleton.js'
    //         },
    //         output: {
    //         path: '/Users/dujichong/workDir/source-code/skeleton-demo/docs',
    //         filename: 'skeleton-app.js',
    //         publicPath: '/',
    //         libraryTarget: 'commonjs2'
    //         },
    //         resolve: { extensions: [Array], alias: [Object] },
    //         module: { rules: [Array] },
    //         node: {
    //         setImmediate: false,
    //         dgram: 'empty',
    //         fs: 'empty',
    //         net: 'empty',
    //         tls: 'empty',
    //         child_process: 'empty'
    //         },
    //         target: 'node',
    //         devtool: false,
    //         externals: [Function (anonymous)],
    //         plugins: []
    //     },
    //     insertAfter: '<div id="app">',
    //     quiet: true
    // } 

        // 遍历entries
    return Promise.all(Object.keys(entries).map(function (entryKey) {
        var skeletonWebpackConfig = Object.assign({}, this$1.options.webpackConfig);

        // 其实是根据每个入口，然后生成多个配置，相当于每个入口都有一个配置文件
        // set current entry & output in webpack config
        skeletonWebpackConfig.entry = entries[entryKey];
        if (!skeletonWebpackConfig.output) {
            skeletonWebpackConfig.output = {};
        }
        skeletonWebpackConfig.output.filename = "skeleton-" + entryKey + ".js";

        // inject router code in SPA mode
        var routerScript = '';
        // spa模式下，可以设置多个骨架屏组件？
        if (router) {
            // 只要有多个入口，就是MPA
            var isMPA = !!(Object.keys(entries).length > 1);
            routerScript = generateRouterScript(router, minimize, isMPA, entryKey);
        }

        // server side render skeleton for current entry
        return ssr(skeletonWebpackConfig, {
            quiet: quiet, compilation: compilation, context: compiler.context
        }).then(function (ref) {
            // 拿到vue-server-renderer解析webpack输出的boundle后，渲染出来的骨架屏文件
                var skeletonHTML = ref.skeletonHTML;
                var skeletonCSS = ref.skeletonCSS;

            return ( obj = {}, obj[entryKey] = {
                    html: skeletonHTML,
                    css: skeletonCSS,
                    script: routerScript
                }, obj );
                var obj;
        });
    }));
};

SkeletonPlugin.loader = function loader (ruleOptions) {
    console.log('loader');
        if ( ruleOptions === void 0 ) ruleOptions = {};

    console.log('[DEPRECATED] SkeletonPlugin.loader is DEPRECATED now. Hot reload in dev mode is supported already, so you can remove this option.');
    return Object.assign(ruleOptions, {
        loader: require.resolve('./loader'),
        options: Object.assign({}, ruleOptions.options)
    });
};

module.exports = SkeletonPlugin;

// 步骤四：骨架屏通过webpack构建后，需要从boundle里反解析出来，这里需要用到vue-server-render
/**
 * @file ssr
 * @desc Use vue ssr to render skeleton components. The result contains html and css.
 * @author panyuqi <panyuqi@baidu.com>
 */

/* eslint-disable no-console, fecs-no-require */

var path = require('path');
var webpack = require('webpack');
var webpackMajorVersion = require('webpack/package.json').version.split('.')[0];
var NodeTemplatePlugin = require('webpack/lib/node/NodeTemplatePlugin');
var NodeTargetPlugin = require('webpack/lib/node/NodeTargetPlugin');
var LoaderTargetPlugin = require('webpack/lib/LoaderTargetPlugin');
var LibraryTemplatePlugin = require('webpack/lib/LibraryTemplatePlugin');
var SingleEntryPlugin = require('webpack/lib/SingleEntryPlugin');
var MultiEntryPlugin = require('webpack/lib/MultiEntryPlugin');
var ExternalsPlugin = require('webpack/lib/ExternalsPlugin');

var createBundleRenderer = require('vue-server-renderer').createBundleRenderer;
var nodeExternals = require('webpack-node-externals');

var MiniCssExtractPlugin;
var ExtractTextPlugin;
if (webpackMajorVersion === '4') {
    MiniCssExtractPlugin = require('mini-css-extract-plugin');
}
else {
    ExtractTextPlugin = require('extract-text-webpack-plugin');
}

module.exports = function renderSkeleton (serverWebpackConfig, ref) {
    var quiet = ref.quiet; if ( quiet === void 0 ) quiet = false;
    var compilation = ref.compilation;
    var context = ref.context;

    var ref$1 = compilation.outputOptions;
    var outputPath = ref$1.path;
    var outputPublicPath = ref$1.publicPath;

    // get entry name from webpack.conf
    // console.log('outputPath', outputPath, serverWebpackConfig.output);
    // /Users/dujichong/workDir/source-code/skeleton-demo/docs 
    // {
    //     path: '/Users/dujichong/workDir/source-code/skeleton-demo/docs',
    //     filename: 'skeleton-app.js',
    //     publicPath: '/',
    //     libraryTarget: 'commonjs2'
    // }

    // 生成文件的最后绝对路径
    var outputJSPath = path.join(outputPath, serverWebpackConfig.output.filename);
    var outputBasename = path.basename(outputJSPath, path.extname(outputJSPath));
    // console.log('path.extname(outputJSPath)', path.extname(outputJSPath), outputJSPath, outputBasename)
    // js /Users/dujichong/workDir/source-code/skeleton-demo/docs/skeleton-app.js skeleton-app
    var outputCssBasename = outputBasename + ".css";
    var outputCSSPath = path.join(outputPath, outputCssBasename);

    // guiet只是控制一些打印日志
    if (!quiet) {
        console.log(("Generate skeleton for " + outputBasename + "..."));
    }

    var originalRules;
    if (webpackMajorVersion !== '4') {
        // if user passed in some special module rules for Skeleton, use it directly
        originalRules = compilation.options.module.rules;
        
        // 可以直接只用用户定义的规则，二者只会选择其一
        if (serverWebpackConfig.module && serverWebpackConfig.module.rules) {
            compilation.options.module.rules = serverWebpackConfig.module.rules;
        }
        else {
            // otherwise use rules from parent compiler
            var vueRule = compilation.options.module.rules.find(function (rule) {
                return rule.test && rule.test.test && rule.test.test('test.vue');
            });

            if (vueRule && vueRule.use && vueRule.use.length) {
                var vueLoader = vueRule.use.find(function (rule) {
                    return rule.loader = 'vue-loader';
                });

                vueLoader.options.extractCSS = true;
                // 删除有什么用？
                delete vueLoader.options.loaders;
            }
        }
    }

    var outputOptions = {
        filename: outputJSPath,
        publicPath: outputPublicPath
    };

    // 创建子编译进程
    var childCompiler = compilation.createChildCompiler('vue-skeleton-webpack-plugin-compiler', outputOptions);

    // 指定上下文
    childCompiler.context = context;
    new LibraryTemplatePlugin(undefined, 'commonjs2').apply(childCompiler);
    new NodeTargetPlugin().apply(childCompiler);
    if (Array.isArray(serverWebpackConfig.entry)) {
        new MultiEntryPlugin(context, serverWebpackConfig.entry, undefined).apply(childCompiler);
    }
    else {
        new SingleEntryPlugin(context, serverWebpackConfig.entry, undefined).apply(childCompiler);
    }
    new LoaderTargetPlugin('node').apply(childCompiler);
    new ExternalsPlugin('commonjs2', serverWebpackConfig.externals || nodeExternals({
        whitelist: /\.css$/
    })).apply(childCompiler);
    if (webpackMajorVersion === '4') {
        new MiniCssExtractPlugin({
            filename: outputCSSPath
        }).apply(childCompiler);
    }
    else {
        new ExtractTextPlugin({
            filename: outputCSSPath
        }).apply(childCompiler);
    }

    return new Promise(function (resolve, reject) {
        childCompiler.runAsChild(function (err, entries, childCompilation) {
            if (childCompilation && childCompilation.errors && childCompilation.errors.length) {
                var errorDetails = childCompilation.errors.map(function (error) { return error.message + (error.error ? ':\n' + error.error : ''); }).join('\n');
                reject(new Error('Child compilation failed:\n' + errorDetails));
            }
            else if (err) {
                reject(err);
            }
            else {
                var bundle = childCompilation.assets[outputJSPath].source();
                // 拿到webpack构建出来的bundle文件，而且骨架屏的相关文件
                // console.log('bundle', bundle);

                // 分离css了
                var skeletonCSS = '';
                if (childCompilation.assets[outputCSSPath]) {
                    skeletonCSS = childCompilation.assets[outputCSSPath].source();
                }

                // delete JS & CSS files
                delete compilation.assets[outputJSPath];
                delete compilation.assets[outputCSSPath];
                delete compilation.assets[(outputJSPath + ".map")];
                delete compilation.assets[(outputCSSPath + ".map")];
                // create renderer with bundle
                var renderer = createBundleRenderer(bundle);

                // use vue ssr to render skeleton
                renderer.renderToString({}, function (err, skeletonHTML) {
                    if (err) {
                        reject(err);
                    }
                    else {
                        if (webpackMajorVersion !== '4') {
                            compilation.options.module.rules = originalRules;
                        }
                        // console.log('skeletonHTML', skeletonHTML, skeletonCSS);
                        // 这里就是渲染出来的骨架屏html和css
                        resolve({skeletonHTML: skeletonHTML, skeletonCSS: skeletonCSS});
                    }
                });
            }
        });
    });
};

```