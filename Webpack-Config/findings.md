### Sample Config

This was the webpack production config I built for ERate.
It uses the inject hash plugin I built which I had to rebuild for the optimizations in this config.



```js
const path = require('path')
const webpack = require('webpack');
const MiniCSSExtractPlugin = require('mini-css-extract-plugin');
const MPAInjectHashPlugin = require('mpa-inject-hash-webpack-plugin');

/**Production Bundling plugins */
const { CleanWebpackPlugin } = require('clean-webpack-plugin');
const OptimizeCssAssetsPlugin = require('optimize-css-assets-webpack-plugin');
const TerserPlugin = require('terser-webpack-plugin');

/* Export config depending on argv specified in npm scripts */
module.exports = (env, argv) => {
  const isProd = argv.mode === `production`;

  /** Common Config defaults to development mode. Prod specs defined in export at bottom **/
  const config = {
    mode: isProd ? 'production' : 'development',
    devtool: isProd ? 'cheap-source-map' : 'eval-source-map',
    entry: {
      app: path.join(__dirname, 'client/reactClient/src/index.js'),
      login: path.join(__dirname, 'client/reactClient/src/pages/login/index.js')
    },
    output: {
      filename: isProd ? '[name].bundle.[contentHash].js' : '[name].bundle.js',
      path: path.join(__dirname, 'dist'),
      // needed to specify for dynamically loading chunks (lazy loading)
      publicPath: '/dist/'
    },
    // greater terminal output for production that dev
    stats: { logging : isProd ? "info" : "warn" },
    module: {
      rules: [
        {
          test: /\.js$/,
          exclude: /node_modules/,
          use: {
            loader: 'babel-loader',
            options: {
              presets: [
                [
                '@babel/preset-react',
                  {
                    // https://babeljs.io/docs/en/babel-preset-env#browserslist-integration
                    // TLDR - auto imports polyfills depending on your browser targets set 
                    // by the browserslist query in package.json
                    useBuiltIns: "usage",
                    corejs: {
                      version: 3,
                      proposals: true
                    }
                  }
                ]
              ],
            },
          },
        },
        {
          test: /\.(png|jpe?g|gif)$/i,
          loader: `url-loader`,
          options: {
            // size limit of asset being loaded in. 
            limit: 10000,
            // Any image attempted to load which is larger will automatically
            // default to being loaded with file-loader. If that happens, then 
            // this name property applies to that new file for the large asset. 
            name: '[name].[ext]'
          }
        },
        {
          test: /\.(s?)css$/i,
          // remember: loaders are processed in reverse order.
          // 1. sass-loader --> 2. css-loader, --> 3. MiniCSSExtractPlugin.loader
          use: [MiniCSSExtractPlugin.loader, 'css-loader', 'sass-loader'],
        },
        {
          test: /\.html$/, 
          loader: 'html-loader',
          options: { minimize: isProd }
        }
      ],
    },
    plugins : [
      new MiniCSSExtractPlugin({ filename: isProd ? "[name].bundle.[contentHash].css" : "[name].bundle.css" }),
      // https://github.com/pckessel/MPAInjectHashWebpackPlugin
      new MPAInjectHashPlugin({
        targets: {
          app: { path: path.join(__dirname, 'main.aspx') },
          login:{ path: path.join(__dirname, 'login.aspx') }
        }
      }),
    ],
    watchOptions: {
      ignored: /node_modules/
    }
  };

  const productionOptimizations = {
  /* The docs explain these optimizations better than I can. Here is
  * where to start:
  * https://webpack.js.org/plugins/terser-webpack-plugin/ 
  * https://www.npmjs.com/package/optimize-css-assets-webpack-plugin 
  * https://webpack.js.org/guides/caching/#extracting-boilerplate 
  * https://webpack.js.org/plugins/split-chunks-plugin/#split-chunks-example-2 
  * Also, this is a pretty funny article but I got many clues for optimization
  * from him and its also a great resource for benchmarking the optimizations: 
  * https://medium.com/hackernoon/the-100-correct-way-to-split-your-chunks-with-webpack-f8a9df5b7758 */
    minimize: true,
    minimizer: [
      // we can't mangle output due to angular's dependencies. They expect strings but shorthand syntax allows
      // for dependencies to not be strings. Mangling the short hand syntax breaks angular with unknows provider errors.
      new TerserPlugin({ terserOptions: { mangle: false } }),
      new OptimizeCssAssetsPlugin({})
    ],
    moduleIds: 'hashed',
    runtimeChunk: 'single',
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendors: {
          test: /[\\/]node_modules[\\/]/,
          name(module) {
            // get the name. E.g. node_modules/packageName/not/this/part.js
            // or node_modules/packageName
            const packageName = module.context.match(/[\\/]node_modules[\\/](.*?)([\\/]|$)/)[1];

            // npm package names are URL-safe, but some servers don't like @ symbols
            return `vendor.${packageName.replace('@', '')}`;
          },
        }
      }
    },
  };

  if(isProd) {
    // clean the output dir prior to building production files.
    config.plugins.push(new CleanWebpackPlugin());

    // So that file hashes dont unexpectedly change which would bust the browser cached modules.
    config.plugins.push(new webpack.HashedModuleIdsPlugin());

    config.optimization = productionOptimizations;
  };

  return config;
};
```