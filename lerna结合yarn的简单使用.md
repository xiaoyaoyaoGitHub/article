## `lerna`结合`yarn`的简单使用

#### 初始化`lerna`

```bash
$ lerna init
```

> 会生成目录结构如下：

```markdown
|-- lerna
    |-- lerna.json
    |-- package.json
    |-- readme.md
    |-- packages
```

> 这个时候`packages`里面的包是空的，我们可以执行命令创建包

#### 创建包

```bash
$ lerna create @wangly/pack1
```

> 在`packages`目录下生成`pack1`目录包

```markdown
|-- lerna
    |-- packages
        |-- pack1
            |-- README.md
            |-- package.json
            |-- tests              //单元测试入口
            |   |-- pack1.test.js
            |-- lib
                |-- pack1.js
```

#### 添加每个包都需要的依赖

```bash
$ lerna add loadsh
  lerna notice cli v4.0.0
  lerna info Adding loadsh in 2 packages
  lerna info Bootstrapping 2 packages
  lerna info Installing external dependencies
  lerna info Symlinking packages and binaries
  lerna success Bootstrapped 2 packages
```

> 执行完成后，会在每个包的根目录都生成一个node_modules，里面保存包里面需要的依赖

#### 单个包安装依赖

> `lerna add loadsh --scope [包名称]`

```bash
$ lerna add loadsh --scope @wangly/pack1
  lerna notice cli v4.0.0
  lerna notice filter including "@wangly/pack1"
  lerna info filter [ '@wangly/pack1' ]
  lerna info Adding loadsh in 1 package
  lerna info Bootstrapping 2 packages
  lerna info Installing external dependencies
  lerna info Symlinking packages and binaries
  lerna success Bootstrapped 2 packages
```

> 只会在`@wangly/pack1`下面安装依赖`loadsh`，而在`pack2`z中不会被安装

#### 修改配置文件

>  当我们开发完包功能后,我们需要关联到本地测试,这个时候需要修改跟目录下的`package.json`中的属性`private`修改为`true`,增加`workspaces`属性,然后修改结果为

```json
{
  "name": "root",
  "private": true,
  "devDependencies": {
    "lerna": "^4.0.0"
  },
  "workspaces":[
    "packages/*"
  ]
}
```

> `lerna.json`增加属性

```json
"useWorkspaces": true,
"npmClient": "yarn",
```

#### 测试安装

> 在根目录下执行`yarn`

```bash
$ yarn
  yarn install v1.17.3
  info No lockfile found.
  [1/4] 🔍  Resolving packages...
  warning lerna > @lerna/add > pacote > @npmcli/run-script > node-gyp > request@2.88.2: request has been deprecated, see https://github.com/request/request/issues/3142
  warning lerna > @lerna/bootstrap > @lerna/run-lifecycle > npm-lifecycle > node-gyp > request@2.88.2: request has been deprecated, see https://github.com/request/request/issues/3142
  warning lerna > @lerna/add > pacote > @npmcli/run-script > node-gyp > request > har-validator@5.1.5: this library is no longer supported
  [2/4] 🚚  Fetching packages...
  [3/4] 🔗  Linking dependencies...
  [4/4] 🔨  Building fresh packages...
  success Saved lockfile.
  ✨  Done in 4.70s.
```

>  根目录出现`node_modules`,里面找到`@wangly/pack1` 和` @wangly/pack2`

我们可以在自己的项目中引入依赖包使用

#### 结合实际项目使用

>  在我们之前提到的[monorepo](https://mp.weixin.qq.com/s?__biz=Mzg3OTEyMzgxNA==&amp;mid=2247483654&amp;idx=1&amp;sn=376a79513af3b90efb6456e58d31d357&amp;chksm=cf080d0ff87f8419e087c47c108c5439efc27b941993c9279d7bcacd24df8dd77d7ca3ecfda0&token=941524049&lang=zh_CN&scene=21#wechat_redirect)中使用`lerna`管理包，可以按照上面的流程修改下`lerna.json`和根目录下的`package.json`文件，后续执行`yarn`安装，可以看到根目录下的`node_modules`中包含我们自己开发的包依赖

简单学习，后续补充

