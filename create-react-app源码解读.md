### 执行顺序

#### 0、实用工具

线上调试node环境/学习node命令 http://run.nodejs.cn/

#### 1、将常用命令注册

commander实际上就是在命令行中以该命令开头调用的指令（node指令）

```js
// 作为node.js命令行界面
const commander = require('commander');
// 终端文字样式
const chalk = require('chalk');

const program = new commander.Command(packageJson.name)
    .version(packageJson.version)
    .arguments('<project-directory>')
    .usage(`${chalk.green('<project-directory>')} [options]`)
    .action(name => {
      projectName = name;
    })
    .option('--verbose', 'print additional logs')
    .option('--info', 'print environment debug info')
    .option(
      '--scripts-version <alternative-package>',
      'use a non-standard version of react-scripts'
    )
    .option(
      '--template <path-to-template>',
      'specify a template for the created project'
    )
    .option('--use-npm')
    .option('--use-pnp')
    .allowUnknownOption()
```

例如：

```shell
yarn -v
v14.15.4
node -v
1.22.10
create-react-app --info
```

#### 2、获取用户环境信息

```js
// 在node环境中获取用户机器信息
const envinfo = require('envinfo');

envinfo.run(
    {
        System: ['OS', 'CPU'],
        Binaries: ['Node', 'Yarn', 'npm'],
        Browsers: ['Chrome', 'Firefox', 'Safari'],
        npmPackages: ['styled-components', 'babel-plugin-styled-components'],
    },
    { json: true, showNotFound: true }
).then(console.log);

// ============ log ============
  
{
    "System": {
        "OS": "macOS High Sierra 10.13",
        "CPU": "x64 Intel(R) Core(TM) i7-4870HQ CPU @ 2.50GHz"
    },
    "Binaries": {
        "Node": {
            "version": "8.11.0",
            "path": "~/.nvm/versions/node/v8.11.0/bin/node"
        },
        "Yarn": {
            "version": "1.5.1",
            "path": "~/.yarn/bin/yarn"
        },
        "npm": {
            "version": "5.6.0",
            "path": "~/.nvm/versions/node/v8.11.0/bin/npm"
        }
    },
    "Browsers": {
        "Chrome": {
            "version": "67.0.3396.62"
        },
        "Firefox": {
            "version": "59.0.2"
        },
        "Safari": {
            "version": "11.0"
        }
    },
    "npmPackages": {
        "styled-components": {
            "wanted": "^3.2.1",
            "installed": "3.2.1"
        },
        "babel-plugin-styled-components": "Not Found"
    }
}
```

#### 3、检验是否输入项目名称

若没输入 退出执行

```js
if (typeof projectName === 'undefined') {
  console.error('Please specify the project directory:');
	// console.log 一坨
  process.exit(1);
}
```

#### 4、检查版本号

首先执行checkForLatestVersion函数,可以直接通过浏览器输入看看返回

```js
// 实际上就是get请求获取最新包的情况
function checkForLatestVersion() {
  return new Promise((resolve, reject) => {
    https
      .get(
        'https://registry.npmjs.org/-/package/create-react-app/dist-tags',
        res => {
          if (res.statusCode === 200) {
            let body = '';
            res.on('data', data => (body += data));
            res.on('end', () => {
              resolve(JSON.parse(body).latest);
            });
          } else {
            reject();
          }
        }
      )
      .on('error', () => {
        reject();
      });
  });
}


=============
  
{
	"next": "4.0.0-next.117", // 下一代版本
	"latest": "4.0.3",	// 最新版本
	"canary": "3.3.0-next.38" // 最低可执行版本
}
```

对请求进行处理：

1、出错=》同步去子进程执行 npm view create-react-app version 并结束

```js 
const execSync = require('child_process').execSync;

return execSync('npm view create-react-app version').toString().trim();
```

2、对版本号进行对比

当前create-react-app的版本(packageJson.version) 小于 最新版本时 会退出当前程序，要求重新全局安装

```JS 
// 操作版本号（校验合法、比较两个版本大小等）
const semver = require('semver');

semver.lt(packageJson.version, latest)
```

3、当都符合时，执行createApp方法

```js
  checkForLatestVersion()
    .catch(() => {
    // 捕获异常（请求错误时）
      try {
        return execSync('npm view create-react-app version').toString().trim();
      } catch (e) {
        return null;
      }
    })
		// 正常时
    .then(latest => {
      if (latest && semver.lt(packageJson.version, latest)) {
        // 当版本和最新的不一致时，console.err,并退出
        // console.log 一坨
        process.exit(1);
      } else {
        createApp(
          projectName,
          program.verbose,
          program.scriptsVersion,
          program.template,
          program.useNpm,
          program.usePnp
        );
      }
    });
```

#### 5、创建app文件【createApp方法】

1、判断node版本是否 >= 10

若小于10版本，自动回退执行react-scripts@0.9.x（这个版本支持node4）

```js 
function createApp(name, verbose, version, template, useNpm, usePnp) {
  const unsupportedNodeVersion = !semver.satisfies(process.version, '>=10');
  if (unsupportedNodeVersion) {
    // 当node版本小于10时将使用的react-scripts版本下降，并提示用户更新
    // ...console.log 一坨
    version = 'react-scripts@0.9.x';
  }

  // ...接下文
}
```



2、校验用户输入要创建的项目名

```js
const validateProjectName = require('validate-npm-package-name');

function checkAppName(appName) {
  // 使用库检验『创建的项目名』
  const validationResult = validateProjectName(appName);
  /*
  返回值可能有三项
   1、现在node库命名规则是否合法
   2、老的node库命名规则是否合法
   3、警告console
  {
      validForNewPackages: false,
      validForOldPackages: true,
      warnings: [
        "name can no longer contain capital letters",
        "name can no longer contain more than 214 characters"
      ]
  }
  */
  // 不满足现node命名规则报错并结束
  if (!validationResult.validForNewPackages) {
    // 一坨console.err
    process.exit(1);
  }

  // 不能和三个依赖名重名、否则也报错结束
  const dependencies = ['react', 'react-dom', 'react-scripts'].sort();
  if (dependencies.includes(appName)) {
    // 一坨console.err
    process.exit(1);
  }
}
```



3、保证文件存在并创建

```js 
function isSafeToCreateProjectIn(root, name) {
  // 常见的合法文件名
  const validFiles = [
    '.DS_Store',
    '.git',
    '.gitignore',
    'LICENSE',
    'README.md',
   // ...other
  ];

  const errorLogFilePatterns = [
    'npm-debug.log',
    'yarn-error.log',
    'yarn-debug.log',
  ];
  const isErrorLog = file => {
    return errorLogFilePatterns.some(pattern => file.startsWith(pattern));
  };

  // 获取有冲突的文件名，筛选三类
  // 1、不是合法文件名
  // 2、IntelliJ IDEA自动创建文件
  // 3、不是合法错误log名
  const conflicts = fs
    .readdirSync(root)
    .filter(file => !validFiles.includes(file))
    .filter(file => !/\.iml$/.test(file))
    // Don't treat log files from previous installation as conflicts
    .filter(file => !isErrorLog(file));

  // 当冲突存在时
  if (conflicts.length > 0) {
    // 一坨console，并退出
    return false;
  }

  // 删除所有的打印log文件
  fs.readdirSync(root).forEach(file => {
    if (isErrorLog(file)) {
      fs.removeSync(path.join(root, file));
    }
  });
  return true;
}
```



```js 
function createApp(name, verbose, version, template, useNpm, usePnp) {
  
  // ...接上文
  const root = path.resolve(name);
  const appName = path.basename(root);
	// 上面已解读
  checkAppName(appName);
  
  fs.ensureDirSync(name);
  // 判断是否有冲突，并删除打印日志文件
  if (!isSafeToCreateProjectIn(root, name)) {
    process.exit(1);
  }
	
  // 将最基础的package.json文件写入
  const packageJson = {
    name: appName,
    version: '0.1.0',
    private: true,
  };
  fs.writeFileSync(
    path.join(root, 'package.json'),
    JSON.stringify(packageJson, null, 2) + os.EOL
  );

  // 默认使用yarn
  const useYarn = useNpm ? false : shouldUseYarn();
  // shouldUseYarn() => 通过查看yarn的版本，成功代表可以使用yarn命令【优先使用yarn】
  
  // 切换目录
  const originalDirectory = process.cwd();
  process.chdir(root);
  
  // 
  if (!useYarn && !checkThatNpmCanReadCwd()) {
    process.exit(1);
  }

	// 确保能使用yarn的情况下，将yarn.lock拷贝到目录下
  if (useYarn) {
    let yarnUsesDefaultRegistry = true;
    try {
      yarnUsesDefaultRegistry =
        execSync('yarnpkg config get registry').toString().trim() ===
        'https://registry.yarnpkg.com';
    } catch (e) {
      // ignore
    }
    if (yarnUsesDefaultRegistry) {
      fs.copySync(
        require.resolve('./yarn.lock.cached'),
        path.join(root, 'yarn.lock')
      );
    }
  }

  // 调用真实安装项目命令
  run(
    root,
    appName,
    version,
    verbose,
    originalDirectory,
    template,
    useYarn,
    usePnp
  );
}
```

