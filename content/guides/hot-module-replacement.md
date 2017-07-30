---
title: Hot Module Replacement
sort: 6
contributors:
  - jmreidy
  - jhnns
  - sararubin
  - aiduryagin
  - rohannair
  - joshsantos
  - drpicox
  - skipjack
  - sbaidon
  - gdi2290
related:
  - title: Concepts - Hot Module Replacement
    url: /concepts/hot-module-replacement
  - title: API - Hot Module Replacement
    url: /api/hot-module-replacement
---

T> This guide extends on code examples found in the [Development](/guides/development) guide.

Hot Module Replacement (or HMR) is one of the most useful features offered by webpack. It allows all kinds of modules to be updated at runtime without the need for a full refresh. This page focuses on __implementation__ while the [concepts page](/concepts/hot-module-replacement) gives more details on how it works and why it's useful.

W> __HMR__ is not intended for use in production, meaning it should only be used in development. See the [building for production guide](/guides/production) for more information.


## Enabling HMR

This feature is great for productivity. All we need to do is update our [webpack-dev-server](https://github.com/webpack/webpack-dev-server) configuration, and use webpack's built in HMR plugin.

__webpack.config.js__

``` diff
  const path = require('path');
  const HtmlWebpackPlugin = require('html-webpack-plugin');
+ const webpack = require('webpack');

  module.exports = {
    entry: {
      app: './src/index.js',
      print: './src/print.js'
    },
    devtool: 'inline-source-map',
    devServer: {
      contentBase: './dist',
+     hot: true
    },
    plugins: [
      new HtmlWebpackPlugin({
        title: 'Hot Module Replacement'
      }),
+     new webpack.HotModuleReplacementPlugin()
    ],
    output: {
      filename: '[name].bundle.js',
      path: path.resolve(__dirname, 'dist')
    }
  };
```

You can also use the CLI to modify the [webpack-dev-server](https://github.com/webpack/webpack-dev-server) configuration with the following command: `webpack-dev-server --hotOnly`.

To get it up and running let's run `npm start` from the command line.

Now let's update the `index.js` file so that when a change inside `print.js` is detected we tell webpack to accept the updated module.

__index.js__

``` diff
  import _ from 'lodash';
  import printMe from './print.js';

+ if (module.hot) {
+   module.hot.accept('./print.js', function() {
+     console.log('Accepting the updated printMe module!');
+     printMe();
+   })
+ }

  function component() {
    var element = document.createElement('div');
    var btn = document.createElement('button');

    element.innerHTML = _.join(['Hello', 'webpack'], ' ');

    btn.innerHTML = 'Click me and check the console!';
    btn.onclick = printMe;

    element.appendChild(btn);

    return element;
  }

  document.body.appendChild(component());
```

Start changing the `console.log` statement in `print.js`, and you should see the following output in the browser console.

__print.js__

``` diff
  export default function printMe() {
-   console.log('I get called from print.js!');
+   console.log('Updating print.js...')
  }
```

__console__

``` diff
[HMR] Waiting for update signal from WDS...
main.js:4395 [WDS] Hot Module Replacement enabled.
+ 2main.js:4395 [WDS] App updated. Recompiling...
+ main.js:4395 [WDS] App hot update...
+ main.js:4330 [HMR] Checking for updates on the server...
+ main.js:10024 Accepting the updated printMe module!
+ 0.4b8ee77….hot-update.js:10 Updating print.js...
+ main.js:4330 [HMR] Updated modules:
+ main.js:4330 [HMR]  - 20
+ main.js:4330 [HMR] Consider using the NamedModulesPlugin for module names.
```


## Gotchas

Hot Module Replacement can be tricky. To show this, let's go back to our working example. If you go ahead and click the button on the example page, you will realize the console is printing the old `printMe` function.

This is happening because the button's `onclick` event handler is still bound to the original `printMe` function.

To make this work with HMR we need to update that binding to the new `printMe` function using `module.hot.accept`:

__print.js__

``` diff
  import _ from 'lodash';
  import printMe from './print.js';

  if (module.hot) {
    module.hot.accept('./print.js', function() {
      console.log('Accepting the updated printMe module!');
-     printMe();
+     document.body.removeChild(element);
+     element = component(); // Re-render the "component" to update the click handler
+     document.body.appendChild(element);
    })
  }

  let element = component();

  function component() {
    var element = document.createElement('div');
    var btn = document.createElement('button');

    element.innerHTML = _.join(['Hello', 'webpack'], ' ');

    btn.innerHTML = 'Click me and check the console!';
    btn.onclick = printMe;  // onclick event is bind to the original printMe function

    element.appendChild(btn);

    return element;
  }

  document.body.appendChild(element);
```

This is just one example, but there are many others that can easily trip people up. Luckily, there are a lot of loaders out there (some of which are mentioned below) that will make hot module replacement much easier.


## HMR with Stylesheets

Hot Module Replacement with CSS is actually fairly straightforward with the help of the `style-loader`. This loader uses `module.hot.accept` behind the scenes to patch `<style>` tags when CSS dependencies are updated.

First let's install both loaders with the following command: 

```bash
npm install --save-dev style-loader css-loader
```

Now let's update the configuration file to make use of the loader.

__webpack.config.js__

```diff
  const path = require('path');
  const HtmlWebpackPlugin = require('html-webpack-plugin');
  const webpack = require('webpack');

  module.exports = {
    entry: {
      app: './src/index.js',
      print: './src/print.js'
    },
    devtool: 'inline-source-map',
    devServer: {
      contentBase: './dist',
      hot: true
    },
+   module: {
+     rules: [
+       {
+         test: /\.css$/,
+         use: ['style-loader', 'css-loader']
+       }
+     ]
+   },
    plugins: [
      new HtmlWebpackPlugin({
        title: 'Hot Module Replacement'
      }),
      new webpack.HotModuleReplacementPlugin()
    ],
    output: {
      filename: '[name].bundle.js',
      path: path.resolve(__dirname, 'dist')
    }
  };
```

Hot loading stylesheets is as easy as importing them into a module:

__project__

``` diff
  webpack-demo
  | - package.json
  | - webpack.config.js
  | - /dist
    | - bundle.js
  | - /src
    | - index.js
    | - print.js
+   | - styles.css
```

__styles.css__

``` css
body {
  background: blue;
}
```

__index.js__

``` diff
  import _ from 'lodash';
  import printMe from './print.js';
+ import './styles.css';

  if (module.hot) {
    module.hot.accept('./print.js', function() {
      console.log('Accepting the updated printMe module!');
      document.body.removeChild(element);
      element = component(); // Re-render the "component" to update the click handler
      document.body.appendChild(element);
    })
  }

  let element = component();

  function component() {
    var element = document.createElement('div');
    var btn = document.createElement('button');

    element.innerHTML = _.join(['Hello', 'webpack'], ' ');

    btn.innerHTML = 'Click me and check the console!';
    btn.onclick = printMe;  // onclick event is bind to the original printMe function

    element.appendChild(btn);

    return element;
  }

  document.body.appendChild(element);
```

Change the style on `body` to `background: red;` and you should immediately see the page's background color change without a full refresh.

__styles.css__

``` diff
  body {
-   background: blue;
+   background: red;
  }
```


## Other Code and Frameworks

There are many other loaders and examples out in the community to make HMR interact smoothly with a variety of frameworks and libraries...

- [React Hot Loader](https://github.com/gaearon/react-hot-loader): Tweak react components in real time.
- [Vue Loader](https://github.com/vuejs/vue-loader): This loader supports HMR for vue components out of the box.
- [Elm Hot Loader](https://github.com/fluxxu/elm-hot-loader): Supports HMR for the Elm programming language.
- [Redux HMR](https://survivejs.com/webpack/appendices/hmr-with-react/#configuring-hmr-with-redux): No loader or plugin necessary! A simple change to your main store file is all that's required.
- [Angular HMR](https://github.com/AngularClass/angular-hmr): No loader necessary! A simple change to your main NgModule file is all that's required to have full control over the HMR APIs.

T> If you know of any other loaders or plugins that help with or enhance Hot Module Replacement please submit a pull request to add to this list!