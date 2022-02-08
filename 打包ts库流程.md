# 打包ts库流程

## 背景简介

> 基于 @baidu/fe-components-v2 开发的高阶组件库 [@baidu/fe-components-hoc](https://console.cloud.baidu-int.com/devops/icode/repos/baidu/personal-code/HOC/tree/master)

- 业务库高阶组件+自定义hooks使用率高，但每个项目都得复制粘贴非常麻烦，因此想到封装成自己的库，在此记录遇到的各种坑！

- 使用webpack5  babel ttsc

> 先看下整体包格式

<img style="display: inline" src="https://raw.githubusercontent.com/caifeng123/pictures/master/image-20211202154219709.png" />

```js
├── babel.config.js // babel配置
├── config          // webpack打包配置
│   ├── webpack.dev.config.js
│   └── webpack.prod.config.js
├── example         // 包中demo，用作dev检验lib与src是否正确
│   └── src
│       ├── app.tsx
│       └── index.html
├── lib             // 打包后的文件
│   ├── @types      // 对应文件的ts  ---start
│   │   ├── form.d.ts
│   │   └── index.d.ts
│   ├── HocComponents
│   │   ├── FetchSelect
│   │   │   └── index.d.ts
│   │   ├── FormItem
│   │   │   ├── index.d.ts
│   │   │   ├── ui.d.ts
│   │   │   └── utils.d.ts
│   │   ├── SeriesSelect
│   │   │   ├── index.d.ts
│   │   │   └── ui.d.ts
│   │   └── index.d.ts
│   ├── hooks
│   │   ├── index.d.ts
│   │   ├── useBoolean.d.ts
│   │   ├── useCascade.d.ts
│   │   ├── useClientToken.d.ts
│   │   ├── useConcurrent.d.ts
│   │   ├── useDeepEffect.d.ts
│   │   ├── useEffectCallback.d.ts
│   │   └── useSetState.d.ts
│   ├── index.d.ts  // 对应文件的ts  ---end
│   ├── index.js    // 打包后的文件
│   └── index.js.LICENSE.txt
├── package.json    // 懒得提
├── src             // 要封装成库的代码
│   ├── @types
│   │   ├── form.ts
│   │   └── index.ts
│   ├── HocComponents
│   │   ├── FetchSelect
│   │   │   └── index.tsx
│   │   ├── FormItem
│   │   │   ├── index.tsx
│   │   │   ├── ui.ts
│   │   │   └── utils.tsx
│   │   ├── SeriesSelect
│   │   │   ├── index.tsx
│   │   │   └── ui.ts
│   │   └── index.ts
│   ├── hooks
│   │   ├── index.ts
│   │   ├── useBoolean.ts
│   │   ├── useCascade.ts
│   │   ├── useClientToken.ts
│   │   ├── useConcurrent.ts
│   │   ├── useDeepEffect.ts
│   │   ├── useEffectCallback.ts
│   │   └── useSetState.ts
│   └── index.ts
├── tsconfig.json // ts配置
└── yarn.lock
```



## 各个文件夹注意点

### config

> ##### 作为存放打包配置 即 webpack.config.js的文件夹，大部分情况分为开发者模式dev和生产者模式prod。下面我们来看看要注意的地方

#### `webpack.dev.config.js`

```js
const path = require('path');

/**  用作dev模式下热加载 联合上上述目录中的example文件夹 **/
const HtmlWebpackPlugin = require('html-webpack-plugin');

const htmlWebpackPlugin = new HtmlWebpackPlugin({
    template: path.join(__dirname, '../example/src/index.html'),
    filename: './index.html'
});
/**  用作dev模式下热加载  **/

module.exports = {
    mode: "development",
  	// 加载入口
    entry: path.join(__dirname, '../example/src/app.tsx'),
    // 解析规则
    module: {
        rules: [{
            test: /\.(jsx?|tsx?)$/,
            use: ['babel-loader'],
            exclude: /node_modules/
        }, {
            test: /\.css$/i,
            use: ['style-loader', 'css-loader']
        }, {
            test: /\.less$/i,
            use: [
                'style-loader',
                'css-loader',
                'less-loader',
            ]
        }]
    },
  	// html热加载插件
    plugins: [
        htmlWebpackPlugin
    ],
    resolve: {
        // 解析扩展名
        extensions: [".ts", ".tsx", ".js"],
        // 解析别名
        alias:{
            '@': path.resolve(__dirname, '../src')
        },
    },
    devServer: {
      // 开发端口号
        port: 3001
    }
};
```

#### `webpack.prod.config.js`

```js
const path = require('path');

module.exports = {
    mode: 'production', // 开发模式
  	// 入口
    entry: path.join(__dirname, '../src'),
  	// 打包出口
    output: {
        path: path.join(__dirname, '../lib/'),
        filename: 'index.js',
	      // 采用通用模块定义
        libraryTarget: 'umd',
 				// 当输出为 library 时，尤其是当 libraryTarget 为 'umd'时，此选项将决定使用哪个全局对象来挂载 library。
      	// 为了使 UMD 构建在浏览器和 Node.js 上均可用，应将 output.globalObject 选项设置为 'this'。
      	// 对于类似 web 的目标，默认为 self。
        globalObject: 'this'
    },
    module: {
        rules: [{
                test: /\.(jsx?|tsx?)$/,
                use: ['babel-loader', {
                    loader: 'ts-loader',
                    options: {
                      // 使用 ttypescript 目的是为了在ts-loader打包生成d.ts时，可使用alias
                      // 目前 TypeScript 不支持 tsconfig.json 中的自定义转换器，但以编程方式支持它。
                      // 并且无法使用tsc命令使用自定义转换器编译您的文件。
                      // 通过动态修补编译模块以使用来自tsconfig.json.
                        compiler: 'ttypescript'
                    }
                }],
                exclude: /node_modules/
        },  {
                test: /\.css$/i,
                use: ['style-loader', 'css-loader'],
        }, {
                test: /\.less$/i,
                use: [
                    'style-loader',
                    'css-loader',
                    'less-loader',
                ],
        }],
    },
    resolve: {
        extensions: ['.js', '.jsx', '.ts', '.tsx'],
        alias:{
            '@': path.resolve(__dirname, '../src')
        },
    },
  	// 防止将某些 import 的包(package)打包到 bundle 中，而是在运行时(runtime)再去从外部获取这些扩展依赖(external dependencies)。
    externals: {
        react: 'react',
        'react-dom': {
            root: 'ReactDOM',
            commonjs2: 'react-dom',
            commonjs: 'react-dom',
            amd: 'react-dom',
        },
        '@baidu/fe-components-v2': {
            commonjs2: '@baidu/fe-components-v2',
            commonjs: '@baidu/fe-components-v2',
            amd: '@baidu/fe-components-v2',
        },
        'lodash': {
            commonjs2: 'lodash',
            commonjs: 'lodash',
            amd: 'lodash',
            root: '_'
        }
    },
};
```



### example/src

> 用作dev环境下，测试组件库src是否正常书写 与 `webpack.dev.config.js` 相关



### lib

> 打包出的文件，由两部分组成，具体配置方法见 tsconfig.json

- 打包后的文件index.js
- 打包出来的ts类型（d.ts结尾文件）目的是为了在引入库时有ts类型校验提示也无需在安装@type/xxx



### package.json

> 懒得提，不过有几项要注意的

- types - 指向打包的声明文件 指向入口文件的类型文件
- main - 指定入口文件
- files - 文件默认会加入到npm publish发布的包中
- peerDependencies - 会在安装结束后检查本次安装是否正确，如果不正确会给用户打印警告提示。

```json
{
  "name": "xxx",
  "description": "xxx",
  "version": "0.0.2",
  // 指定入口文件
  "main": "lib/index.js",
  // 指向打包的声明文件 指向入口文件的类型文件
  "types": "lib/index.d.ts",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "dev": "webpack-dev-server --config config/webpack.dev.config.js",
    "build": "webpack --config config/webpack.prod.config.js",
    "pub": "npm run build && npm publish",
    "lint": "npx fecs src --color --rule --type=js"
  },
  "config": {
    "ghooks": {
      "pre-commit": "npm run lint",
      "commit-msg": "commitlint -x @commitlint/config-conventional -e"
    }
  },
  "keywords": [],
  "author": "cc123nice",
  "license": "MIT",
  // 文件默认会加入到npm publish发布的包中
  "files": [
    "lib"
  ],
  // 会在安装结束后检查本次安装是否正确，如果不正确会给用户打印警告提示。
  "peerDependencies": {
    "内部库": "xxx",
    "lodash": ">=4.0.0",
    "moment": ">=2.0.0",
    "react": ">=16.0.0",
    "react-dom": ">=16.0.0"
  },
  "dependencies": {
    "内部库": "xxx",
    "react": "^16.13.1",
    "react-dom": "^16.13.1",
    "styled-components": "^5.2.1"
  },
  "devDependencies": {
    "@babel/cli": "^7.16.0",
    "@babel/core": "^7.16.0",
    "@babel/preset-env": "^7.16.4",
    "@babel/preset-react": "^7.16.0",
    "@babel/preset-typescript": "^7.16.0",
    "@types/react": "^17.0.37",
    "@webpack-cli/serve": "^1.6.0",
    "babel-eslint": "^11.0.0-beta.2",
    "babel-loader": "^8.2.3",
    "babel-plugin-import": "^1.13.3",
    "css-loader": "^6.5.1",
    "eslint": "^7.5.0",
    "html-webpack-plugin": "^5.5.0",
    "less-loader": "^10.2.0",
    "mini-css-extract-plugin": "^2.4.5",
    "style-loader": "^3.3.1",
    "ts-loader": "^9.2.6",
    "ttypescript": "^1.5.13",
    "typescript": "^4.5.2",
    "typescript-transform-paths": "^3.3.1",
    "webpack": "^5.64.4",
    "webpack-cli": "^4.9.1",
    "webpack-dev-server": "^4.6.0"
  }
}
```

参考: [package.json字段介绍](https://segmentfault.com/a/1190000039747110)



### tsconfig.json

> ts特殊配置覆盖，默认配置请看 [tsconfig.json](https://segmentfault.com/a/1190000022809326)

```json
{
    "compilerOptions": {
        "jsx": "react", // 需要编译jsx（或tsx）
        "declaration": true, // 将生成.d.ts文件
        "noImplicitAny": false, // 允许隐式的any类型
        "removeComments": false, // 不删除注释
        "outDir": "./lib", // 输出目录
        "paths": {  // 别名
            "@/*": ["./src/*"]
        },
        // ttsc转换插件，目前 TypeScript 不支持 tsconfig.json 中的自定义转换器
        "plugins": [
            { "transform": "typescript-transform-paths" },
            { "transform": "typescript-transform-paths", "afterDeclarations": true }
        ],
        // 声明文件目录
        "typeRoots": ["./node_modules/@types", "./src/@types"],
        "lib": ["es2020"]
    },
    "include": ["src/**/*"],
    "exclude": ["node_modules", "example/**/*"]
}
```

参考: [tsconfig.json](https://segmentfault.com/a/1190000022809326)、[ttypescript](https://github.com/cevek/ttypescript)



### babel.config.js

> 兼容ts，babel预设，react、按需加载组件库样式

```js
module.exports = {
    presets: [
        [
	          // babel预设
            '@babel/preset-env',
            {
              	// babel 推荐usage打包方式自动加载polyfill体积会大，为了小可以使用entry+polyfill
                useBuiltIns: 'usage',
                corejs: '3',
                loose: true // 只使用es6 不转义成es5
            }
        ],
        '@babel/preset-react',	// react 预设编译
        "@babel/preset-typescript" // typescript 编译预设
    ],
    plugins: [
        [
          	// 按需加载css，无需全局导入组件库样式，使用umi插件
            'babel-plugin-import',
            {
                libraryName: '内部库',
                style: true
            },
            '内部库'
        ],
    ]
};
```

参考: [babel-preset-env](https://www.babeljs.cn/docs/babel-preset-env)、[@babel/preset-react](https://www.babeljs.cn/docs/babel-preset-react)、[@babel/preset-typescript](https://www.babeljs.cn/docs/babel-preset-typescript)、[babel-plugin-import](https://github.com/umijs/babel-plugin-import)



## 坑点

> 对于编译tsx来说有三种方式

### 1、使用 @babel/preset-typescript

> 正常babel-loader是用来将新功能新方法降级的loader，换句话说是将箭头函数、promise、jsx转义成低版本js可认识的语言，兼容所有端
>
> 对于babel来说tsx和jsx一样，可以被转义但ts最需要的type、interface都被✂️，只留下最纯朴的js

```js
// webpack.config.js
...
module: {
    rules: [{
        test: /\.(jsx?|tsx?)$/,
        use: ['babel-loader'],
        exclude: /node_modules/
    }]
}
...
```

```json
// babel.config.json
{
  "presets": [
    [
      "@babel/preset-env",
      {
        "corejs": "3",
        "useBuiltIns": "usage"
      }
    ],
    '@babel/preset-react',
    "@babel/preset-typescript"
  ]
}
```

优点：快

缺点：没有d.ts、type、interface

### 2、使用 ts-loader

> 开发npm包的时候，往往最需要的是对应的声明文件即 d.ts，能够引入库时一样得到ts对应提示
>
> 实际上是使用typescript调用 ` tsc --declaration -p ./ --emitDeclarationOnly --outDir types`

<img style="display: inline" src="https://raw.githubusercontent.com/caifeng123/pictures/master/image-20211203154046714.png" />

```json
// webpack.config.js
module: {
    rules: [{
        test: /\.(jsx?|tsx?)$/,
        use: ['babel-loader', 'ts-loader'],
        exclude: /node_modules/
    }]
}
```

通过组合tsconfig.json对应配置声明文件

```json
{
    "compilerOptions": {
        "jsx": "react", // 需要编译jsx（或tsx）
        "declaration": true, // 将生成.d.ts文件
        "removeComments": false, // 不删除注释
        "outDir": "./lib", // 输出目录
        "paths": {  // 别名
            "@/*": ["./src/*"]
        },
        ...
    }
    ...
}
```

最后需要我们引入时能找到对应ts文件需要在package.json中添加types路径【上面也有提及】

```json
{
  "name": "xxx",
  "description": "xxx",
  "version": "0.0.2",
  // 指定入口文件
  "main": "lib/index.js",
  // 指向打包的声明文件 指向入口文件的类型文件
  "types": "lib/index.d.ts"
  ...
}
```

优点：为每个ts生成d.ts

缺点：稍慢、生成对应 .d.ts 无法解析别名/将别名转换为相对路径

![image-20211203160023439](https://raw.githubusercontent.com/caifeng123/pictures/master/image-20211203160023439.png)

- 主要原因是因为typescript认为出现别名这个事情不应该是个问题，因此使用tsc编译得到的别名需要自己处理/放弃别名

### 3、使用 ttsc + tsloader

> 目前 TypeScript 不支持 tsconfig.json 中的自定义转换器，但以编程方式支持它。
>
> 并且无法使用`tsc`命令使用自定义转换器编译您的文件。
>
> TTypescript（Transformer TypeScript）通过动态修补编译模块以使用来自`tsconfig.json`.
>
> 使用 ttsc 和 ttsserver 包装器代替 tsc 和 tsserver。这个包装器首先尝试使用本地安装的typescript

因此 首先对webpack.config.json进行调整,打包时 使用ttypescript打包

```js
// webpack.config.js
module: {
    rules: [{
        test: /\.(jsx?|tsx?)$/,
        use: ['babel-loader', {
            loader: 'ts-loader',
            options: {
                compiler: 'ttypescript'
            }
        }],
        exclude: /node_modules/
    }]
}
```

对于tsconfig中也要些许改变 添加上ttsc的转换器

```json
{
    "compilerOptions": {
        "jsx": "react",
        "declaration": true, // 将生成.d.ts文件
        "noImplicitAny": false, // 允许隐式的any类型
        "removeComments": false, // 不删除注释
        "outDir": "./lib", // 输出目录
        "paths": {  // 别名
            "@/*": ["./src/*"]
        },
        // ttsc转换插件
        "plugins": [
            { "transform": "typescript-transform-paths" },
            { "transform": "typescript-transform-paths", "afterDeclarations": true }
        ],
				...
    },
		...
}
```

