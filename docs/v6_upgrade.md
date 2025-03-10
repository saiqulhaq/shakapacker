# Upgrading from Webpacker v5 to Shakapacker v6

There are several substantial changes in Shakapacker v6 that you need to manually account for when coming from Webpacker 5. This guide will help you through it.

## Webpacker/Shakapacker has become a slimmer wrapper around Webpack

By default, Webpacker 6 is focused on compiling and bundling JavaScript. This pairs with the existing asset pipeline in Rails that's setup to transpile CSS and static images using [Sprockets](https://github.com/rails/sprockets). For most developers, that's the recommended combination. But if you'd like to use Webpacker for CSS and static assets as well, please see [integrations](https://github.com/shakacode/shakapacker#integrations) for more information.

Webpacker used to configure Webpack indirectly, which lead to a [complicated secondary configuration process](https://github.com/rails/webpacker/blob/5-x-stable/docs/webpack.md). This was done in order to provide default configurations for the most popular frameworks, but ended up creating more complexity than it cured. So now Webpacker delegates all configuration directly to Webpack's default configuration setup. Additionally, all major dependencies, like `webpack` and `babel` are now **peer** dependencies, so you are free to upgrade those.

While you have to configure integration with frameworks yourself, [`webpack-merge`](https://github.com/survivejs/webpack-merge) helps with this. See this example for [Vue](https://github.com/shakacode/shakapacker#other-frameworks) and scroll to the bottom for [more examples](#examples-of-v5-to-v6).

## webpacker v6.0.0.rc.6 to shakapacker v6.0.0 
See an example migration here: [PR 27](https://github.com/shakacode/react_on_rails_tutorial_with_ssr_and_hmr_fast_refresh/pull/27).
          
### Update Steps to v6.0.0 from v6.0.0.rc.6
_If you're on webpacker v5, follow below steps to get to v6.0.0.rc.6 first._

1. Change the gem name from `webpacker` to `shakapacker` and the NPM package from `@rails/webpacker` to `shakapacker`.
1. Install the peer dependencies. Run `yarn add @babel/core @babel/plugin-transform-runtime @babel/preset-env @babel/runtime babel-loader compression-webpack-plugin terser-webpack-plugin webpack webpack-assets-manifest webpack-cli webpack-merge webpack-sources webpack-dev-server`
1. Update any scripts that called `bin/webpack` or `bin/webpack-dev-server` to `bin/webpacker` or `bin/webpacker-dev-server`
1. Update your webpack config for a single config file, `config/webpack/webpack.config.js`. If you want to use the prior style of having a separate file for each NODE_ENV, you can use this shim for `config/webpack/webpack.config.js`. WARNING, previously, if you did not set `NODE_ENV`, `NODE_ENV` defaulted to `development`. Thus, you might expect `config/webpack/development.js` to run, but you'll instead be using the `config/webpack/RAILS_ENV.js  
   ```js
   const { env, webpackConfig } = require('shakapacker')
   const { existsSync } = require('fs')
   const { resolve } = require('path')

   const envSpecificConfig = () => {
     const path = resolve(__dirname, `${env.nodeEnv}.js`)
     if (existsSync(path)) {
       console.log(`Loading ENV specific webpack configuration file ${path}`)
       return require(path)
     } else {
       console.log(`WARNING: Using default webpack configuration. Did not find a Env specific file at path ${path}`)
       return webpackConfig
     }
   }
  
   module.exports = envSpecificConfig()
   ```
1. Update `babel.config.js` if you need JSX support. See [Customizing Babel Config](./customizing_babel_config.md)

## How to upgrade to Webpacker v6.0.0.rc.6 from v5
1. Ensure you have a clean working git branch. You will be overwriting all your files and reverting the changes that you don't want.

1. Consider changing from the v5 default for `source_entry_path` in `webpacker.yml`.
  ```yml
    source_path: app/javascript
    source_entry_path: packs
  ```
  consider changing to the v6 default:
  ```yml
    source_path: app/javascript
    source_entry_path: /
  ```
  Then consider moving your `app/javascript/packs/*` (including `application.js`) to `app/javascript/` and updating the configuration file.

  Note, moving your files is optional, as you can stil keep your entries in a separate directory, called something like `packs`, or `entries`. This directory is defined within the source_path.

1. **Ensure no nested directories in your `source_entry_path`.** Check if you had any entry point files in child directories of your `source_entry_path`. Files for entry points in child directories are not supported by shakacode/shakapacker v6. Move those files to the top level, adjusting any imports in those files.

  The new v6 configuration does not allow nesting, so as to allow placing the entry points at in the root directory of JavaScript. You can find this change [here](https://github.com/rails/webpacker/commit/5de0fbc1e16d3db0c93202fb39f5b4d80582c682#diff-7af8667a3e36201db57c02b68dd8651883d7bfc00dc9653661be11cd31feeccdL19).

1. Upgrade the Webpacker Ruby gem and the NPM package

   Note: [Check the gem page to verify the latest version](https://rubygems.org/gems/shakapacker), and make sure to install identical version numbers of `shakapacker` gem and package. (Gems use a period and packages use a dot between the main version number and the beta version.)

   Example going to a specific version:

   ```ruby
   # Gemfile
   gem 'shakapacker', '6.0.0.rc.13'
   ```

   ```bash
   bundle install
   ```

   ```bash
   yarn add shakapacker@6.0.0-rc.13 --exact
   ```

  ```bash
  bundle exec rails webpacker:install
  ```

  Overwrite all files and check what changed.

  Note, the webpacker:install will install the peer dependencies:
  ```bash
  yarn add @babel/core @babel/plugin-transform-runtime @babel/preset-env @babel/runtime babel-loader compression-webpack-plugin terser-webpack-plugin webpack webpack-assets-manifest webpack-cli webpack-merge webpack-sources webpack-dev-server
  ```

1. There is now a single default configuration file of `config/webpack/webpack.config.js`. Previously, the config file was set
  to `config/webpack/#{NODE_ENV}.js`. In the `config/webpack/` directory, you can either refactor your code in `test.js`, `development.js`, and `production.js` to a single file, `webpack.config.js` or you can replace the contents of `config/webpack/config.webpack.js` to conditionally load the old file based on the NODE_ENV with this snippet:
   ```js
   const { env, webpackConfig } = require('shakapacker')
   const { existsSync } = require('fs')
   const { resolve } = require('path')

   const envSpecificConfig = () => {
     const path = resolve(__dirname, `${env.nodeEnv}.js`)
     if (existsSync(path)) {
       console.log(`Loading ENV specific webpack configuration file ${path}`)
       return require(path)
     } else {
       return webpackConfig
     }
   }

   module.exports = envSpecificConfig()
   ```

1. Review the new default's changes to `webpacker.yml`. Consider each suggested change carefully, especially the change to have your `source_entry_path` be at the top level of your `source_path`.
   The v5 default used `packs` for `source_entry_path`:
   ```yml
   source_path: app/javascript
   source_entry_path: packs
   ```
   The v6 default uses the top level directory.
   ```yml
   source_path: app/javascript
   source_entry_path: /
   ```
   If you prefer this configuratiom, then you will move your `app/javascript/packs/*` (including `application.js`) to `app/javascript/` and update the configuration file.

   Note, moving your files is optional, as you can stil keep your entries in a separate directory, called something like `packs`, or `entries`. This directory is defined with the `source_path`.

1. **Ensure no nested directories in your `source_entry_path`.** Check if you had any entry point files in child directories of your `source_entry_path`. Files for entry points in child directories are not supported by shakacode/shakapacker v6. Move those files to the top level, adjusting any imports in those files.

   The new v6 configuration does not allow nesting, so as to allow placing the entry points at in the root directory of JavaScript. You can find this change [here](https://github.com/rails/webpacker/commit/5de0fbc1e16d3db0c93202fb39f5b4d80582c682#diff-7af8667a3e36201db57c02b68dd8651883d7bfc00dc9653661be11cd31feeccdL19).

1. Update `webpack-dev-server` to the current version, greater than 4.2, updating `package.json`.

1. Update API usage of the view helpers by changing `javascript_packs_with_chunks_tag` and `stylesheet_packs_with_chunks_tag` to `javascript_pack_tag` and `stylesheet_pack_tag`. Ensure that your layouts and views will only have **at most one call** to `javascript_pack_tag` and **at most one call** to `stylesheet_pack_tag`. You can now pass multiple bundles to these view helper methods. If you fail to changes this, you may experience performance issues, and other bugs related to multiple copies of React, like [issue 2932](https://github.com/rails/webpacker/issues/2932).  If you expose jquery globally with `expose-loader,` by using `import $ from "expose-loader?exposes=$,jQuery!jquery"` in your `app/javascript/application.js`, pass the option `defer: false` to your `javascript_pack_tag`.

1. If you are using any integrations like `css`, `postcss`, `React` or `TypeScript`. Please see https://github.com/shakacode/shakapacker#integrations section on how they work in v6.

1. Import `environment` changed to `webpackConfig`. For example, the new code looks like:
   ```js
   // config/webpack/webpack.config.js
   const { webpackConfig, merge } = require('shakapacker')
   const customConfig = require('./custom')

   module.exports = merge(webpackConfig, customConfig)
   ```

1. Copy over custom browserlist config from `.browserslistrc` if it exists into the `"browserslist"` key in `package.json` and remove `.browserslistrc`.

1. Remove `babel.config.js` if you never changed it. Configure your `package.json` to use the default:
  ```json
  "babel": {
    "presets": [
      "./node_modules/shakapacker/package/babel/preset.js"
    ]
  }
  ```
  See customization example the [Customizing Babel Config](./docs/customizing_babel_config.md) for React configuration.

1. `extensions` was removed from the `webpacker.yml` file. Move custom extensions to your configuration by merging an object like this. For more details, see docs for [Webpack Configuration](https://github.com/shakacode/shakapacker/blob/master/README.md#webpack-configuration)

    ```js
    {
      resolve: {
        extensions: ['.ts', '.tsx', '.vue', '.css']
      }
    }
    ```
1. In `webpacker.yml`, check if you had `watched_paths`. That is not `additional_paths`.

1. Some dependencies were removed in [PR 3056](https://github.com/rails/webpacker/pull/3056). If you see the error: `Error: Cannot find module 'babel-plugin-macros'`, or similar, then you need to `yarn add <dependency>` where <dependency> might include: `babel-plugin-macros`, `case-sensitive-paths-webpack-plugin`, `core-js`, `regenerator-runtime`. Or you might want to remove your dependency on those.

1. Review the new default's changes to `webpacker.yml` and `config/webpack`. Consider each suggested change carefully, especially the change to have your `source_entry_path` be at the top level of your `source_path`.

1. Make sure that you can run `bin/webpacker` without errors.

1. Try running `RAILS_ENV=production bin/rails assets:precompile`. If all goes well, don't forget to clean the generated assets with `bin/rails assets:clobber`.

1. Run `yarn add webpack-dev-server` if those are not already in your dev dependencies. Make sure you're using v4+.

1. Update any scripts that called `/bin/webpack` or `bin/webpack-dev-server` to `/bin/webpacker` or `bin/webpacker-dev-server`

1. In `bin/webpacker` and `bin/webpacker-dev-server`, The default NODE_ENV, if not set, will be the RAILS_ENV. Previously, NODE_ENV would default to development if not set. Thus, the old bin stubs would use the webpack config corresponding to `config/webpack/development.js`. After the change, if `RAILS_ENV` is `test`, then `NODE_ENV` is `test`. The final 6.0 release changes to using a single `webpack.config.js`.

1. Now, follow the steps above to get to shakapacker v6 from webpacker v6.0.0.rc.6


## Examples of v5 to v6

1. [React on Rails Project with HMR and SSR](https://github.com/shakacode/react_on_rails_tutorial_with_ssr_and_hmr_fast_refresh/compare/webpacker-5.x...master)
2. [Vue and Sass Example](https://github.com/guillaumebriday/upgrade-webpacker-5-to-6)
