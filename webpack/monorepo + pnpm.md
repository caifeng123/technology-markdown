# 代码包管理方式

主要有两种方法来托管和管理 Git 代码

- Mono-repo
- Multi-repo

接下来我们来了解一下区别与优劣

![](https://raw.githubusercontent.com/caifeng123/pictures/master/a4672bb4175cfcd07f03730197831d76d28626-20221003110049722.png)

## multirepo

> 大部分个人项目都是使用这种管理方式，即每个项目都是一个git管理项目。

### 优点

- 每个服务和库都有自己的版本控制。
- 代码 checkout 和 pull 是小型且独立的，因此即使项目规模增大，也不存在性能问题。
- 团队可以独立工作，不需要访问整个代码库。
- 更快的开发和灵活性。
- 每个服务都可以单独发版，并有自己的部署周期，从而使 CI 和 CD 更易于实现。
- 更好的权限访问控制——所有的团队不需要完全访问所有的库——需要的时候，再获得读访问权限。

### 劣势

- 跨服务和项目使用的公共依赖和库必须定期同步以获得最新版本。
- 某种程度上鼓励孤立文化，导致重复代码和各个团队试图解决相同问题。
- 每个团队可能遵循不同的一组最佳实践来编写代码，从而导致难以遵循通用的最佳实践。

### 示例

像下面代码库中只有一个package.json即一个项目，src中都是业务代码。

![image-20221003112129090](https://raw.githubusercontent.com/caifeng123/pictures/master/image-20221003112129090.png)



## monorepo

> 对于物可视就是一个monorepo的项目，整体的前后端+libs共23个库共用一个git进行管理

### 优点

- 存储所有项目代码的单独位置，团队中的每个人都可以访问。
- 易于重用和共享代码，与团队合作。
- 很容易理解你的变更对整个项目的影响。
- 代码重构和代码大变更的最佳选择。
- 团队成员可以获得整个项目的总体视图。
- 易于管理依赖关系。

### 劣势

- 项目规模增大，代码 checkout 和 pull 时间就会增长、文件搜索需要更长的时间。
- 对于安全性来说，访问整个代码库不那么安全。
- 持续部署CD也很困难，因为许多人可以合入他们的更改，而持续集成CI系统可能需要进行多次重构。

### 示例

像下面代码库中有一个总 `package.json` 进行管理，所有的子库都在一个文件夹(packages)中，每个子库都有各自的`package.json`
使用 `lerna + yarn workspace` 进行多包管理。

![image-20221003112420100](https://raw.githubusercontent.com/caifeng123/pictures/master/image-20221003112420100.png)

在最外侧的 `package.json` 中，需要补充 `workspaces` 子库的文件夹。

```json
{
    "name": "vue-learning",
    "version": "1.0.0",
    "main": "index.js",
    "repository": "https://github.com/caifeng123/vueTeaching.git",
    "author": "caifeng01",
    "license": "MIT",
    "private": true,
    "devDependencies": {
        "lerna": "^5.4.3"
    },
    "workspaces": [
        "packages/*"
    ]
}
```

### 总结

当前市面上实现monorepo最有名的就是 [**lerna库**](http://www.febeacon.com/lerna-docs-zh-cn/routes/basic/start.html) 了。

直接使用npm + lerna和yarn + lerna都能实现多包管理，但对于npm多包管理来说，很容易会出现幽灵依赖的问题

> 幽灵依赖：子包1中引用了A库子包2中的`package.json` 明明没有A库 ,但还是能正常import且不报错

因此我们就需要进行分割的概念，对于yarn来说使用yarn 2、3即可解决，对于npm、yarn 1.x却没有解决方法，最后我们的主角pnpm就要登场了，他就能很好处理这些问题。



# pnpm

## 优势

### 体积

存储在硬盘上，都会以软链形式挂钩

很多小伙伴和我说mac存储空间不够，看到硬盘里有块『其他』部分占用很大体积，找又找不到，更别提删掉了。其中其实有一部分就是我们各个项目中的 `node_modules` 了。由于每个项目都需要安装 `node_modules` 实际上重复性很高，又忘记删除就被遗留下来了。

![image-20221003142615852](https://raw.githubusercontent.com/caifeng123/pictures/master/image-20221003142615852.png)

### 处理幽灵依赖

> 网状 + 平铺的node_modules结构 `.pnpm` 以平铺的形式储存着所有的包，正常的包都可以在这种命名模式的文件夹中被找到

```
.pnpm/<organization-name>+<package-name>@<version>/node_modules/<name>

// 组织名(若无会省略)+包名@版本号/node_modules/名称(项目名称)
```

- 在最外侧的 `node_modules` 中的.pnpm目录中，将所有的依赖包以上述格式生成硬链接。即他们和硬盘上的包是同一文件。

- 在子库内部的依赖都是外层`.pnpm` 的软链。

PS: 软链与硬链接的区别？

<img src="https://raw.githubusercontent.com/caifeng123/pictures/master/image-20221003144119489.png" style="width:47%;" /><img src="https://raw.githubusercontent.com/caifeng123/pictures/master/image-20221003145514210.png" style="width:47%;" />

### PeerDependencies

如果 `foo` 有 peer 依赖（peer dependencies），那么它可能就会有多组依赖项，所以我们为不同的 peer 依赖项创建不同的解析：

![image.png](https://raw.githubusercontent.com/caifeng123/pictures/master/4dd2ecc8c09b4cf7a01d72e6f5113102%7Etplv-k3u1fbpfcp-zoom-in-crop-mark%3A3024%3A0%3A0%3A0.awebp)

若有多个peer依赖时，会在命名规则处做手脚，确保项目软链到唯一的.pnpm/xx项目

### 多包管理

天生支持monorepo



## monorepo项目搭建

### 安装环境

```shell
npm install -g pnpm
yarn global add pnpm
```

### 特殊准备

- 使用 `pnpm init` 生成package.json
- 创建 `pnpm-workspace.yaml` 中配置子库地址，进行识别隔离

![image-20221003153250091](https://raw.githubusercontent.com/caifeng123/pictures/master/image-20221003153250091.png)

### 常用指令

[pnpm cli指令](https://www.pnpm.cn/cli/add) 详细可看官网，同lerna指令很像，有单独安装，同时执行指令等。具体使用看官网即可，无需过多记忆。